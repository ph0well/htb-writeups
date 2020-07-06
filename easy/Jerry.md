# Jerry by mrh4sh
![alt text](https://i.imgur.com/AUqZQAj.png)

## Initial recon
The initial nmap scan of the box shows only one service to investigate - a website on port 8080
```sh
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88
```
A full TCP scan doesn't reveal any new ports.

## The website
Navigating to the website reveals the default Apache Tomcat configuration page which also tells us the version of Tomcat that is running.
![alt text](https://i.imgur.com/o3bdHTV.png)

Some further enumeration of the website revealed 3 other directories
```sh
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.95:8080/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/07/06 10:19:18 Starting gobuster
===============================================================
/docs (Status: 302)
/examples (Status: 302)
/manager (Status: 302)
```

Browsing to any of these pages prompts you to login
![alt text](https://i.imgur.com/cTo7Ipa.png)

I tried to use several combinations of [default credentials](https://github.com/netbiosX/Default-Credentials/blob/master/Apache-Tomcat-Default-Passwords.mdown) without success which eventually landed me on an error page
![alt text](https://i.imgur.com/SxhuCL8.png)

As you can see in the screenshot above the error page leaks some credentials, I tried these on the login page and they worked which took me to the Tomcat Web Application Manager page
![alt text](https://i.imgur.com/qAtAe5t.png)

Looking around the page something that caught my eye was the ability to upload a WAR file
![alt text](https://i.imgur.com/m71wCgZ.png)

## Exploitation
As seen in the initial scan the webserver runs off of JavaServer Pages (JSPs), Tomcat exists to serve these pages which are distributed using WAR files. This means that it is possible to upload a JSP reverse shell disguised inside of a WAR file.

There are [several ways to do this](https://www.hackingarticles.in/multiple-ways-to-exploit-tomcat-manager/) but I chose to use msfvenom to create the malicious WAR file
![alt text](https://i.imgur.com/LSMI8Nv.png)

This can then be uploaded to the manager page and once you navigate to the malicious WAR file it will connect back to a listener on the port specified during creation
![alt text](https://i.imgur.com/2gyg4Wc.png)

The reverse shell is running as the root user so no privilege escalation is needed making it trivial to find both of the flags
![alt text](https://i.imgur.com/XHK79Pk.png)
