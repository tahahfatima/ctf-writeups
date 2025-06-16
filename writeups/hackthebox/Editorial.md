HTB: Editorial (Hard) — Full Walkthrough

    TL;DR

        Gained SSRF to discover internal API

        Extracted dev credentials from exposed endpoint

        Escalated via git logs to prod user

        Abused sudo-allowed Python script running vulnerable GitPython (CVE-2022-24439)

        Achieved root via RCE in a misconfigured repo pull

🔍 Initial Recon

A classic nmap scan didn’t yield much aside from port 80 (HTTP). Visiting the website revealed a basic interface, with a potentially interesting /upload feature.

This /upload endpoint allowed input for either a URL (bookurl) or a file (bookfile) — a strong candidate for SSRF testing.
🔁 SSRF Exploitation

Intercepted the request using Burp Suite and replaced bookurl with a reference to my attack box listener:

bookurl=http://<attacker-ip>:<port>

Got a hit on my listener — confirmed SSRF.

To probe internal services, I used Burp Intruder to scan ports 1–9999. Only port 5000 responded differently, returning content under /static/uploads.

Navigating to http://editorial.htb/static/uploads revealed a file listing various internal API endpoints — jackpot.

Accessing /api/latest/metadata/messages/authors leaked a file containing dev credentials (probably from a debug log). Despite a warning that the password should’ve been changed, it still worked.

Used the creds to SSH in as dev.
✔ Foothold gained.
🔐 Lateral Movement to prod

Enumerating git logs revealed the prod user was demoted to dev, and the commit history disclosed prod user credentials.

SSH’d in successfully as prod.
⚙ Privilege Escalation via GitPython RCE

As prod, I had sudo permissions for:

sudo /usr/bin/python /opt/internal_apps/clone_changes/clone_prod_change.py

The script's name hinted at git operations. Reviewing its source revealed it leveraged the GitPython library — a known vulnerability (CVE-2022-24439) exists here if attacker-controlled Git URLs are used.

Referenced the Snyk writeup and used a modified PoC that triggers RCE through a malicious ext::sh -c payload.

First, tested a simple file write to /tmp — worked.

Then executed the final payload to exfiltrate root.txt:

sudo /usr/bin/python /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c "cat /root/root.txt > /tmp/root.txt"'

Checked /tmp/root.txt — root flag obtained.
✔ Rooted.
🏁 Conclusion

Vulns chained:

    SSRF to internal enumeration

    Insecure credential storage

    Git log exposure

    Sudo access to unsafe Python script

    GitPython RCE (CVE-2022-24439)

“Editorial” was a clean escalation path from SSRF to root via multiple real-world misconfigurations.
