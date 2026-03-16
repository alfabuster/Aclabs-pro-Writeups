<p align="center">
  <img src="https://img.shields.io/badge/Platform-ACLabs.pro-blueviolet?style=for-the-badge&logo=hackthebox&logoColor=white" alt="Platform">
  <img src="https://img.shields.io/badge/Focus-Offensive_Security-critical?style=for-the-badge&logo=kalilinux&logoColor=white" alt="Focus">
  <img src="https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white" alt="OS">
</p>

# 🔓 ACLabs.pro — Writeups

Writeups for machines from the [ACLabs.pro](https://aclabs.pro) platform, documenting real attack chains and methodology used during each engagement.

Each writeup covers the full kill chain — from initial reconnaissance through exploitation to privilege escalation and flag capture.

---

## 📂 Repository Structure

Each machine has its own directory with a detailed writeup:

```
aclabs-writeups/
│
├── dragon/
│   └── dragon_writeup.md
│
├── <pwnelines>/
│   └── pwnelines_writeup.md
│
└── README.md
```

### Writeup Format

Every writeup follows a consistent structure:

- **Summary** — one-liner describing the attack chain
- **Reconnaissance** — port scanning, service enumeration
- **Enumeration** — directory fuzzing, version fingerprinting, vulnerability research
- **Exploitation** — initial foothold, proof of concept
- **Privilege Escalation** — local enumeration, escalation vector
- **Flag** — proof of compromise
- **Lessons Learned** — key takeaways and techniques to remember

---

## 🛠 Toolbox

| Phase | Tools |
|---|---|
| **Scanning** | Nmap, Rustscan |
| **Web** | Burp Suite, ffuf, Gobuster, curl |
| **Exploitation** | SQLMap, Metasploit, custom scripts |
| **Post-Exploitation** | LinPEAS, GTFOBins, pspy |
| **General** | Python, CyberChef, John the Ripper, Hashcat |

---


## ⚠️ Disclaimer

These writeups are published strictly for **educational and documentation purposes**.  
Unauthorized access to computer systems is illegal. Always obtain proper authorization before testing.

---

<p align="center">
  <a href="https://github.com/alfabuster"><img src="https://img.shields.io/badge/GitHub-@alfabuster-181717?style=flat-square&logo=github" alt="GitHub"></a>
  <a href="https://tryhackme.com/p/alfabuster"><img src="https://img.shields.io/badge/TryHackMe-alfabuster-212C42?style=flat-square&logo=tryhackme&logoColor=white" alt="TryHackMe"></a>
  <a href="https://alfabuster.com"><img src="https://img.shields.io/badge/Blog-alfabuster.com-00FF41?style=flat-square&logo=ghost&logoColor=white" alt="Blog"></a>
</p>
