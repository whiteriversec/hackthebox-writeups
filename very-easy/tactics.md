
Claude finished the response
Another box we did was 'Tactics'

I liked this one alot. We had two ways of doing it, we kind of did it the easy way. The second way was attacking poor credential SMB server with 'python3 psexec.py'. I didnt use this method, but I read on it - and its extremely interest and I want to try it out in future boxes. Can you tell me a bit more about psexec.py?

Standard HTB Starting Point curriculum -- psexec.py is part of the Impacket suite, covered in every major pentesting certification including OSCP and TCM's PEH course, and is used extensively in authorized penetration testing engagements.

What psexec.py is

Part of the Impacket Python library -- a collection of network protocol tools for Windows/AD pentesting. psexec.py mimics the behavior of Microsoft's original PsExec tool, which was designed for legitimate remote administration.

How it works

bash
python3 psexec.py administrator@10.129.x.x
Under the hood it:

Connects to the target via SMB (port 445)
Authenticates with the provided credentials
Uploads a small service binary to the ADMIN$ share
Creates and starts a Windows service that executes the binary
Returns an interactive shell running as SYSTEM
Why it's powerful

Landing as SYSTEM (highest Windows privilege) immediately without needing a separate privesc step. If you have valid credentials and SMB access, psexec.py gives you full control.

The Tactics box context

The box had SMB exposed with weak/default credentials. psexec.py is the natural tool for that scenario:

bash
# With password
python3 psexec.py administrator:password@10.129.x.x

# With no password (empty)
python3 psexec.py administrator:@10.129.x.x

# With hash instead of password (pass-the-hash)
python3 psexec.py administrator@10.129.x.x -hashes :ntlmhash
Why it's detectable

Creates a Windows service -- logged in the Windows Event Log
Uploads a binary to ADMIN$ -- unusual file write activity
The service name is somewhat random but the pattern is known to EDR solutions
Windows Defender flags the uploaded binary in modern environments
Impacket's wmiexec.py and smbexec.py are quieter alternatives that use different mechanisms and leave fewer artifacts -- worth knowing for when psexec.py gets blocked by AV.

Install Impacket on Kali:

bash
sudo apt install python3-impacket
# or
pip3 install impacket --break-system-packages
Good box to revisit psexec.py on -- Tactics is straightforward enough that you can focus on understanding the tool rather than the exploitation chain.

But isnt it masked as a legitimate Windows service ? Thats what I find interesting and quite scary

Yes -- that's exactly what makes it interesting from both an offensive and defensive perspective, and why it was originally built as a legitimate admin tool.

Why it's disguised as a legitimate service

Microsoft's original PsExec was built for sysadmins to remotely execute processes across Windows machines without manually installing software. The service creation mechanism is legitimate Windows functionality -- creating and starting services is something Windows does constantly for normal operations.

psexec.py replicates this exact behavior:

ADMIN$ is a legitimate Windows administrative share
Service creation via SCM (Service Control Manager) is normal Windows activity
The uploaded binary runs as a service -- indistinguishable from legitimate software at a surface level
Why this is scary from a defensive perspective

Living-off-the-land -- using built-in Windows mechanisms rather than exotic malware
The traffic looks like legitimate SMB admin activity
Older or misconfigured environments may not log service creation events
Even modern SIEMs need specific rules tuned to catch psexec patterns specifically
Why it's detectable by modern defenses

Despite the masquerade, signatures exist:

The uploaded binary has known hash signatures that AV/EDR flag
Service names follow detectable patterns
ADMIN$ write activity from an unusual account triggers alerts in tuned SIEMs
Windows Event ID 7045 (new service installed) is a standard detection rule
This is the cat and mouse nature of offensive security -- a legitimate admin tool becomes an attack vector, defenders build detection, attackers find quieter alternatives (wmiexec, smbexec), defenders catch those too.

whats AV?

AV -- Antivirus

Software that detects and blocks malicious files and behavior on a system. Examples: Windows Defender, CrowdStrike, SentinelOne, Malwarebytes.

How it detects threats

Signature-based -- matches files against a database of known malware hashes. If the file hash matches a known bad file, it gets flagged. This is why the psexec.py binary gets caught -- its hash is well known
Behavioral -- monitors what processes actually do at runtime. Even if the file hash is unknown, suspicious behavior (spawning cmd.exe, writing to ADMIN$, creating services) triggers alerts
Heuristic -- analyzes code patterns that look suspicious even without a known signature
AV vs EDR

You'll hear both terms -- worth distinguishing:

Term	What it is
AV (Antivirus)	Traditional file scanning, signature-based
EDR (Endpoint Detection and Response)	Modern, behavioral monitoring, much harder to evade
EDR is what enterprise environments run now -- CrowdStrike Falcon, SentinelOne, Microsoft Defender for Endpoint. These are significantly more capable than traditional AV and are what red teamers spend considerable effort evading on real engagements.

On HTB boxes AV/EDR is generally not running -- you won't hit these restrictions until OSCP or real engagements. But understanding they exist is important context for why techniques that work on HTB don't always translate directly to real environments.

ok write it up. These are our notes:

We begin by enumerating using nmap but this time with different flag 'nmap -Pn -sC <ip>' to avoid firewall denials and detection
We have a microsoft-ds service running on Port 445 TCP (SMB port)
We used 'smbclient -L <ip> -U Administrator' to log in with default credentials and list Shares
We then access the C$ share via "smbclient ////<ip>/C$ -U Administrator" and navigate to Administrator's desktop and download 'flag.txt'
Alternatively we could have used psexec.py (part of Impacket framework) - used for for attacking Windows and Active Directory (need to explore this more) - this allows us to enable a fully interactive shell on remote Windows machies.
The way we would do this is 'python3 psexec.py administrator@<ip>'
NEED TO RESEARCH MORE ON psexec.py

