# Must Do
nmap all ports tcp: `nmap -Pn -p- -O -A $ip -o nmap_IP_aggresive`
nmap UDP ports: `sudo nmap -sU %ip -o nmap_IP_udp`
	follow up with -sV for open ports
**Identify service versions for any service**
	use searchsploit and google to see if there are known vulns
	if one exploit doesn't work simply try another
**note commands that work and get screenshots**
**Be through in trying to exploit** --> exhaust all options before moving on
	try multiple ports to get a callback
	try multiple payloads for reverse shells
	try multiple ways of moving files
	try multiple places for storing the file
		Linux: /dev/shm , /tmp , ./
		Windows: C:\\Users\\Public , C:\\WIndows\\Temp , .\
**Enumerate everything before exploiting**

==revert boxes and double check steps before continuing==
# General
Try using usernames as passwords
Search for default creds
Random looking strings might just be passwords if they are not crackable
be wary of rabbit holes --> if something isn't working, but it "should" do a double take
	like not finding a referenced subdomain or not cracking a password hash that is actually just a random string
**Track links to any premade exploit used and any edits made**


# FTP
anonymous access `ftp anonymous@$ip`
bruteforce/spray creds if we know usernames
see if we can get/put files with access
	If we put files, are they accessible anywhere else like a web page?
# SMB
anonymous Access `smbmap -H $ip -u anonymous -p anonymous`
try spraying any other credentials
smbmap can map the whole file tree if we have access
crackmapexec can execute commands to get files --> useful for ones with special characters
can try to establish shells with smbexec/psexec
nbtscan
# SMTP
might be able to enumerate valid users, see notes on info gathering
can send client side attacks with valid creds --> swaks


# SNMP
brute force the community string --> commonly public
snmpwalk to enumerate through the whole mib tree
potentialy get  a reverse shell through extended MIBs
Check extended mibs for sensitive info

# RPC
check for null sessions
enum4linux
rid cycling to identify users --> try with a larger range if none found intially
# Web app
Pay attention to all details output by nmap scan
	google everything for exploits, even if ti looks basic/custom
try to enumerate everything
## Enumeration
Gobuster can be used for brute forcing directories AND vhost
brute force any directories we find --> even if they are 403
we need a host entry in /etc/hosts/ to bruteforce vhosts
websites may not load properly, if at all without host entry
Website scanners can be useful, use one specific to technology if found
	wpscan - wordpress
	nikto - general
	joomscan - joomla
Make note of all possible injection points for user data
Identify any cms running
view source code and common pages like sitemap.xml and robots.txt
php wrappers exist, they may provide additional functionality
## Command Injection
In an AD environment, see if we can get the service to call back to us and get the hash
	http
	smb
	ftp
try multiple delimiters
## LFI/Path traversal
Can we pull in ssh keys on linux
can we pull in configuration files
try in repeater if we see a "?page=..." anyhwere
log poisoning --> we can edit request so that logged info will render as a shell if included int he browser

## File Upload
try to upload reverse shells
	might need to avoid filters with encodings / other tricks
if we can upload unknown file types try uploading a .httaccess file to map the unknown extension to a scripting language
if we can't reach the file, we can try combining file upload with path traversal to blindly overwrite files --> try to overwrite authorized keys


## SQL injection
try to break down the full query --> write it out
Seclists has wordlists for sqli syntax
Don't panic, read errors and don't get flustered
MySql we qant to try and write a php shell out to a file
MsSql we want to try xpcmdshell
Remember some sqli is blind --> we can try RCE payloads anyways

# Privilege Escalation
Read the winpeas /linpeas output carefully
If all else fails try watching processes using pspy or watchdog
make note of anyservices listening on local hosts

## Linux
Read linpeas carefully
su to to other users as hail mary
## Windows
Read winpeas carefully


# Active Directory
Enumerate your ass off
	backup files
	config files --> especially non native programs
	databases
	sekurlsa::logonpasswords
	sekurlsa::tickets
	lsadump::sam
	lsadump::lsa
	any log or text fie in user directories
	git files
	
make sure to do all post exploitation before continuing
may need to tunnel connections back to Kali to receive callbacks from exploits at the second level
Try multiple tools for password spraying if not getting results --> all else fails just try using the exploit tools with known cred pairs
