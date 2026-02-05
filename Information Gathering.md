## Passive Information Gathering
Also Known as OSINT
passive = gathering info through normal interaction

### whois enumeration
whois - a tcp service, tool, and database that can provide more info about a domain name
a lot of info may not be useful, but some can be
nameserver ip can be found --> DNS servers

```bash
whois megacorpone.com -h 192.168.241.258
```
-h specifies the whois server
some useful output is names of employees and nameservers
### Google Hacking
Google can be used to uncover critical info, vulnerabilities and misconfigured websites
Done by using clever search strings and operators
Process is iterative. Start broad and narrow search as we go
**Site operator** limits search to one domain
Example "site:megacorpone.com"
**Filetype** operator specifies a specific filetype we want to search for
Example "site:megacorpone.com filetype:txt"
Using this we may be able to uncover useful pages like robots.txt which can lead to further resources to check. Additionally we can try specifying php, xml, or py to try and figure out the underlying programming language(s) used
"-" can be used to exclude an operator
Example "site:megacorpone.com -filetype:html"
This will return all results that are not html files
We can use intitle to locate pages that contain certain titles
Example "intitle:"index of" "parent directory"
This finds pages with index of in the title and parent directory on the page content
Google hacking database on exploit db website contains mroe advanced hacking strings to show what is possible
### Netcraft
Netcraft gathers information on websites: (**https://searchdns.netcraft.com**)
We can search "site contains" \*.megacorpone.com to fins subdomains
can also find site technologies info
### Open Source Code
Sites like github, gitlab and Source Forge may contain code that can give us more information
Many of these sites support operators like google
if we navigate to megacorpone's repo and search filename:users we find a web app file that has a username and salted password
### Shodan
Search engine that crawls devices connected to the internet
need free account, but gives port and service info
will also reveal if there are any published vulns running on sleeted host
## Active Information Gathering
### DNS Enumeration
DNS is a distributed database responsible for translating domain name into ip addresses
Common DNS record types:
- NS: name server records that contain name of authoritive servers hosting DNS records for domain
- A: contains ipv4 address of host
- AAAA: contains ipv6 address of host
- MX: Mail exchange records contain name of servers responsible for handling mail to the domain
- PTR: Pointer records are used in reverse lookup zones and can finds records associated with an IP address
- CNAME: Canonical Name Records are used to create aliases for other host records
- TXT: text records contain arbitrary data and can be used for any purpose

host commnad can help enumerate dns info
```shell
host www.megacorpone.com
```
by default will search for A record
use -t  to specify record type
```shell
host -t mx megacorpone.com
```
if we search for a server and it is not found we get the following error:
```shell
host idontexist.megacorpone.com
```
Host idontexist.megacorpone.com not found: 3(NXDOMAIN) --> we can use this knowledge to automate our effort
we can develop a brute forcing technique for dns look ups
- build a list of possible host names ex: list.txt
```shell
for ip in $(cat list.txt); do host $ip.megacorpone.com; done
```
much more complex lists are found in Seclists which can be installed running
```shell
sudo apt install seclists
```
If we have identified IP addresses in the same range we can also use reverse lookups to identify servers running within that range
```shell
for ip in $(seq 200 254); do host 51.222.169.$ip; done | grep -v "not found"
```
grep -v will filter out results that contain the string "not found"

#### Tools
There are a few tools in Kali linux that will do dns enumeration
dnsrecon is an advanced tool for this enumeration
```shell
dnsrecon -d megacorpone.com -t std
```
-d: specify domain name
-t: specify type of enum performed
	std: standard scan
	brt: brute force
-D: supplies list for Brute force

dnsenum is another popular tool --> seems more comprehensive than dns recon off the bat
```shell
dnsenum megacorpone.com
```

nslookup can be used on windows machines if necessary
```cmd
nslookup -type=TXT info.megacorptwo.com 192.168.50.151
```
queries can be automated through powershell or bash scripting if needed

### TCP/UDP Port Scanning
dynamic process --> results of one scan should feed into input of the next
We can use netcat as a port scanner. It is usually not a port scanner, but it is on most systems os we can repurpose it as one if needed
```shell
nc -nvv -w 1 -z 192.168.50.152 3388-3390
```
-w: specifies timeout
-z: specifies zero-I/O --> used for scanning and sends no data
-n: numeric IPs (no dns)
-vv: very verbose
-u: udp
note udp may get reported as open if there is a firewall. this is due to the lack of an "icmp port unreachable message" if a firewall is guarding the port
Be aware that some scanners will not scan all ports, especially udp
Bash loop to sweep subnet for a specific port (445 in this example)
```bash
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445; done
```
### Port scanning with NMAP
Basic format
 ```shell
 nmap <ip>
```
-p: ports to scan --> can be number, range or list
-sS: syn stealth scan --> default when nothing is specified and the user has raw socket privileges
	usually faster because it needs to send less data and doesn't compete the handshake
-sT connect scan --> default when user doesn't have raw socket privileges
	Takes longer because nmap has to wait for connection to complete
	Might be useful to perform when scanning through certain types of proxies
-sU UDP scan --> will use the "ICMP port unreachable" method for most ports. Will send specific protocol packets for common ports. If firewalls drop packets no "ICMP unreachable" packet will be passed back and ports might falsely be reported as open
-sn Host discovery --> will use more than just icmp to get data. will also send tcp syn packets to port 80 and 443 to verify if host is available
-oG grepable output -->
-A: Aggressive OS and service enumeration
-sV: plain service scan --> subset of A
--top-ports=20 --> can specify how many of the top ports we want to scan using scripts
-O: OS fingerprinting --> attempts to guess the targets OS
	--osscan-guess: option will force nmap to guess on OS type for O scan

To deal with large networks we can do simple network sweeps and then more detailed scans on discovered hosts
#### Scripts
--script-help can be used to gain more info about a script
```shell
nmap --script-help http-headers
```
if internet isn't available much of that info can be found in the .nse file
general syntax for scripts
```shell
nmap --script=http-enum 192.168.225.0/24
```
scripts:
	--http-enum : enumerates common directories for http
	--http-feed: checks http feeds --> idk what feeds are
	--http-title: returns title of http sites
	--smb-os-discovery:
	
#### Living off the land
Other options if nmap is not available
```Powershell
Test-NetConnection -Port 445 192.168.50.151
```
We can further script this whole process with the PowerShell one liner
```Powershell
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"} 2>$null
```
### SMB Enumeration
NetBios listens on port 139 TCP and several UDP ports. SMB listens on port 445 TCP. These are two separate protocols
NetBIOS: independent session layer protocol and service that allows computer on a local network to communicate with each other
Modern implementation of SMB can work without NetBIOS, but need Netbios to be backwards compatible so they usually go hand in hand
we can still use nmap for these
```shell
nmap -v -p 139,445 -o smb.txt 192.168.225.1-254
```
Remember there are many nse scripts for smb
There are also more specialized tools
```shell
sudo nbtscan -r 192.168.50.0/24
```
nbtscan - queries information about the NetBIOS configuration
-r: specifies UDP port 137
nmap also has useful smb scripts
#### From Windows Client
net view: will list the domains, resources, and computers belonging to a given host
```cmd
net view \\dc01 /all
```
### SMTP
we can enumerate smtp to find existing usernames/emails in an system
```shell
nc -nv 192.168.50.8 25
```
Above command will connect to the SMTP server.
We can then use the following commands:
- VRFY - can use to verify the servers
	- 252 confirms user exists
	- 550 error given for unknown user
Following script will check 1 username
```python
#!/usr/bin/python

import socket
import sys

if len(sys.argv) != 3:
        print("Usage: vrfy.py <username> <target_ip>")
        sys.exit(0)

# Create a Socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect to the Server
ip = sys.argv[2]
connect = s.connect((ip,25))

# Receive the banner
banner = s.recv(1024)

print(banner)

# VRFY a user
user = (sys.argv[1]).encode()
s.send(b'VRFY ' + user + b'\r\n')
result = s.recv(1024)

print(result)

# Close the socket
s.close()
```
For Windows we can can use the Test-NetConnection cmdlet
```powershell
Test-NetConnection -Port 25 192.168.50.8
```
Using the above command we cannot query info like we did previously.
However, we can overcome this if the telnet client is installed
```cmd
telnet 192.168.50.8 25
```
### SNMP Enumeration
MIB trees contain information related to network management. The database is organized like a tree.
We can use the tool **onesixtyone** to bruteforce community strings for snmp
```shell
onesixtyone -c community -i ips
```
where community is a list of community strings to try and ips is a list of IPs
we can use **snmpwalk** to gather info from snmp tables granted we know the read only community string
we can enumerate the OID MIB to get users on a windows machine
```shell
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.4.1.77.1.2.25
```
Other useful MIBs for Microsoft:
	1.3.6.1.2.1.25.1.6.0 System Processes
	1.3.6.1.2.1.25.4.2.1.2 Running Programs
	1.3.6.1.2.1.25.4.2.1.4 Processes Path
	1.3.6.1.2.1.25.2.3.1.4 Storage Units
	1.3.6.1.2.1.25.6.3.1.2 Software Name
	1.3.6.1.4.1.77.1.2.25 User Accounts
	1.3.6.1.2.1.6.13.1.3 TCP Local Ports
		One advantage of this is it will reveal ports that are only listening locally, so network scanning will not have discovered them