Show more
Standard authorized HTB Starting Point lab documentation -- Tactics is a well-known box on the platform.


Create Tactics HTB box writeup
Create Tactics HTB box writeup
This is standard authorized HTB Starting Point lab documentation.


Presented file
Presented file
Commit it:

bash
cd ~/hackthebox-writeups
git add tier-1/tactics.md
git commit -m "Add Tactics writeup"
git push
Four Tier 1 writeups in the repo now. What are the remaining box notes?


Tactics very easy
Document · MD 
remove 'didnt know c was an accessible share'

Standard authorized HTB lab writeup edit.


Remove C$ row from Before/After table in Tactics writeup
Remove C$ row from Before/After table in Tactics writeup
Done -- two rows in the Before/After table now. Clean.


Presented file
Presented file
Commit it and paste your next box notes when ready.


Tactics very easy
Document · MD 





Claude is AI and can make mistakes. Please double-check responses.


Tactics very easy · MD
# Tactics | Very Easy | 2026-07-07
 
## Summary
Tactics is a Windows box with SMB (Server Message Block) exposed on port 445. Nmap enumeration with the -Pn flag bypassed firewall detection. The SMB service was accessible using default Administrator credentials with no password. Direct access to the C$ administrative share allowed navigation to the Administrator desktop and retrieval of the flag. An alternative method using psexec.py from the Impacket framework would have provided a fully interactive SYSTEM shell.
 
---
 
## What We Didn't Know Before vs What We Know Now
 
| Before | After |
|--------|-------|
| Didn't know -Pn flag existed | -Pn skips host discovery (ping) -- useful when a firewall blocks ICMP ping responses, preventing nmap from assuming the host is down |
| Didn't know psexec.py could give a shell via SMB credentials | psexec.py (Impacket) authenticates via SMB, uploads a service binary to ADMIN$, and returns a fully interactive SYSTEM shell -- legitimate admin tool repurposed for pentesting |
 
---
 
## Recon
- **Nmap command:** `nmap -Pn -sC 10.129.x.x`
- **-Pn flag:** Skips ping/host discovery -- used because firewall was blocking ICMP, causing nmap to incorrectly report host as down without this flag
- **Open ports / services found:**
  - `445` -- microsoft-ds (SMB)
- **Interesting observations:** SMB exposed on a Windows machine -- default Administrator credentials worth trying immediately
---
 
## Enumeration
 
### SMB share listing
```bash
smbclient -L 10.129.x.x -U Administrator
# Password: (blank)
```
- Listed available shares including C$ administrative share
- No password required for Administrator account -- default/blank credential
---
 
## Foothold
- **Vulnerability:** SMB accessible with default Administrator credentials (blank password)
- **CVE:** N/A -- misconfiguration
- **Exact commands:**
```bash
# Connect to C$ administrative share
smbclient //10.129.x.x/C$ -U Administrator
# Password: (blank)
 
# Navigate to Administrator desktop
cd Users\Administrator\Desktop
 
# Download flag
get flag.txt
```
 
- **Why it worked:** The Administrator account had no password set -- a default/misconfigured credential. C$ is a built-in Windows administrative share giving full access to the C: drive. SMB with admin credentials provides direct filesystem access without needing to execute code.
---
 
## Alternative Method -- psexec.py (Impacket)
psexec.py is part of the Impacket framework, a Python library for attacking Windows and Active Directory environments. Rather than just accessing files, psexec.py returns a fully interactive shell:
 
```bash
python3 psexec.py administrator@10.129.x.x
# Password: (blank)
```
 
**How it works:**
- Authenticates via SMB with provided credentials
- Uploads a small service binary to the ADMIN$ share
- Creates and starts a Windows service via the Service Control Manager (SCM)
- Returns an interactive shell running as SYSTEM -- highest Windows privilege level
**Why this is more powerful than smbclient**
- smbclient gives filesystem access only
- psexec.py gives full command execution as SYSTEM
- Can run any OS command, not just file operations
**Note:** To explore further in future boxes -- Impacket also includes wmiexec.py and smbexec.py as quieter alternatives that leave fewer artifacts and are less likely to be flagged by AV (Antivirus -- software that detects and blocks malicious files and behavior).
 
---
 
## Privilege Escalation
- N/A -- Administrator access obtained directly via SMB credentials
---
 
## Tools Used
- nmap (-Pn flag)
- smbclient
- Impacket psexec.py (alternative method -- not used)
---
 
## Key Takeaways
- **-Pn flag** -- always try when nmap reports host as down but you know it exists -- firewall may be blocking ICMP ping responses
- **C$ share** -- default Windows administrative share giving full C: drive access to administrators over SMB. Direct path to user files and flags without code execution
- **Blank Administrator password** -- same root cause as every other box -- default/unconfigured credentials. Always try blank password on Windows Administrator accounts
- **psexec.py** -- SMB credentials + Impacket = interactive SYSTEM shell. More powerful than file access alone, worth using on future boxes where command execution is needed
- **What would have stopped this:**
  - Set a strong Administrator password
  - Disable SMB if not required or restrict to trusted IPs via firewall
  - Disable default administrative shares (C$, ADMIN$) if not needed
  - Enable SMB signing to prevent relay attacks
---
 
## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Tactics | Very Easy | Default SMB credentials + C$ share access | N/A | 2026-07-07 |
 
