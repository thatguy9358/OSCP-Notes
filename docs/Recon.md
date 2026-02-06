# Recon
Reconnaissance

WHen finding web servers try and brute force any sub directories to find credentials, submission forms, etc.

For webservers manually enumerate discovered directories to look for underlying service and version 

on wep apps we can also use vhost fuzzing to bruteforce subdomains of the webpage. subdomains may reveal additional attack surfaces. gobuster has the ability to vhost fuzz

If a web page redirects add the IP and hostname into the etc/hosts file for it to properly load --> you can identify this by not being able to connect to a website/subdomain you can ping, but cannot connect to in browser

on webapps we can check all user influenced fields in post requests for injection vulnerabilities. Getting a "500" Internal server error is a good indication that the field may be vulnerable to injection

Check Desktop for user flags

## Active Directory
Once initial foothold is gained on an AD host there are several methods we can use to enumerate the AD structure
### credential injection
credential injection will allow you to run commands / programs on over the network using the authenticated AD credentials recovered
using the command
```cmd
runas.exe /netonly /user:<domain>\<username> cmd.exe
```
this will open up a command prompt. any further programs started/commands given will use the creds provided if they are going over the network
you may need to configure the DNS server of the machine you are running on. Safest bet is to use the Domain controller. We can configure dns in powershell as follows:
```powershell
$dnsip = "<DC IP>"
$index = Get-NetAdapter -Name 'Ethernet' | Select-Object -ExpandProperty 'ifIndex'
Set-DnsClientServerAddress -InterfaceIndex $index -ServerAddresses $dnsip
```
Ethernet will be the name of the network interface you are using
check credentials properly authenticate by running
```cmd
dir \\za.tryhackme.com\SYSVOL\
```
All users in AD should be able to access SYSVOL.
These injected credentials can now be used to authenticate to any web app that uses NTLM Authentication
### Microsoft Management Console
Microsoft Management console can be used through the GUI to inspect the AD structure
Using injected credentials we can launch MMC through the cmd line
NOTE: this method uses the Remote Server Administration Tools' (RSAT) AD Snap-Ins. These may or may not be present on the system.
n MMC, we can now attach the AD RSAT Snap-In:
    Click File -> Add/Remove Snap-in
    Select and Add all three Active Directory Snap-ins
    Click through any errors and warnings
    Right-click on Active Directory Domains and Trusts and select Change Forest
    Enter za.tryhackme.com as the Root domain and Click OK
    Right-click on Active Directory Sites and Services and select Change Forest
    Enter za.tryhackme.com as the Root domain and Click OK
    Right-click on Active Directory Users and Computers and select Change Domain
    Enter za.tryhackme.com as the Domain and Click OK
    Right-click on Active Directory Users and Computers in the left-hand pane
    Click on View -> Advanced Features

If everything up to this point worked correctly, your MMC should now be pointed to, and authenticated against, the target Domain
From here we can manually enumerate users and computers through the GUI. Additionally the search function can be used for rapid results.
If the user has AD privileges, you can update AD objects or add new ones
CONS:
- requires RDP access
- No efficient way to gather AD wide properties or attributes
### Command Prompt Enumeration
Command prompt can be useful to provide stealthy info about the AD structure. 
the "net" command allows us to enumerate AD and Local system info from command line
```cmd
net user /domain
```
The above command will list all users in the domain
```cmd
net user <username> /domain
```
The above command will display further info about a specific user
We can also use net to enumerate groups on the AD
```cmd
net group /domain
```
We can also get more detail about a group using the same method as the user sub option

Password Policy can be queried using the net command. We need to use the accounts sub-option
```cmd
net accounts /domain
```
The above command will provide the password policy for the domain. We can use this to figure out some useful info including:
- Length of password history kept. Meaning how many unique passwords must the user provide before they can reuse an old password.
- The lockout threshold for incorrect password attempts and for how long the account will be locked.
- The minimum length of the password.
- The maximum age that passwords are allowed to reach indicating if passwords have to be rotated at a regular interval.
Pros:
- No additional tools required, and simple commands are often not monitored by blue teams
- No GUI needed
- These commands can be included in phishing payloads
Cons:
- commands must be executed froma domain-joined machine. If not the default WORKGROUP will be domain
### Powershell



