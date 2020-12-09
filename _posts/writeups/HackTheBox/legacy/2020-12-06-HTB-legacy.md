---
title: HackTheBox Legacy
date: 2020-12-06
categories: [HackTheBox]
thumbnail: /assets/img/htb/legacy/info.png
excerpt: This is an easy level box which is vulnerable to ms08_067.
tags: [ms08_067,samba,smb]
---

### NMAP

Ran a full port scan prior, then used those ports for a service and script scan.

> nmap -p139,445 -sV -sC -T4 -Pn 10.10.10.4 -oA 10.10.10.4

![nmap](/assets/img/htb/legacy/nmap.png)

Ran a vuln script with nmap:

![vuln](/assets/img/htb/legacy/vuln.png)


## Exploit

ms08-067 was likely vulnerable so we can try this with metasploit

```
use exploit/windows/smb/ms08_067_netapi 
```

![ms08](/assets/img/htb/legacy/ms08.png)

Root:

![root](/assets/img/htb/legacy/root.png)

User flag is in:

```
C:\Documents and Settings\john\Desktop
```

Root flag is in:

```
C:\Documents and Settings\Administrator\Desktop
```