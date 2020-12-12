---
title: HackTheBox OpenKeys
date: 2020-12-12
categories: [HackTheBox]
thumbnail: /assets/img/htb/openkeys/info.png
excerpt: This is a medium level box which made use of vim swap files to find an interesting file that turned out to be exploitable. Which we used to gain initial access through SSH, then used an OpenBSD exploit for PrivEsc.
tags: [openbsd,ssh,vim,gobuster,scp,php]
---

## Enumeration

### NMAP

![nmap](/assets/img/htb/openkeys/nmap.png)

### Website

Let's take a look at the website.

![website](/assets/img/htb/openkeys/website.png)

Basic login credentials didn't work.

Ran gobuster

![gobuster](/assets/img/htb/openkeys/gobuster.png)

We were able to find `/includes` which contained some files that we can grab.

![includes](/assets/img/htb/openkeys/includes.png)

We know that swap files are created when a vi/vim file is initiated.

`vim auth.php` showed nothing.

### Vi/Vim swap file

When we viewed the `.swp` file outright we just get random output:

![vimswp](/assets/img/htb/openkeys/vimswp1.png)

We can see some strings within that file:

![strings](/assets/img/htb/openkeys/strings.png)

And examining the file with `file` command:

![file](/assets/img/htb/openkeys/file.png)

At least we have a name to work with now.

Vim has a way to recover the file with the `-r` flag set.

`vim -r auth.php.swp`

> -r: Recover edit state for this file

We can see that the auth.php file makes use of something called `check_auth`. 

![check_auth](/assets/img/htb/openkeys/checkauth.png)

Looking up to see what it was, come to found out it is exploitable, you can read more about it here: [check_auth](https://thehackernews.com/2019/12/openbsd-authentication-vulnerability.html).

In essence, we can bypass authentication with a username set to `-schallenge` and the password can be anything.

`-schallenge:password`

I captured the request with Burp and sent it to repeater. I got stuck here and realized we can try to add the username of `jennifer` along with the cookie. We'll have to follow the redirection. 

![checkauth](/assets/img/htb/openkeys/ssh-key.png)

Now we have a private key which we can use to login to SSH.

We can do this with the browser's developer as well.

Just edit the value and add `;username=jennifer`

![developer](/assets/img/htb/openkeys/developer.png)

Refresh and login with the `-schallenge:password` and you'll be greeted with this:

![sshweb](/assets/img/htb/openkeys/sshweb.png)

### SSH

Now we can login to SSH with the private key 

Copy the key over to a file, I just named it jennifer.key and change the permissions.

`chmod 600 jennifer.key`

`ssh -i jennifer.key jennifer@openkeys.htb`

![ssh](/assets/img/htb/openkeys/jennifer.png)

`wget` was not on the box so since we are working in SSH we can use `scp`.

__On Attacker__

`scp -i jennifer.key linpeas.sh jennifer@openkeys.htb:/tmp`

Heading back over to our target, We are working with OpenBSD 6.6, there wasn't much else looking through linpeas.

![openbsd](/assets/img/htb/openkeys/openbsd.png)

### PrivEsc

So let's try to find an exploit for OpenBSD 6.6. There was a repository for various local exploits contained [here](https://github.com/bcoles/local-exploits).

The one I went with was CVE-2020-7247: [github](https://github.com/bcoles/local-exploits/blob/master/CVE-2020-7247/root66)

__On Attacker__

`scp -i jennifer.key root66 jennifer@openkeys.htb:/tmp`

__On Target__
```
chmod +x root66
./root66
```

![exploit](/assets/img/htb/openkeys/exploit.png)

![root](/assets/img/htb/openkeys/root.png)

There you have it, ROOT!