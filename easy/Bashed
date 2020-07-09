# Bashed by Arrexel

## Initial recon
The initial nmap scan of the box reveals only one service - a webserver running on port 80
```sh
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site
```

A full TCP port scan didnt't reveal any new ports.

## Enumerating the website
When you first visit the website you are greeted with a blogpost about a PHP webshell. Reading carefully you'll see that the author mentions that it was developed on that webserver, so it has to be somewhere right?

Running gobuster to brute force directory results yields some interesting results
```sh
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.68/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/07/09 07:49:12 Starting gobuster
===============================================================
/images (Status: 301)
/uploads (Status: 301)
/php (Status: 301)
/css (Status: 301)
/dev (Status: 301)
/js (Status: 301)
/fonts (Status: 301)
```
The two which stick out to me are uploads and dev, uploads is empty but dev contains a link to the PHP webshell mentioned in the blog post.

## Initial foothold
Navigating to the webshell allows us to interact with the webserver as a low privilege user, www-data
```sh
www-data@bashed:/var/www/html/dev# whoami

www-data
```
TO DO 

## Privilege Escalation
TO DO
