# CTF Writeup - FTP → Web → SSH → Privilege Escalation

## 🧭 Overview

This is a Linux-based CTF machine involving multiple stages of exploitation:

- Network reconnaissance
- FTP anonymous access
- Information disclosure via file hints
- Web enumeration using Gobuster
- SSH brute force attack
- Privilege escalation to root

---

## 🔍 Step 1: Reconnaissance (Nmap Scan)

We started by scanning the target using Nmap:

```

nmap -sC -sV <ip>

```

### Open Ports Found:
- **21/tcp** → FTP (vsftpd 3.0.3)
- **80/tcp** → HTTP (Apache 2.4.18 Ubuntu)
- **2222/tcp** → SSH (OpenSSH 7.2p2)

### Key Observations:
- FTP allowed anonymous login
- SSH was running on a non-standard port (2222)
- Web server hosted a default Apache page with hidden directories

---

## 📁 Step 2: FTP Enumeration

We logged into FTP using anonymous credentials:

```

ftp <iP>
Username: anonymous
Password: (blank)

```

During directory listing, passive mode issues were encountered, which were resolved by enabling passive mode.

After successful navigation, we discovered a file:

```

ForMitch.txt

```

We downloaded and read it:

```

get ForMitch.txt -

```

### 📌 File Content Insight

The file contained a message indicating:

- The system password is very weak
- The password is reused across system users
- A user named **mitch** exists on the system

### Key Conclusion:
We now have a valid username: `mitch`

---

## 🌐 Step 3: Web Enumeration

We performed directory brute forcing using Gobuster:

```

gobuster dir -u [http://<iP>](http://<iP>) -w /usr/share/wordlists/dirb/common.txt

```

### Discovered Endpoint:
```

/simple

```

This revealed a CMS system.

### Observation:
- The CMS likely contains known vulnerabilities
- It serves as a secondary attack surface for credential discovery

---

## 🔐 Step 4: SSH Brute Force Attack

Based on FTP hints, we identified:
- Username: `mitch`
- Password: weak/reused password likely present in system

We used Hydra for brute force:

```

hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://<iP>:2222

```

### Result:
- Password was successfully cracked
- SSH login access was obtained

We logged in:

```

ssh mitch@<iP> -p 2222

```

---

## ⚡ Step 5: Privilege Escalation

```md id="vimsec1"
## Privilege Escalation

After gaining access as the user `mitch`, we checked sudo permissions:

```

sudo -l

```id="vimsec2"

The output showed that the user was allowed to run Vim with elevated privileges.

### Exploitation

Since Vim was allowed via sudo, we used it to spawn a root shell:

```

sudo vim -c ':!/bin/sh'

```id="vimsec3"

### Explanation:
- `sudo vim` runs Vim as root
- `-c` executes a command inside Vim on startup
- `:!/bin/sh` spawns a system shell from within Vim

### Result:
This granted a root shell, allowing full system compromise.

---

### Alternative Methods

Inside Vim, the following can also be used:

```

:!bash

```id="vimsec4"

or

```

:shell

```id="vimsec5"

---

### Outcome

We successfully escalated privileges from user `mitch` to `root` and obtained the final flag.
```

---

## 🧠 Key Learnings

- FTP anonymous access can leak critical hints
- Enumeration is more important than brute force
- Weak password reuse is common in CTF environments
- Gobuster is useful for discovering hidden web directories
- Hydra can be used for SSH brute forcing when required
- Privilege escalation depends on system misconfigurations

---

## 🚀 Attack Flow Summary

```

Nmap Scan
↓
FTP Anonymous Login
↓
ForMitch.txt (Password Hint)
↓
Web Enumeration (/simple)
↓
Hydra SSH Brute Force
↓
SSH Login (mitch)
↓
Privilege Escalation
↓
Root Flag

```

---

## 🛠 Tools Used

- Nmap
- FTP client
- Gobuster
- Hydra
- SSH
- Linux enumeration commands
```
