# GhostBusters — ACLabs.pro
<img width="1200" height="630" alt="ghostbusters-writeup" src="https://github.com/user-attachments/assets/b9adb0d0-3838-479c-8570-740564161b63" />


![](https://img.shields.io/badge/Platform-ACLabs.pro-blue?style=for-the-badge)
![](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![](https://img.shields.io/badge/Category-Web%20%7C%20Linux-orange?style=for-the-badge)

## Summary

GhostBusters is an **Easy**-rated Linux machine on ACLabs.pro running Ghost CMS 6.19 on nginx. An unauthenticated SQL injection (**CVE-2026-26980**) dumps admin credentials from the database. After cracking the bcrypt hash, a malicious theme upload exploit (**CVE-2026-29053**) achieves RCE as user `slimer`. A Debian 6.12 kernel is vulnerable to **DirtyFrag**, granting root. However, neither flag exists yet — a custom security service binary (`secservice`) monitors the filesystem and only generates flags when specific conditions are met: the user flag appears after removing a sentinel PNG file, and the root flag materializes after changing the root password (altering the hash in `/etc/shadow`). Understanding this requires Ghidra analysis of the binary's two monitoring loops.

```
Ghost CMS 6.19 → CVE-2026-26980 (SQLi) → Admin Creds → CVE-2026-29053 (Theme RCE)
  → Shell as slimer → DirtyFrag Kernel Exploit → Root
  → Binary Reversing → Remove sentinel file (User Flag) → Change root password (Root Flag)
```

## MITRE ATT&CK Mapping

| Phase | Tactic | Technique | ID |
|:------|:-------|:----------|:---|
| Port scanning | [Discovery](https://attack.mitre.org/tactics/TA0007/) | [Network Service Discovery](https://attack.mitre.org/techniques/T1046/) | `T1046` |
| CMS version fingerprinting | [Reconnaissance](https://attack.mitre.org/tactics/TA0043/) | [Gather Victim Host Info: Client Configurations](https://attack.mitre.org/techniques/T1592/004/) | `T1592.004` |
| Unauthenticated SQLi (CVE-2026-26980) | [Initial Access](https://attack.mitre.org/tactics/TA0001/) | [Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) | `T1190` |
| bcrypt hash cracking | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Brute Force: Password Cracking](https://attack.mitre.org/techniques/T1110/002/) | `T1110.002` |
| RCE via malicious theme (CVE-2026-29053) | [Execution](https://attack.mitre.org/tactics/TA0002/) | [Command and Scripting Interpreter: Unix Shell](https://attack.mitre.org/techniques/T1059/004/) | `T1059.004` |
| DirtyFrag kernel exploit | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Exploitation for Privilege Escalation](https://attack.mitre.org/techniques/T1068/) | `T1068` |
| Custom SUID binary analysis | [Discovery](https://attack.mitre.org/tactics/TA0007/) | [File and Directory Discovery](https://attack.mitre.org/techniques/T1083/) | `T1083` |

## CVE Reference

| CVE | CVSS 3.1 | Severity | Description |
|:----|:---------|:---------|:------------|
| [CVE-2026-26980](https://github.com/vognik/CVE-2026-26980) | **7.5** | High | Ghost CMS ≤6.19 — unauthenticated SQL injection via Content API allows extraction of user credentials (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N) |
| [CVE-2026-29053](https://github.com/AC8999/CVE-2026-29053) | **7.2** | High | Ghost CMS — authenticated RCE via malicious theme upload; crafted Handlebars template executes arbitrary commands (AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:H) |
| CVE-2026-43284 | **8.8** (est.) | High | DirtyFrag — Linux kernel 6.12 local privilege escalation via memory fragmentation abuse (AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H) |
