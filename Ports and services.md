## 80, 8080 - http server
	can look for vulnerabilities against the service running the server
		Known vulnerable services:
		rejetto http file server <= 2.3
		webmin <= 1.580
	bruteforce any logins with hydra
	use burpsuite to recon any forms
	- if sending a reverse shell over web application make sure to url encode
## 145, 139 - Samba
	-try anonymous login
	-use smbmap to enumerate shares
	-bruteforce logins with hydra
	-are there any valuable files

## Services
### Jenkins
	- default username is admin
	- try script c 
