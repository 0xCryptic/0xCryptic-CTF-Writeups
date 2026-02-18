# Nmap

`nmap -p21,22,80,111,139,445,2049 -A -T4 10.48.174.24`

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   3072 55:0d:ad:b2:f2:01:d6:82:9d:7c:23:8a:28:00:24:98 (RSA)
    |   256 df:56:04:98:f9:c2:69:10:7c:3b:08:02:67:8a:0d:25 (ECDSA)
    |_  256 90:c4:7e:ed:01:dc:41:56:7a:04:7a:7f:14:7d:11:09 (ED25519)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
    |_http-server-header: Apache/2.4.41 (Ubuntu)
    | http-robots.txt: 1 disallowed entry 
        |_/admin.html
    |_http-title: Site doesn't have a title (text/html).
111/tcp  open  rpcbind     2-4 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
2049/tcp open  nfs         3-4 (RPC #100003)
```
# Information Disclosure from Robots.txt 
`/admin.html` => A trap or honeypot 

# Samba Enumeration

SMB ran on top of NetBIOS using port 135 allowing same-network communication for Windows clients. 
Later versions of SMB began using port 445 on top of a TCP stack, allowing communication over the internet. 

`smbclient -L \\\\10.48.174.24 -U anonymous` 
```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
anonymous       Disk      
IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))
```
`smbclient //10.48.174.24/anonymous -U anonymous`
`get log.txt`

To recursively download an SMB share, the following command can be run: `smbget -R smb://10.48.174.24/anonymous`

A few interesting things can be found on the file, including: 
- Information generated for Kenobi when generating an SSH key for the user
- Information about the ProFTPD server.

"Earlier nmap scan showed port 111 running rpcbind, just a server that converts remote procedure call (RPC) program number into universal addresses. 
When an RPC service is started, it tells rpcbind the address at which it is listening and the RPC program number its prepared to serve.
In our case, port 111 is access to a network file system. Lets use nmap to enumerate this."

`nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.48.174.24`

```
PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-statfs: 
|   Filesystem  1K-blocks  Used       Available  Use%  Maxfilesize  Maxlink
|_  /var        9183416.0  5699764.0  2993056.0  66%   16.0T        32000
| nfs-showmount: 
|_  /var *
| nfs-ls: Volume /var
|   access: Read Lookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID  GID  SIZE  TIME                 FILENAME
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  .
| ??????????  ?    ?    ?     ?                    ..
| rwxr-xr-x   0    0    4096  2026-01-21T22:17:26  backups
| rwxr-xr-x   0    0    4096  2025-08-10T06:48:58  cache
| rwxrwxrwx   0    0    4096  2019-09-04T08:43:56  crash
| rwxrwsr-x   0    50   4096  2016-04-12T20:14:23  local
| rwxrwxrwx   0    0    9     2019-09-04T08:41:33  lock
| rwxrwxr-x   0    108  4096  2026-01-21T22:15:39  log
| rwxr-xr-x   0    0    4096  2025-08-09T13:38:21  snap
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  www
|_

Nmap done: 1 IP address (1 host up) scanned in 5.03 seconds
```


