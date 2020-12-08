# TryHackeMe TomGhost

---

## Nmap results

![nmap](nmap.png)

As we can see `ports 22, 53, 8009, 8080` are open

`Port 8080` is a default Tomcat instance. Running Gobuster on this port didn't yield anything. Let's look into `port 8009`

---

What is AJP?

The Apache JServ Protocol (AJP) is a binary protocol that can proxy inbound requests from a web server through to an application server that sits behind the web server.

[AJP Wiki](https://en.wikipedia.org/wiki/Apache_JServ_Protocol)

---

## Searchsploit

We see there's one exploit that stands out, and judging by the name of the box, we want 
`Apache Tomcat - AJP 'Ghostcat File Read/Inclusion`.

![exploitdb](exploitdb.png)

We can grab the file with `searchsploit -m <path of exploit>`

`searchsploit -m exploits/multiple/webapps/48143.py`

Running the exploit `python 48143.py <target>`.
We are able to gather some creds
![creds](sshcreds.png)

The only place we can use these is through SSH.

---

## SSH
Let's login in with the newly found creds!

`skyfuck:8730281lkjlkjdqlksalks`

Bingo!!

![user](user.png)

## User

Our user flag will be in `/home/merlin`

---

## Gaining Root

If we go back into `/home/skyfuck` We can see in our current directory, we have 2 files: One being an ASCII armor file which is the PGP key and the other being the PGP file.

![credential](credential.png)

The credential.pgp will need a password to view the contents which we don't have yet. We will have to get the key file `tryhackme.asc` onto our host and use trusty John. One way we can do this is by using secure copy or `scp`.

On host machine:
`scp skyfuck@<IP of target>:/home/skyfuck/* .`
This will get all files(`*`) from the skyfuck directory and place it in your current working directory(`.`).

We will now use gpg2john on `.asc` file `gpg2john tryhackme.asc > hash`.
John will be able to crack this using `john --wordlist=/usr/share/wordlists/rockyou.txt <name of file>`

![cracked](crack.png)

And now we have a password we should be able to decrypt the `pgp` file using this password.

![decrypt](decrypt.png)

Now we can switch over to merlin's account and check his sudo privileges:

![privileges](privs.png)

Looks like we can use `zip` as sudo with no password. Checking on [GTFOBins](https://gtfobins.github.io/gtfobins/zip/#sudo), we can use this command:

![zip](zip.png)

---

## Root

And now we have root.

![root](root.png)

Root flag will be in `/root`
