# Valley
Can you find your way into the Valley?

## Recon 

`sudo nmap -A -T4 10.49.150.196`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c2:84:2a:c1:22:5a:10:f1:66:16:dd:a0:f6:04:62:95 (RSA)
|   256 42:9e:2f:f6:3e:5a:db:51:99:62:71:c4:8c:22:3e:bb (ECDSA)
|_  256 2e:a0:a5:6c:d9:83:e0:01:6c:b9:8a:60:9b:63:86:72 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

`gobuster dir -u http://10.49.150.196 -w /usr/share/wordlists/dirb/common.txt`

```
.htaccess            (Status: 403) [Size: 278]
.hta                 (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
gallery              (Status: 301) [Size: 316] [--> http://10.49.150.196/gallery/]
index.html           (Status: 200) [Size: 1163]
pricing              (Status: 301) [Size: 316] [--> http://10.49.150.196/pricing/]
server-status        (Status: 403) [Size: 278]
static               (Status: 301) [Size: 315] [--> http://10.49.150.196/static/]
```
- Accessing http://10.49.150.196/pricing/ reveals an internal note addressed to "J" from "RP": 

```
J,
Please stop leaving notes randomly on the website
-RP
```

`gobuster dir -u http://10.49.150.196/static -w /usr/share/wordlists/dirb/common.txt` 

```
.hta                 (Status: 403) [Size: 278]
.htaccess            (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
00                   (Status: 200) [Size: 127]
11                   (Status: 200) [Size: 627909]
3                    (Status: 200) [Size: 421858]
12                   (Status: 200) [Size: 2203486]
10                   (Status: 200) [Size: 2275927]
1                    (Status: 200) [Size: 2473315]
5                    (Status: 200) [Size: 1426557]
15                   (Status: 200) [Size: 3477315]
9                    (Status: 200) [Size: 1190575]
13                   (Status: 200) [Size: 3673497]
14                   (Status: 200) [Size: 3838999]
2                    (Status: 200) [Size: 3627113]
6                    (Status: 200) [Size: 2115495]
7                    (Status: 200) [Size: 5217844]
8                    (Status: 200) [Size: 7919631]
4                    (Status: 200) [Size: 7389635]
```

- Accessing http://10.49.150.196/static/00 reveals another internal note:

```
dev notes from valleyDev:
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
```

- A developer login page is available at `http://10.49.150.196/dev1243224123123/`.
- Source code analysis of the login page reveals the use of two scripts: `dev.js` and `button.js` 

`dev.js` contains hardcoded credentials for the user `siemDev`. Upon login, this redirects the user to another note file: 

```
dev notes for ftp server:
-stop reusing credentials
-check for any vulnerabilies
-stay up to date on patching
-change ftp port to normal port
```

- An nmap scan can be performed to check which port FTP is hosted on: `sudo nmap -A -T4 -p- 10.49.150.196` 
- FTP can be accessed with the previously found credentials, where three files are stored: `siemFTP.pcapng`, `siemHTTP1.pcapng` and `siemHTTP2.pcapng`. 
- In the Wireshark file `siemHTTP2.pcapng`, a POST request can be viewed which contains credentials for initial access. 

## Initial Access
- Performed user enumeration, listing all contents of the `/home` directory. An authenticator ELF file : 

```
valleyDev@valley:/home$ ls -al
total 752
drwxr-xr-x  5 root      root        4096 Mar  6  2023 .
drwxr-xr-x 21 root      root        4096 Mar  6  2023 ..
drwxr-x---  4 siemDev   siemDev     4096 Mar 20  2023 siemDev
drwxr-x--- 16 valley    valley      4096 Mar 20  2023 valley
-rwxrwxr-x  1 valley    valley    749128 Aug 14  2022 valleyAuthenticator
drwxr-xr-x  5 valleyDev valleyDev   4096 Mar 13  2023 valleyDev
valleyDev@valley:/home$ file valleyAuthenticator
valleyAuthenticator: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
```

## Lateral Movement 
- Reading human-readable text contained within the binary with `strings valleyAuthenticator` indicates that it is a UPX encoded file. 
- Decompress the file: `upx -d valleyAuthenticator`
- Upon further analysis of the file with `strings valleyAuthenticator`, the file includes crackable hashed credentials for the user `valley`.

- Switched to the user: `su valley` and entered the password. SSH access as `valley` has been obtained.

# Privilege Escalation

- A cronjob is present on the user, scheduled to run every minute for everyday of every month. The cronjob runs a python script `photosEncrypt.py` which imports base64. 
- The Python script itself cannot be modified as there is no set UID bit for the user `valley`. However, `/usr/lib/python3.8/base64.py`can be edited to spawn a reverse shell. 
- Root flag obtained.
