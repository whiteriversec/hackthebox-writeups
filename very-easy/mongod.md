# MONGOD | Very Easy | 30/06/2026
## Summary
The attack was focused on a Linux machine running two services, a SSH service on port 22 and a MongoDB service on Port 27017 TCP. MongoDB service had an oudated version and was therefore first in order to explore as an attack vector. We came across a challenge when we couldn't utilise Mongo shell because our version did not match. We used a Docker container to solve this challenge. The DB was unauthenticated and allowed us free access into the databases which contained the flag.

---

## Recon
**Nmap command:** `nmap -sV -sC -p- --min-rate 5000 <target>`
**Open ports / services found:**
  - `22` -- SSH (OpenSSH)
  - `27017` -- Mongodb (MongoDB 3.6.8)
  
  **Interesting observations:**

  - The machine ran an old/oudated version of MongoDB
  - Our Mongo toolset on the CLI was actually not compatible because it was up to date

---

**Service / port targeted:**
- Port 27017 TCP: MongoDB 3.6.8

**Tools used + exact commands:**
- nmap -sV -sC -p- --min-rate 5000 <target>

**What was found (versions, misconfigs, files):**
- MongoDB was old, didn't require authentication

**Dead ends and why they were ruled out:**

---

## Foothold
**Vulnerability / misconfiguration:**
- Version was old, had no authentication enabled. Old versions came shipped with no authentication by default unless configured.

**CVE (if applicable):**
**Exact commands / payloads used:**
- sudo docker run -it --rm mongo:3.6 mongo --host [ip] --port [port] (-i to allow typing in container, -t pseudo-terminal, --rm auto remove once we exit)
- show dbs (lists dbs)
- use [db name] (enters that database)
- show collections (shows collections/tables in the database)
- db.[collection].find() (lists the contents of the table)

**Why it worked:**

---

## Privilege Escalation
**Method:** N/A
**Exact commands:** N/A
**Why it worked:** N/A

---

## Flags
- Flag located in 'flag' collection inside 'sensitive_information' db in the misconfigured and outdated MongoDB service
- "1b6e6fb359e7c40241b6d431427ba6ea"

---

## Tools Used
- nmap
- docker
- mongo

---

## Key Takeaways
**What did I learn?**
- Even though I found a misconfigured/exploitable service I had trouble accessing it due to my own version being higher.
- Learn that I can utilise Docker to run the exact version of that service in order to access it.
- We also learned how to navigate the MongoDB service using CLI once we had access.

**What would I do differently?** N/A
**Technique to revisit:** Using Docker container to match the service required
**What would have stopped this attack?** Updated MongoDB version (comes with automatic authentication) or configured authentication manually.




