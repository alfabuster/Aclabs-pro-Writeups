# SaveWalterWhite — Aclabs.pro - WriteUP

![](https://img.shields.io/badge/Platform-ACLabs.pro-blue?style=for-the-badge)
![](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![](https://img.shields.io/badge/Category-Web%20%7C%20LFI%20%7C%20RFI%20%7C%20PrivEsc%20%7C%20Docker-orange?style=for-the-badge)

<img width="1919" height="883" alt="SaveWalterWhite_aclabs pro" src="https://github.com/user-attachments/assets/94afc805-1c8c-4722-9ff1-512c11d8e945" />

## Summary

SaveWalterWhite is a **Medium**-rated Linux machine on Aclabs.pro featuring five flags across multiple layers of exploitation. An Apache web server exposes a PHP file inclusion endpoint (`image.php?file=`) that allows both **Path Traversal** (reading `/etc/passwd`) and **Remote File Inclusion**, getting RCE as `www-data`. A world-writable `backup.sh` script runnable as `jesse` via sudo provides the first lateral move (**Flag 1**). Database credentials found in `admin.php` lead to a Borg Backup passphrase hidden in a donations table, unlocking Saul's SSH key — which itself requires cracking with `john` (**Flag 2**). Saul's sudo permission to `tee` into Walter's `authorized_keys` grants SSH access as `walter` (**Flag 3**). A custom SUID binary expecting a Breaking Bad character alias ("Heisenberg") escalates to root inside the container (**Flag 4**). Finally, a **privileged Docker container escape** via host disk mounting yields the true root flag (**Flag 5**).

```
Path Traversal → RFI to RCE → sudo backup.sh → jesse (Flag 1)
  → DB creds → Borg passphrase → SSH key crack → saul (Flag 2)
  → sudo tee authorized_keys → walter (Flag 3)
  → SUID OSINT binary → container root (Flag 4)
  → Privileged container escape → host root (Flag 5)
```

<img width="1919" height="883" alt="MITRE_ATTACK_SaveWalterWhite_aclabs pro" src="https://raw.githubusercontent.com/alfabuster/Aclabs-pro-Writeups/515b1e2ca3b3b5c4bc83a4c7cc3340434d370b6e/CTF%206%20Season/SaveWalterWhite/MITRE_ATTACK_Aclabspro_SaveWalterWhite.svg" />

## MITRE ATT&CK Mapping

| Phase | Tactic | Technique | ID |
|:------|:-------|:----------|:---|
| Port scanning | [Discovery](https://attack.mitre.org/tactics/TA0007/) | [Network Service Discovery](https://attack.mitre.org/techniques/T1046/) | `T1046` |
| Source code review | [Reconnaissance](https://attack.mitre.org/tactics/TA0043/) | [Gather Victim Host Info: Client Configurations](https://attack.mitre.org/techniques/T1592/004/) | `T1592.004` |
| Path Traversal / RFI | [Initial Access](https://attack.mitre.org/tactics/TA0001/) | [Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) | `T1190` |
| Remote PHP webshell | [Execution](https://attack.mitre.org/tactics/TA0002/) | [Command and Scripting Interpreter: Unix Shell](https://attack.mitre.org/techniques/T1059/004/) | `T1059.004` |
| Writable `backup.sh` abuse | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Abuse Elevation Control: Sudo and Sudo Caching](https://attack.mitre.org/techniques/T1548/003/) | `T1548.003` |
| Database credential extraction | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Unsecured Credentials: Credentials In Files](https://attack.mitre.org/techniques/T1552/001/) | `T1552.001` |
| Borg passphrase from DB | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Credentials from Password Stores](https://attack.mitre.org/techniques/T1555/) | `T1555` |
| SSH private key cracking | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Brute Force: Password Cracking](https://attack.mitre.org/techniques/T1110/002/) | `T1110.002` |
| SSH key injection via `tee` | [Persistence](https://attack.mitre.org/tactics/TA0003/) | [Account Manipulation: SSH Authorized Keys](https://attack.mitre.org/techniques/T1098/004/) | `T1098.004` |
| SUID binary exploitation | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Abuse Elevation Control: Setuid and Setgid](https://attack.mitre.org/techniques/T1548/001/) | `T1548.001` |
| Privileged container escape | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Escape to Host](https://attack.mitre.org/techniques/T1611/) | `T1611` |

## CVSS Reference

| ID | CVSS 3.1 | Severity | Description |
|:----|:---------|:---------|:------------|
| Path Traversal | **4.3** (est.) | Medium | Path Traversal in `image.php` — unauthenticated arbitrary file read via `file` parameter CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N |
| RFI | **9.8** (est.) | Critical | Remote File Inclusion in `image.php` — unauthenticated RCE via remote PHP payload inclusion CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H |
| SUID Script | **7.8** (est.) | High | Writable SUID script executable via sudo — allows arbitrary command execution as another user CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H |
| Misconfiguration in Docker | **8.2** (est.) | High | Privileged Docker container — allows host filesystem mount and full host compromise CVSS:3.1/AV:L/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H |
