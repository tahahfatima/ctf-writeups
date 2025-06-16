🚨 Hack The Box - MonitorsThree (Hard) - Walkthrough

Difficulty: Hard
Author: HTB
Category: Linux | Web | Exploitation | Privilege Escalation

🛰️ Summary

The MonitorsThree machine requires strategic chaining of vulnerabilities including SQL injection, CVE exploitation, lateral movement via credentials, and an authentication bypass in Duplicati. The attack path:

    🔍 Extract admin credentials via time-based blind SQLi

    ⚡ Exploit Cacti CVE-2024–25641 for remote code execution

    🔑 Dump and crack Marcus’s password from MySQL

    🔐 SSH lateral movement via private key

    🌀 Port forward to access Duplicati interface

    🧬 Bypass Duplicati login using server-passphrase manipulation

    🎯 Abuse backup/restore to exfiltrate root flag

🧪 Recon

A standard Nmap scan revealed only two open ports:

22/tcp  ssh
80/tcp  http

The main web interface at http://monitorsthree.htb was basic, but fuzzing via ffuf revealed a subdomain:
➡️ cacti.monitorsthree.htb

Both hosts featured login pages. Digging deeper into the main login form exposed a “Password Recovery” endpoint.
🧬 Exploiting SQL Injection

Testing revealed a time-based blind SQL injection on the recovery form. Due to the sluggish nature of this vector, a custom Python script was used to extract the admin’s password character-by-character based on timing logic:

# Key logic: check each char via timing-based logic until SUCCESS message is seen

🔥 The script successfully retrieved the admin password hash, which was cracked using hashes.com.
🎯 CVE-2024-25641 — Cacti RCE

Cacti’s authenticated package import feature was vulnerable. Despite automated PoCs failing, a manual payload was crafted using PHP to generate a .xml.gz archive containing a reverse shell.

# File: cati.xml.gz
# Payload: Reverse shell injected via fake Cacti package

After importing the package, the PHP file was accessible via a predictable path. Upon execution, a reverse shell as www-data was established.
🔓 MySQL Loot — Marcus's Credentials

Inside /var/www/html/cacti/include/config.php, MySQL credentials were found. This enabled access to the Cacti database:

mysql -u cactiuser -p

Inside the user_auth table, a password hash for marcus was recovered and cracked with hashcat:

hashcat -m 3200 marcus.hash /usr/share/wordlists/rockyou.txt

🔑 Marcus’s SSH key was also extracted and reused to gain shell access as Marcus.
🚪 Pivot — Duplicati on Port 8200

While enumerating internal services, TCP port 8200 was found to host a Duplicati instance. A local port forward exposed the web interface:

ssh -L 8200:localhost:8200 marcus@monitorsthree.htb

🧬 Bypassing Duplicati Authentication

🔍 The key to bypass was hidden in /opt/duplicati/, which contained a .sqlite database holding:

    server-passphrase

    server-passphrase-salt

The passphrase was decoded and turned into a 256-bit hex string. Intercepting the login request in BurpSuite revealed a Nonce value used in client-side password transformation.
🔐 JavaScript Console Magic

In DevTools console:

var saltedpwd = '<256-bit hex>';
var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('<NonceFromBurp>') + saltedpwd)).toString(CryptoJS.enc.Base64);
console.log(noncedpwd);

The resulting noncedpwd was Base64-encoded and inserted back into the password field of /login.cgi.

✅ Access to the Duplicati dashboard was achieved.
🧨 Privilege Escalation via Backup/Restore

From the GUI:

    🔧 Created a new backup job named rootLoot

    🗂️ Path set to /source/tmp

    ➕ Manually added /source/root/root.txt

    ▶️ Ran the backup

    📦 Navigated to "Restore"

    ☑️ Selected root.txt, restored to /home/marcus/root.txt

Boom 💥 — the root flag was now accessible in Marcus’s home directory.
