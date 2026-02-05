See if any secret information is added to the code
	- passwords
	- ssh keys
	- API keys
	- urls w/ authentication info

What are the packages or modules being pulled in?
 - out of date modules can be a source of vulnerability
 - are there 3rd party libraries
	 - at best you are relying on soemone else to maintain secure coding practices
	 - at worst someone could put in a hidden vuln

Are any files included using the absolute or the relative path?
- if files only use the relative path an attacker can spoof whatever is being called to get malicious code to run
- on implicit imports if the module is found in the system path it will be pulled in from there, which might not be the intended  path
	- an attacker could put the "module" somewhere in the system path to et it imported into the program

Sanitize inputs
- it is very bad practice to not sanitize user input.
- Depending on where the input is used a user could include input for sql injection, file traversal, or RCE
	- Sql injectiom

Templates can be used to prevent string formatting vulnerabilities in python code
	- another way to sanitize user input data
	- example of string format abuse:
CONFIG = {

    “API_KEY”: “secret_key”
}
class User:

    name = “”

    email = “”

    def __init__(self, name, email):

        self.name = name

        self.email = email

    def __str__(self):

        return self.name

name = “Toby”

email = “oyetoketoby80@gmail.com”

user = User(name, email)

print(f”{user.__init__.__globals__[‘CONFIG’][‘API_KEY’]}”) <-- here we see the user supplied user."--init--.--globals--['CONFIG']['API_KEY ']" to print the secret value

/* secret_key */

Python 2 is EOL
- also supports implicit relative imports which is bad

OWASP top 10 ish vulns
Path traversla attack: allowing users to pass in a path to a file makes it vulnerable to path traversal injection. An attacker could get a file from anywhere on the system
	example:
	**def get_video(self, path=None):**  
		self.check_user_auth()  
		data = None  
		if not path:  
			path = self.get_video_path()  
			path = path[0] if path else None  
		if path:  
			with open(path, 'rb') as f:  
			data = bytearray(f.read())  
		return data
	==in this code the user can supply the path to any file on the system. Could recover /etc/passwd with the command get_video('etc/passwd')==
	*Fix:* build the path on the server side instead of trusting user input. Use function call to find correct path of video


OS Command Injection: Using unsantiized input value as a parameter in an OS command call might allow an attacker to inject malicious OS commands
	example:
	def insert_from_yaml(self, client_yaml):
	client = yaml.safe_load(**os.popen('cat %s' % client_yaml))**
	if type(client) is not dict:  
            e = ValueError('File %s could not be loaded.' % client_yaml)  
            logger.error('%s' % e)  
            raise e
    ==In this code the os level call is made to open the file using input sent by the user==
    *Fix*: Use lagunage libraries instead of OS calls when applicable. In this case use "open" instead of 'Cat'
    another example:
    try:  
		**os.system('wget -O %s %s' % (json_file, json_url))**  
		with open(json_file) as f:  
		providers = json.load(f)  
		os.remove(json_file)  
		except Exception as e:  
			logger.error('Could not GET the JSON URL - %s' % e)  
			raise e  
			# Insert Providers JSON list  
		self.insert(providers)
		==In this code it uses the OS level commands to download a json file. An attacker could replace the json_url value with a malicious expression (eg. rm -fr /) to be executed==
		*Fix:* In order to mitigate, OS command injection attacks the program must not execute inputs as OS commands from untrusted sources to read files. For this example replpace wget with requests.get()
	How to identify:
		- subprocess module
		- os.<whatever>
		- 

SQL injection
	Unsanitized user input into a sql data base can lead to injection attacks.
	On login screens if username and password are not sanitized an attacker can format a string to gain unauthorized access
		vulnerable code:  cursor.execute("SELECT * FROM users WHERE username = '%s' AND password = '%s'" % (username, password))
		Injection attack: OR 'a'='a';-- 
		this will end the username field and inject the code OR a=a which will return true
		;-- will comment out the rest of the field so the AND password... will not be included
		this leaves the program with 
	*Fix:* Use python libraries that sanitize input. Also use parameters. Providing user input with single quotes around it will pass it directly to the database. Instead put %s or %(username)s as the named parameter to pass it as the intended format.
	BAD EXAMPLES:
	cursor.execute("SELECT admin FROM users WHERE username = '**" + username + '**");
	cursor.execute("SELECT admin FROM users WHERE username = **'%s'** % username);
	cursor.execute("SELECT admin FROM users WHERE username = **'{}'**".format(username));
	cursor.execute(f"SELECT admin FROM users WHERE username = **'{username}'**");
	GOOD EXAMPLES: 
	cursor.execute("SELECT admin FROM users WHERE username = **%s**'", (username, ));
	cursor.execute("SELECT admin FROM users WHERE username = **%(username)s**", {'username': username});

Attacks

	'; DROP TABLE USERS; --     will try and drop the entire user table database
	You can query `information_schema.tables` to list the tables in the database:
	SELECT * FROM information_schema.tables to get table names
	SELECT * FROM <table name> to get info