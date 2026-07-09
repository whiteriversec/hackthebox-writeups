We begin by enumerating using nmap but this time with different flag 'nmap -Pn -sC <ip>' to avoid firewall denials and detection
We have a microsoft-ds service running on Port 445 TCP (SMB port)
We used 'smbclient -L <ip> -U Administrator' to log in with default credentials and list Shares
We then access the C$ share via "smbclient ////<ip>/C$ -U Administrator" and navigate to Administrator's desktop and download 'flag.txt'
Alternatively we could have used psexec.py (part of Impacket framework) - used for for attacking Windows and Active Directory (need to explore this more) - this allows us to enable a fully interactive shell on remote Windows machies.
The way we would do this is 'python3 psexec.py administrator@<ip>'
NEED TO RESEARCH MORE ON psexec.py
