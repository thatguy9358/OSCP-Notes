==If searchsploit exploits aren't working search online for other ones== --> search all exploits to begin with and pick the best sounding one

For webshell to interactive shell make sure we try all possible avenues --> ==common ones may not work, but others might, so keep trying==

GTFO bins may not have every way to exploit binaries for cron jobs --> might need to do more research on internet if we see a potentially vulnerable cron job

On web pages google the http-title name + "exploit" even if it looks like a custom page

might have to go multiple layers deep with gobuster findings, even on pages that initially don't return results, but exist

Try tunning reverse shells back to ports we think should be open like 443 --> especially if shells that should work aren't connecting back

run UDP scans with nmap

if shells aren't executing, but they have been pulled onto remote host try a different known port. If not you might need to try a different path to store them

==crackmapexec has the ability to execute commands/download/upload files on target ips== --> this is helpful when we are trying to retreive files from internal machines. It avoids the need to open another tunnel for data transfer back to kali
	ex: proxychains crackmapexec mssql 10.10.103.147 -u 'sql_svc' -p 'PasswordHere' --get-file "C:\windows.old\windows\system32\SYSTEM" SYSTEM.

local creds will not authenticate using crackmap exec unless they are reused by a domain user. If we find local account creds on one domain computer they may be valid for a local account on another computer. We can try using them with the exploit tool instead of crackmapexec

we can see if searchsploit contains exploits to create malicious filetypes like ODT

==If one exploit does not work, we can look for more==

enum4linux works on both windows and linux

we can brute force uplaoded file names if they follow a convention

if full names are present on a website/service we can try to make usernames to as-rep roast
	can use tool username anarchy

We can use xp_dirtree to browse file system --> can potentially identify files accessible via web/other shares like backups and configs

Furthermore, we can also combine file upload mechanisms with XML External Entity (XXE)424 or Cross Site Scripting (XSS)425 attacks. For example, when we are allowed to upload an avatar to a profile with an SVG426 file type, we may embed an XXE attack to display file contents or even execute code.

Retry all login opportunities when new credentials are found, i.e web pages, smb. winrm, sql, etc