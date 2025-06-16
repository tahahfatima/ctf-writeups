🛰️ Anthem — TryHackMe Walkthrough (Beginner-Level)

🚀 Initial Recon: Nmap All the Way

We start with a full port scan:

nmap -p- <target-ip>

Open Ports:

    80 – HTTP

    3389 – RDP (Remote Desktop Protocol)

That immediately tells us two things:

    There’s a web app to investigate

    Remote login might be available later (via xfreerdp or similar)

🕷️ Web Crawlers & Hidden Clues

Basic enumeration on http://<target-ip> reveals the classic robots.txt, often used to guide crawlers — but sometimes it leaks sensitive info. Sure enough, we find:

UmbracoIsTheBest!

That’s likely a password. Let’s keep it handy.
⚙️ CMS Identification

Nothing obvious in the source code or with tools like Wappalyzer. But that password gives away the CMS:

    🛠️ CMS: Umbraco

🌐 Domain Discovery

Visible within the page footer (or right at the top in some themes), we identify the internal domain as:

anthem.com

We update /etc/hosts accordingly to resolve it locally.
🔍 Administrator Username Enumeration

Digging into author metadata and blog content, we uncover references to “Solomon Grundy” — likely the admin user.

Based on the email pattern (jane.doe@anthem.com), we infer:

    Admin Email: SG@anthem.com
    Username: SG

🚩 Flag Hunting

Most flags are embedded in the HTML of various blog posts. Here's the flag-hunting breakdown:

    ✅ Flag 1 – Found in "We're Hiring" blog post source.

    ✅ Flag 2 – Hidden in homepage source code.

    ✅ Flag 3 – Jane Doe’s author page.

    ✅ Flag 4 – Source metadata of IT blog post.

Solid example of how source code inspection is vital in web enumeration.
🔑 RDP Login: Initial Access

Using previously found credentials:

    User: SG

    Pass: UmbracoIsTheBest!

We log in with:

xfreerdp /u:SG /p:'UmbracoIsTheBest!' /v:anthem.com

    Make sure anthem.com resolves via /etc/hosts to the box’s IP.

We gain user-level access, and quickly grab user.txt.
🕵️ Admin Password & Privilege Escalation

Exploration reveals a hidden backups folder that we can’t initially access. However, misconfigured file permissions allow us to modify access rights and reveal sensitive data.

With a GUI-based permission tweak (since CLI is restricted), we uncover the Administrator credentials and escalate to full control.
