---
title: CyberSecLabs Eternal
date: 2020-12-29
categories: [CyberSecLabs]
thumbnail: /assets/img/csl/eternal/eternal.png
excerpt: This is a box that is on the easy side that makes use of the EternalBlue exploit.
tags: []
---

> If you have done Blue from HackTheBox this should seem pretty familiar to you.

-----------

## Enumeration

### Port scan

We will start of by running our nmap full port scan.

`nmap -sC -sV -T4 --max-rate 10000 -p- $IP -oN full`

![nmap](/assets/img/csl/eternal/nmap.png)

Judging by the box name _Eternal_ and from what the nmap script enumeration is showing _Windows 7 SP1_, we can assume this is vulnerable to the EternalBlue exploit. We can also run nmap again using the vuln script:

`nmap --script=vuln -p 445 $IP`

![vuln](/assets/img/csl/eternal/vuln.png)

### Gaining root

![msfconsole](/assets/img/csl/eternal/msfconsole.png)

We will `use exploit/windows/smb/ms17_010_eternalblue`

![exploit](/assets/img/csl/eternal/exploit.png)

Make sure to set your settings appropriately.

Once you've done that just simply run the exploit

![run](/assets/img/csl/eternal/run.png)

![root](/assets/img/csl/eternal/root.png)