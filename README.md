# THM_labs

Writeups for [TryHackMe](https://tryhackme.com) rooms and labs I've solved —
notes, methodology, and the commands I used, kept for my own reference and
in case they help someone else.

## Structure

Each lab gets its own folder:

```
thm_labs/
├── support-lab/
│   ├── writeup.md     # full walkthrough: recon → exploitation → flags
│   └── cmd.md          # every command used, explained line by line
├── <next-lab>/
│   ├── writeup.md
│   └── cmd.md
└── README.md
```

- **writeup.md** — the story of the room: what I found, what I tried, what
  worked, root causes, and fixes.
- **cmd.md** — a command-by-command breakdown explaining *why* each tool /
  flag was used, for quick reference later.

## Index

| Lab | Category | Writeup |
|-----|----------|---------|
| Support | Web Exploitation (BOLA, Auth Bypass, LFI, Command Injection) | [support-lab/writeup.md](./support-lab/writeup.md) |

*(Table updated as new labs are added.)*

## Notes

- These are my own solves, written up after completing each room — no
  spoilers beyond what's needed to explain the methodology.
- Flags are included where TryHackMe's room rules don't ask for them to be
  withheld; otherwise they're redacted.
- Feedback, corrections, and alternate approaches are welcome via issues/PRs.

## Disclaimer

All testing shown here was performed against intentionally vulnerable
machines provided by TryHackMe for educational purposes. None of the
techniques described should be used against systems you don't own or don't
have explicit permission to test.
