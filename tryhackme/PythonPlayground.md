Box: PythonPlayground — TryHackMe (Hard)

Crafted by the talented @stuxnet, this box takes a unique route: instead of hammering weak services, the challenge is built entirely around the security of Python scripting. It’s a deep dive into how poor coding practices — especially in scripting languages like Python — can lead to devastating vulnerabilities.

Let’s get into the breakdown. 🧠💥
🔍 Reconnaissance

Starting with our usual scan:

nmap -sV -p- -T4 <target-ip>

We find:

    22/tcp – SSH

    10000/tcp – Unidentified service (possibly HTTP-based)

Nmap isn’t confident about port 10000, but it smells like HTTP. Let’s check it out in the browser.
🐾 Initial Foothold via Python Evaluation

Accessing the page shows a Python error. Specifically, it complains that GET is not defined — that’s a strong hint that the server is evaluating request methods as code.

Burp Suite confirms it: the app takes our request method or parameters and directly plugs it into a Python input() function.

Let’s test that idea.

    Sending 0 — returns nothing

    Sending 0+1 — triggers a string with “Exploiting tryhackme” → confirms eval() or input() is used

Now we go for RCE.

Using classic Python injection with __import__('os').system() gives us command execution as user king.

Let's drop a reverse shell. Boom — we’re in. 🐚
👑 Privilege Escalation to Root

Inside king’s home directory, we find a root.sh file — owned by root. It contains:

python /media/script.py

Suspicious. Why run Python like this in a cronjob without a full path? This makes it a candidate for PATH hijacking — except root likely has a clean default path, and we don't control python.

Instead, we turn to pspy to inspect cron activity.

We discover a cronjob triggers anything inside /media. Also, port 8080 is listening locally. Using socat, we forward it externally and visit it in the browser — it’s a file upload web app.

Here’s the plan:

    Create a .py file with a reverse shell payload

    Upload it via the web interface

    Wait for the cronjob to execute it from /media

Within a minute — ✅ root shell acquired.
🔥 Bonus Insight — Python Insecurity

The real vulnerability started with an outdated and dangerous coding practice in Python 2: the use of input().

data = input("Enter something: ")  # BAD!

In Python 2, input() literally evaluates user input. This means an attacker can inject code directly into the interpreter.

Solution?

raw_input()  # Treats all input as strings safely

Python 3 now makes input() behave like raw_input(), closing this hole. But legacy scripts remain at risk.
🧠 Final Thoughts

This was an excellent challenge in:

    Python code evaluation abuse

    Creative privilege escalation via cron + upload + execution

    Deep understanding of how code and infrastructure interact