## Tools
Nmap - enumerate all services runnign on a target machine
	Can do quick scan of ports to get started and then do all higher ports in background
	nmap -sV --script vuln <ip> --> check for any known vulnerabilities
	-Pn will get results if host is blocking ICMP

	-------------Scripts---------------
	nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.121.196.
		enumerate smb shares
		can use if we see port 445 open on windows or linux machine
    nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.121.196
		gets more info on NFS


GoBuster - tool used to brute-force URIs (directories and files), DNS subdomains and virtual host names.
	GoBuster flag	Description
	-e	 Print the full URLs in your console
	-u	The target URL
	-w	Path to your wordlist
	-U and -P	Username and Password for Basic Auth
	-p <x>	Proxy to use for requests
	-c <http cookies>	Specify a cookie for simulating your auth

smbclient -used to connect to smb shares
	ex: smbclient //10.10.121.196/anonymous

rdesktop - tool used to establish RDP connections to remote hosts
	-u: user name
   -d: domain
   -s: shell / seamless application to start remotely
   -c: working directory
   -p: password (- to prompt)
   -n: client hostname
   -k: keyboard layout on server (en-us, de, sv, etc.)
   -g: desktop geometry (WxH[@DPI][+X[+Y]])
   -i: enables smartcard authentication, password is used as pin
   -f: full-screen mode
   -b: force bitmap updates
   -L: local codepage
   -A: path to SeamlessRDP shell, this enables SeamlessRDP mode
   -V: tls version (1.0, 1.1, 1.2, defaults to negotiation)
   -B: use BackingStore of X-server (if available)
   -e: disable encryption (French TS)
   -E: disable encryption from client to server
   -m: do not send motion events
   -M: use local mouse cursor
   -C: use private colour map
   -D: hide window manager decorations
   -K: keep window manager key bindings
   -S: caption button size (single application mode)
   -T: window title
   -t: disable use of remote ctrl
   -N: enable numlock synchronization
   -X: embed into another window with a given id.
   -a: connection colour depth
   -z: enable rdp compression
   -x: RDP5 experience (m[odem 28.8], b[roadband], l[an] or hex nr.)
   -P: use persistent bitmap caching
   -r: enable specified device redirection (this flag can be repeated)
         '-r comport:COM1=/dev/ttyS0': enable serial redirection of /dev/ttyS0 to COM1
             or      COM1=/dev/ttyS0,COM2=/dev/ttyS1
         '-r disk:floppy=/mnt/floppy': enable redirection of /mnt/floppy to 'floppy' share
             or   'floppy=/mnt/floppy,cdrom=/mnt/cdrom'
         '-r clientname=<client name>': Set the client name displayed
             for redirected disks
         '-r lptport:LPT1=/dev/lp0': enable parallel redirection of /dev/lp0 to LPT1
             or      LPT1=/dev/lp0,LPT2=/dev/lp1
         '-r printer:mydeskjet': enable printer redirection
             or      mydeskjet="HP LaserJet IIIP" to enter server driver as well
         '-r sound:[local[:driver[:device]]|off|remote]': enable sound redirection
                     remote would leave sound on server
                     available drivers for 'local':
                     alsa:      ALSA output driver, default device: default
         '-r clipboard:[off|PRIMARYCLIPBOARD|CLIPBOARD]': enable clipboard
                      redirection.
                      'PRIMARYCLIPBOARD' looks at both PRIMARY and CLIPBOARD
                      when sending data to server.
                      'CLIPBOARD' looks at only CLIPBOARD.
         '-r scard[:"Scard Name"="Alias Name[;Vendor Name]"[,...]]
          example: -r scard:"eToken PRO 00 00"="AKS ifdh 0"
                   "eToken PRO 00 00" -> Device in GNU/Linux and UNIX environment
                   "AKS ifdh 0"       -> Device shown in Windows environment 
          example: -r scard:"eToken PRO 00 00"="AKS ifdh 0;AKS"
                   "eToken PRO 00 00" -> Device in GNU/Linux and UNIX environment
                   "AKS ifdh 0"       -> Device shown in Microsoft Windows environment 
                   "AKS"              -> Device vendor name                 
   -0: attach to console
   -4: use RDP version 4
   -5: use RDP version 5 (default)
   -o: name=value: Adds an additional option to rdesktop.
           sc-csp-name        Specifies the Crypto Service Provider name which
                              is used to authenticate the user by smartcard
           sc-container-name  Specifies the container name, this is usually the username
           sc-reader-name     Smartcard reader name to use
           sc-card-name       Specifies the card name of the smartcard to use
   -v: enable verbose logging

