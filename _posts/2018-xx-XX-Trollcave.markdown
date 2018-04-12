---
layout: post
title:  "Trollcave: v1.2 created by ???"
date:   2018-04-06 12:19:21 +0700
categories: [ctf]
---
Other VMs of this series can be downloaded from [VulnHub] too.\\
Go check them out - there`re awesome!

# Enumeration
As we boot the VM in `bridged mode` there is an output on the console which shows us the IP address of the VM:`192.168.0.11`. \\
First, we start with an `nmap` scan to check for any open ports.

- `-sC: equivalent to --script=default`
- `-sV: Probe open ports to determine service/version info`
- `-O: Enable OS detection`

``` text

```


`dirb http://192.168.0.7/ /home/ben/CODE/HACKING/SecList/SecLists-master/big.txt`
``` text
---- Scanning URL: http://192.168.0.7/ ----
+ http://192.168.0.7/404 (CODE:200|SIZE:1564)                                                      + http://192.168.0.7/422 (CODE:200|SIZE:1547)                                                      + http://192.168.0.7/500 (CODE:200|SIZE:1477)                                                      + http://192.168.0.7/admin (CODE:302|SIZE:90)                                                      + http://192.168.0.7/comments (CODE:302|SIZE:90)                                                   + http://192.168.0.7/favicon.ico (CODE:200|SIZE:0)                                                 + http://192.168.0.7/inbox (CODE:302|SIZE:90)                                                      + http://192.168.0.7/login (CODE:200|SIZE:2208)                                                    + http://192.168.0.7/register (CODE:302|SIZE:85)                                                   + http://192.168.0.7/reports (CODE:302|SIZE:90)                                                    + http://192.168.0.7/robots.txt (CODE:200|SIZE:202)                                                + http://192.168.0.7/user_files (CODE:302|SIZE:90)                                                 + http://192.168.0.7/users (CODE:302|SIZE:90)
-----------------
```


![image-title-here](/static/img/portallogin.png){:class="img-responsive"}

# Privilege escalation USER

# Privilege escalation ROOT



[VulnHub]: https://www.vulnhub.com/entry/pinkys-palace-v1,225/
