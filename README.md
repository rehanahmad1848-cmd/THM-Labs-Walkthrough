**TryHackMe Labs - Writeups & Solutions**
This repository contains my personal notes, walkthroughs, and scripts for rooms and challenges completed on TryHackMe.
> ⚠️ **Disclaimer:** All write-ups are for educational purposes only and follow TryHackMe's content/disclosure guidelines. For rooms that explicitly prohibit publishing full answers/flags, this repo documents methodology and techniques only, with sensitive answers redacted. Do not use any techniques described here against systems you do not own or have explicit permission to test.
Repository Structure
```
thm-writeups/
├── Rooms/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
├── Paths/
│   ├── Pre-Security/
│   ├── Cyber-Security-101/
│   ├── Jr-Penetration-Tester/
│   ├── Offensive-Pentesting/
│   ├── SOC-Level-1/
│   └── SOC-Level-2/
├── Scripts/
└── Notes/
```
Writeup Format
Each room folder includes:
`README.md` — full walkthrough (recon, enumeration, exploitation, privilege escalation, task answers)
`notes.md` — quick notes, commands used, and useful one-liners
`scripts/` — any custom exploit scripts or tools used
`screenshots/` — proof of completion (where applicable)
Example Walkthrough Layout
```markdown
# Room Name (Difficulty - OS)

## Information
- Room URL: https://tryhackme.com/room/<room-name>
- IP: 10.10.x.x
- OS: Linux/Windows
- Difficulty: Easy/Medium/Hard

## Enumeration
- Nmap scan results
- Service enumeration findings

## Foothold
- Vulnerability identified
- Exploitation steps

## Privilege Escalation
- Enumeration for PrivEsc
- Exploitation steps

## Flags / Task Answers
- user.txt: (redacted/hash only)
- root.txt: (redacted/hash only)
- Task answers: (redacted where the room prohibits disclosure)
```
Tools Commonly Used
Nmap
Gobuster / Feroxbuster
Burp Suite
Metasploit
LinPEAS / WinPEAS
John the Ripper / Hashcat
Custom Python/Bash scripts
How to Use
```bash
git clone https://github.com/<your-username>/thm-writeups.git
cd thm-writeups
```
Browse to the relevant room folder for full documentation.
Uploading Solutions to GitHub
```bash
# Initialize repo (first time only)
git init
git add .
git commit -m "Add writeup for <Room Name>"

# Connect to remote repo
git remote add origin https://github.com/<your-username>/thm-writeups.git
git branch -M main
git push -u origin main

# For subsequent updates
git add .
git commit -m "Add writeup: <Room Name>"
git push
```

Disclaimer

This content is shared strictly for learning purposes related to cybersecurity and penetration testing. Flags, IPs, and sensitive identifiers are redacted or omitted in compliance with TryHackMe's rules.

THM Profile: https://tryhackme.com/p/LashariLR181
LinkedIn: https://www.linkedin.com/in/rehan-ahmad-16195a325/

License

This project is licensed under the MIT License.
