# TryHackMe Hacker Holidays 2026: The Byte Lotus

This repository contains my 14-day write-up series for TryHackMe's **Hacker
Holidays 2026** event, set at The Byte Lotus resort. Each day includes a
walkthrough, the techniques used, screenshots, and lessons learned.

> **Educational use only:** These write-ups describe intentionally vulnerable
> TryHackMe lab machines. Do not apply the techniques to systems without
> explicit authorization. Challenge flags and sensitive details are retained
> in the individual write-ups for personal study.

## Challenge write-ups

| Day | Room | Main topics | Write-up |
|---:|---|---|---|
| 1 | The Concierge Knows Too Much | Prompt injection, role impersonation, system-prompt leakage | [Day 1](Day%201/Day1.md) |
| 2 | Room 404 | Directory enumeration, exposed `.git`, Git history recovery | [Day 2](Day%202/Day2.md) |
| 3 | Complimentary | Client-side source review, AWS Cognito, IAM, DynamoDB enumeration | [Day 3](Day%203/Day3.md) |
| 4 | Packed Light | Wireshark, C2 identification, cookie exfiltration, XOR/Base64 decoding | [Day 4](Day%204/Day4.md) |
| 5 | Beach Bar | Web reconnaissance, unsafe YAML deserialization, RCE, privilege escalation | [Day 5](Day%205/Day5.md) |
| 6 | Overheard at Breakfast | OSINT, Gravatar enumeration, hash identification, Base64 decoding | [Day 6](Day%206/Day6.md) |
| 7 | Do Not Disturb | NoSQL injection, EJS SSTI, Node Inspector abuse, raw-disk access | [Day 7](Day%207/Day7.md) |
| 8 | Towel on the Sunbed | Client-side validation, Burp Repeater, TOCTOU race condition | [Day 8](Day%208/Day8.md) |
| 9 | CryptoCabana | Azure Blob Storage, SAS scope, service principals, Key Vault versions/RBAC | [Day 9](Day%209/Day9.md) |
| 10 | The Hollow Shell | Zip Slip, arbitrary file write, upload validation, RCE | [Day 10](Day%2010/Day10.md) |
| 11 | Infinity Pool | Boot2root, OS command injection, internal service discovery, Chisel, FreePBX, bearer-token command injection | [Day 11](Day%2011/Day11.md) |
| 12 | After Hours | WMI repository forensics, encoded PowerShell, fileless loading | [Day 12](Day%2012/Day12.md) |
| 13 | The Guestbook | Indirect prompt injection, blocklist evasion, command injection, exfiltration | [Day 13](Day%2013/Day13.md) |
| 14 | Management Wants a Word | KAPE, Chrome credentials, DPAPI, LSA secrets, VeraCrypt | [Day 14](Day%2014/Day14.md) |

## Skills covered

Across the series, the write-ups practice:

- **Reconnaissance:** Nmap, Gobuster, browser developer tools, source review,
  and technology fingerprinting.
- **Web application security:** authentication bypass, NoSQL injection, SSTI,
  unsafe deserialization, Zip Slip, race conditions, and exposed repositories.
- **Cloud security:** AWS Cognito and IAM, DynamoDB access control, Azure SAS
  tokens, Blob Storage, service principals, Key Vault versioning, and RBAC.
- **AI security:** prompt injection, authority impersonation, instruction
  leakage, blocklist evasion, and unsafe tool/command execution.
- **Digital forensics:** Wireshark, WMI repository analysis, encoded payload
  recovery, KAPE triage, DPAPI, browser credential decryption, and container
  analysis.
- **Post-exploitation:** reverse shells, shell stabilization, process-argument
  disclosure, Node.js Inspector access, and raw disk permissions.

## Repository structure

Each day is kept in its own directory with its Markdown write-up and the
screenshots referenced by that write-up:

```text
.
├── README.md
├── Day 1/
│   ├── Day1.md
│   └── image*.png
├── Day 2/
│   ├── Day2.md
│   └── image*.png
├── ...
└── Day 14/
    ├── Day14.md
    └── image*.png
```

The Markdown files are the source of truth for each room's objective, attack
path, commands, screenshots, and lessons learned.

## Overall reflection

The event builds from lightweight information gathering into increasingly
complex attack chains. Early rooms focus on trusting user input and exposing
secrets; later rooms combine multiple weaknesses across application, cloud,
operating-system, and forensic boundaries. The recurring defensive lessons
are:

1. Enforce authentication and authorization on the server or service, not in
   client-side behavior.
2. Apply least privilege to cloud identities, tokens, roles, and filesystem
   groups.
3. Treat uploaded files, templates, serialized data, and model input as
   untrusted.
4. Protect secrets throughout their lifecycle, including source code,
   process arguments, backups, version history, and old secret versions.
5. Investigate persistence and execution paths beyond conventional startup
   locations.

## Next steps

- Revisit difficult rooms without relying on the walkthrough.
- Turn each room's lessons into defensive notes and detection ideas.
- Practice the same techniques in authorized labs and local test environments.
- Review weak areas such as cloud IAM, Windows internals, and concurrent
  request handling.
