🚀 Jupiter | HTB Write-up

Grafana RawSQL RCE → Shadow Simulator PrivEsc → Jupyter Abuse → Root via Sattrack
🛰️ Overview

Jupiter is a medium-difficulty HTB machine that blends web exploitation with misconfigured services and clever privilege escalation. The box revolves around abusing:

    A Grafana panel with raw SQL injection via PostgreSQL

    A Shadow Simulator cronjob for lateral movement

    A Jupyter Notebook interface for privilege escalation

    A custom binary (sattrack) for final root compromise

🧠 Initial Enumeration

We start with a full TCP port scan:

nmap -p- -sCV --min-rate=1000 10.10.11.216

Open ports:

    22 (SSH)

    80 (HTTP)

Browsing port 80 shows nothing interesting. We fuzz for hidden directories & subdomains using ffuf and discover a subdomain running Grafana.

Update /etc/hosts with the new subdomain and revisit it in the browser.
📊 Grafana → Raw SQL Injection → RCE

While interacting with Grafana dashboards, we intercept a request to:

POST /api/ds/query

This request has a rawSql field and uses the PostgreSQL backend. We inject a custom SQL payload to test execution:

SELECT version();

It works — we’ve got SQL Injection.

Next, we pivot to Remote Code Execution (RCE) using PostgreSQL's COPY FROM PROGRAM trick:

DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'id';
SELECT * FROM cmd_exec;

Boom — we get uid=postgres, confirming command execution. After chaining commands (whoami, etc.), we spawn a reverse shell as the postgres user.
🔍 Postgres → Shadow Simulator Exploitation

From the shell, we notice two users: juno and jovian.

postgres can't access home directories directly, so we drop pspy to watch cronjobs.

💡 A script called shadow-simulation.sh runs periodically and consumes /dev/shm/network-simulation.yml. It's linked to the open-source Shadow Simulator tool, which executes simulation-based workloads using real system binaries.

We inject our payload into network-simulation.yml:

cmd: /tmp/sh -p

Next cron run executes it and we get a shell as juno.
🔐 SSH Key Persistence for Juno

We generate an SSH key, and append the public key to ~/.ssh/authorized_keys. Now we have stable SSH access as juno.
🔭 Jupyter Notebook → Jovian Access

Upon privilege escalation attempts, we find Jupyter Notebook running under the jovian user, hosting a notebook in /opt/solar-flares.

We port-forward Jupyter from port 8888 (localhost-only) to our machine using SSH:

ssh -L 8888:localhost:8888 juno@jupiter.htb

Browsing localhost:8888, we grab the Jupyter token from the log files and authenticate.
📓 Jupyter Notebook → Command Execution

We open flares.ipynb, switch to a code cell, and use Jupyter magic commands:

%%bash
mkdir -p /home/jovian/.ssh
echo "<our-public-key>" > /home/jovian/.ssh/authorized_keys
chmod 600 /home/jovian/.ssh/authorized_keys

Now we have SSH access as jovian.
🛰️ Root Privilege via sattrack

jovian can run sattrack as root via sudo.

sudo -l

It fails due to a missing config file. We inspect the binary using strings and discover it looks for /tmp/config.json.

After reverse-engineering the binary, we learn it loads TLE satellite data from a URL. By inserting a local file read path in config.json, we can abuse it to read /root/root.txt.

{
  "url": "file:///root/root.txt"
}

Running sudo ./sattrack with this config file dumps the root flag.
