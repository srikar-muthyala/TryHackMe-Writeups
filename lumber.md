## Lumberjack turtle
## IP: 10.10.249.237

## Enumeration
Nmap scan results
```
PORT   STATE SERVICE     VERSION
22/tcp open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6a:a1:2d:13:6c:8f:3a:2d:e3:ed:84:f4:c7:bf:20:32 (RSA)
|   256 1d:ac:5b:d6:7c:0c:7b:5b:d4:fe:e8:fc:a1:6a:df:7a (ECDSA)
|_  256 13:ee:51:78:41:7e:3f:54:3b:9a:24:9b:06:e2:d5:14 (ED25519)
80/tcp open  nagios-nsca Nagios NSCA
|_http-title: Site doesn't have a title (text/plain;charset=UTF-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Visiting webpage reveals nothing but some simple text,
![image](https://user-images.githubusercontent.com/31000079/150718042-0e32195f-a83d-4b75-9f51-718dd553d6c9.png)

