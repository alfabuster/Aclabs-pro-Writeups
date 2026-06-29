# Bank — aclabs.pro

<img width="1200" height="630" alt="bank-writeup" src="https://github.com/user-attachments/assets/5b07cb5a-015d-4736-9a18-a7b15e8649f5" />


![](https://img.shields.io/badge/Platform-ACLabs.pro-blue?style=for-the-badge)
![](https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge)
![](https://img.shields.io/badge/OS-Windows%20%2B%20Linux-informational?style=for-the-badge)
![](https://img.shields.io/badge/Category-AD%20%7C%20Crypto%20%7C%20Web%20%7C%20Pivoting%20%7C%20Docker-orange?style=for-the-badge)
![](https://img.shields.io/badge/Flags-6%20%2F%206-brightgreen?style=for-the-badge)

## Summary

Bank is a **Hard**-rated multi-machine challenge on Aclabs.pro featuring 6 flags across two hosts and three network layers. The journey begins on a Windows domain controller running IIS, where a hidden script file leaks base64-encoded credentials for WinRM access (**Flag 1**). BloodHound reveals `GenericWrite` on the `Server Operators` group — self-addition grants `SeBackupPrivilege`, enabling admin desktop file exfiltration (**Flag 2**). A second network interface exposes a Linux host running a banking vault web app. The vault's session cookie uses HMAC-SHA256 with an 8-digit PIN as the key — a custom Python brute-forcer cracks it in minutes (**Flag 3**). Opening the vault unlocks new ports: ProFTPD 1.3.9 (vulnerable to CVE-2026-42167 pre-auth SQLi backdoor) leaks PostgreSQL credentials, and the database's `superuser` role enables RCE via `COPY FROM PROGRAM` into a Docker container (**Flag 4**). Inside the container, a custom SUID binary (`/usr/bin/cati`) — reversed with Ghidra — reads a hardcoded path as root; a symlink attack exposes `/etc/shadow`, and cracking the root hash grants container root (**Flag 5**). Finally, a writable `/var/run/docker.sock` allows spawning a privileged container with the host filesystem mounted, yielding the true root flag (**Flag 6**).

```
Hidden Script → Base64 Creds → WinRM (Flag 1)
  → GenericWrite ACL → Server Operators → SeBackupPrivilege (Flag 2)
  → Second NIC → Ligolo Pivot → HMAC PIN Brute-Force (Flag 3)
  → ProFTPD CVE-2026-42167 → PostgreSQL RCE → Container (Flag 4)
  → SUID Binary Reversing → Symlink /etc/shadow → Container Root (Flag 5)
  → Docker Socket Escape → Host Root (Flag 6)
```

## MITRE ATT&CK Mapping

| Phase | Tactic | Technique | ID |
|:------|:-------|:----------|:---|
| Port scanning | [Discovery](https://attack.mitre.org/tactics/TA0007/) | [Network Service Discovery](https://attack.mitre.org/techniques/T1046/) | `T1046` |
| Source code & hidden file discovery | [Reconnaissance](https://attack.mitre.org/tactics/TA0043/) | [Active Scanning: Vulnerability Scanning](https://attack.mitre.org/techniques/T1595/002/) | `T1595.002` |
| Base64 credentials in script | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Unsecured Credentials: Credentials In Files](https://attack.mitre.org/techniques/T1552/001/) | `T1552.001` |
| WinRM remote access | [Lateral Movement](https://attack.mitre.org/tactics/TA0008/) | [Remote Services: Windows Remote Management](https://attack.mitre.org/techniques/T1021/006/) | `T1021.006` |
| GenericWrite ACL abuse | [Persistence](https://attack.mitre.org/tactics/TA0003/) | [Account Manipulation](https://attack.mitre.org/techniques/T1098/) | `T1098` |
| SeBackupPrivilege file copy | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [OS Credential Dumping: Security Account Manager](https://attack.mitre.org/techniques/T1003/002/) | `T1003.002` |
| Internal network scanning | [Discovery](https://attack.mitre.org/tactics/TA0007/) | [Remote System Discovery](https://attack.mitre.org/techniques/T1018/) | `T1018` |
| HMAC PIN brute-force | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Brute Force: Password Cracking](https://attack.mitre.org/techniques/T1110/002/) | `T1110.002` |
| ProFTPD pre-auth backdoor | [Initial Access](https://attack.mitre.org/tactics/TA0001/) | [Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) | `T1190` |
| PostgreSQL COPY FROM PROGRAM | [Execution](https://attack.mitre.org/tactics/TA0002/) | [Command and Scripting Interpreter: Unix Shell](https://attack.mitre.org/techniques/T1059/004/) | `T1059.004` |
| Custom SUID binary abuse | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Abuse Elevation Control: Setuid and Setgid](https://attack.mitre.org/techniques/T1548/001/) | `T1548.001` |
| /etc/shadow hash cracking | [Credential Access](https://attack.mitre.org/tactics/TA0006/) | [Brute Force: Password Cracking](https://attack.mitre.org/techniques/T1110/002/) | `T1110.002` |
| Docker socket container escape | [Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) | [Escape to Host](https://attack.mitre.org/techniques/T1611/) | `T1611` |

## CVE Reference

| CVE | CVSS 3.1 | Severity | Description |
|:----|:---------|:---------|:------------|
| [CVE-2026-42167](https://github.com/ZeroPathAI/proftpd-CVE-2026-42167-poc) | **9.8** | Critical | ProFTPD 1.3.9 pre-auth SQL injection via `is_escaped_text()` bypass — enables backdoor user creation with uid=0 |

