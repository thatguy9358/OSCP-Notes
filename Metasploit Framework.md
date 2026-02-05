# Getting familiar with metasploit
Metasploit doesn't enable database by default
```bash
sudo msfdb init
sudo systemctl enable postgresql
```
The above commands start the database and configure it to start at runtime.
```bash
sudo msfconsole
db_status
```
starts metasploit and gives database status

The database can be useful for tracking results during a pentest. There are tools like db_nmap that execute a common tool and directly import results to the database. We can then query these results using the services (and other) command(s)

# Axillary Modules
Provide functionalities such as protocol enumeration, port scanning, fuzzing, sniffing, and more. Auxiliary modules are useful for many tasks, including information gathering (under the gather/ hierarchy), scanning and enumeration of various services (under the scanner/ hierarchy), and so on.
We can search aux modules specifically using commands like
```bash
search type:auxiliary smb
```
We can set options for modules manually, or we can auto populate using database results
```bash
services -p 445 --rhosts
```
after running scanners we can see if metasploit automatically detected vulnerabilities by running
`vulns`
# Exploit Modules
once we run an exploit, if we receive an interactive session we can background it with crtl+z, and switch between other active sessions
Use `sessions -l` to list all active sessions
Use `sessions -i <target>` to switch sessions
Use -k flag to kill sessions

# Metasploit payloads
Non staged payload is sent in its entirety along with the exploit --> these "all in one" payloads are generally more stable
Staged payloads usually have two parts, a small primary payload that connects back to the attacking machine and larger transfer once connection is made. Then the shell code is executed --> could be advantageous if there is size concern or anti virus active since anti virus can detect shell code

all meterpreter payloads are staged, "non staged" ones just contain the full shellcode for meterpreter connection in the initial payload
once we have a meterpreter session we can run `help` to get a list of commands
We can create new channels by running `shell`
Channels are all contained within one session
we can look at fIle system commands to view commands related to upload/download of files
Meterpreter has an https shell option which can be good for IDS evasion

# msfvenom
we can use -l to list available options for various settings
only metasploit multi handler can handle staged payloads


# Post Exploitation
Meterpreter has a lot of post exploit functions. More on windows than linux though
On windows we can try to become the system user by using the `getsystem` command --> we need to check if the SeImperosnate and SeDebug privileges are set. We can use `whoami /priv` for that
`migrate` can be used to takeover running proccesses --> this is useful because if  a user kills the process we have compromised, we will lose connection. We can switch to system level or more mundane processes to decrease chance of that happening
*We can only migrate to processes at or below our current privilege level*
`execute` will create a new process we can migrate to
For example we can make a hidden process using the following command
```
execute -H -f notepad
```
 metasploit also has modules that can help elevate privilege --> if we search UAC we get plenty of results for bypassing UAC on windows
 "bypassuac_sdclt" is an effective one
 `load kiwi` will load mimikatz into a meterpreter session
# Pivoting
`route add` can be used to add routes through meterpreter
```bash
route add <ip range> <session id>
```
Once we add rotues we can use axillary tools to recon the new network and exploits for any new targets
*Note route add will only work with established connections. I f we exploit new targets through an added route, we need to use bind shells*
`autoroute` module will automatically add new routes given an active session id
We can set a proxy server by running `use auxiliary/server/socks_proxy` and setting the options --> we can then edit the proxychains config to use our meterpreter proxy server --> noe we can run commands using proxychains like in port redirecting and tunneling
`portfwd` will set up a port forward
```bash
portfwd add -l 3389 -p 3389 -r 172.16.5.200
```
the above command sets up a port forward on localhost 3389 to a remote rdp port. In this example we can now connect using the command
```bash
sudo xfreerdp /v:127.0.0.1 /u:luiza
```

# Resource Scripts
Resource scripts can chain together a series of metasploit console commands and ruby code
