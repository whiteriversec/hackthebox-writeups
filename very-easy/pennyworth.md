# Pennyworth | Very Easy | 2026-07-07

## Summary
Pennyworth is a Linux box running a Jenkins automation server on port 8080 via the Jetty web server. Default credentials (root:password) granted access to the Jenkins dashboard. The Jenkins Script Console at /script accepts Groovy code and executes it server-side. A Groovy reverse shell payload established a connection back to Kali via Netcat, landing directly as root.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know Jenkins had a script console | Jenkins exposes a Groovy script console at /script that executes code server-side -- direct RCE once authenticated |
| Didn't know Groovy was used for Jenkins scripting | Jenkins uses Groovy as its scripting language -- reverse shell payloads need to be written in Groovy, not bash or Python |
| Didn't know about the concept of a reverse shell | Instead of connecting to the target directly, a reverse shell makes the target connect back to your machine -- bypassing inbound firewall restrictions. Netcat (nc -lvnp 4242) listens for that incoming connection and hands you an interactive shell |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 8080 10.129.x.x`
- **Open ports / services found:**
  - `8080` -- HTTP (Jetty web server running Jenkins)
- **Interesting observations:** Jenkins login page on port 8080 -- default credentials worth trying immediately

---

## Enumeration

### Web application
- Browsed to `http://10.129.x.x:8080`
- Found Jenkins login page
- Tried default credentials: `root:password` -- successful login
- Navigated to `http://10.129.x.x:8080/script` -- Jenkins Script Console accessible

---

## Foothold
- **Vulnerability:** Default credentials on Jenkins + unauthenticated Script Console access = RCE
- **CVE:** N/A -- misconfiguration
- **Exact commands:**

Step 1 -- set up Netcat listener on Kali:
```bash
nc -lvnp 4242
```

Step 2 -- craft Groovy reverse shell payload from:
https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#nodejs

Modified `cmd.exe` to `/bin/bash` since target is Linux.

Step 3 -- paste payload into Jenkins Script Console at /script and execute

Step 4 -- Netcat listener receives connection, shell returned as root

- **Why it worked:** Jenkins was running with default credentials never changed. The Script Console executes arbitrary Groovy code with the same privileges as the Jenkins process -- which was running as root. Groovy has direct access to Java runtime which can spawn OS processes, making reverse shell execution trivial once authenticated.

---

## Privilege Escalation
- N/A -- landed directly as root via Jenkins process privileges

---

## Tools Used
- nmap
- Netcat (nc)
- Jenkins Script Console
- Groovy reverse shell payload

## Resources
- Reverse Shell Cheat Sheet: https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/

---

## Key Takeaways
- **Jenkins Script Console** -- accessible at /script once authenticated, executes arbitrary Groovy code server-side. Effectively RCE by design if an attacker gains access
- **Default credentials are always the first check** -- root:password, admin:admin, jenkins:jenkins. Jenkins has well-documented default credentials worth trying before any other technique
- **Reverse shells** -- instead of getting output directly in a response (like SSTI), a reverse shell makes the target connect back to your machine, giving an interactive shell session. nc -lvnp sets up the listener that catches the incoming connection
- **cmd.exe vs /bin/bash** -- reverse shell payloads are often written for Windows (cmd.exe). On Linux targets, change to /bin/bash
- **What would have stopped this:**
  - Change default credentials immediately on Jenkins setup
  - Restrict Script Console access to administrators only
  - Run Jenkins as a low-privilege service account rather than root
  - Don't expose Jenkins directly to the network without authentication hardening

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Pennyworth | Very Easy | Default credentials + Jenkins Script Console RCE | N/A | 2026-07-07 |