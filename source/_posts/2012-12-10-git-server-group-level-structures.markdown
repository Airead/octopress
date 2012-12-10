---
layout: post
title: "Git server group-level structures"
date: 2012-12-10 09:57
comments: true
categories: linux-tools git
---

If you use a small number of git, you can use the following steps to quickly deploy a git server environment. For example, as I am now the situation to two people on assignment to Qita company to do projects, but to do something that is completely independent of the time in addition to the integration of other projects with the company code group interaction, and usually can not be free to interact with the code, alas, shit's security, other times on my two visits. 

<!--more-->
## SSH public key generation
Each need to use git server engineers, they need to generate a ssh public key into your ~ /. Ssh directory to see if a file name and file name to use. Pub named after a pair of files, the file name is usually id_dsa or id_rsa . . Pub is the public key file, another file is key. Without these files (or simply connected. Ssh directory are not), you can use the program ssh-keygen to create them, the program in the Linux / Mac systems provided by the SSH package in Windows, is included in MSysGit bag:  

<code>
$ ssh-keygen  
Generating public/private rsa key pair.  
Enter file in which to save the key (/Users/schacon/.ssh/id_rsa):   
Enter passphrase (empty for no passphrase):   
Enter same passphrase again:   
Your identification has been saved in /Users/schacon/.ssh/id_rsa.  
Your public key has been saved in /Users/schacon/.ssh/id_rsa.pub.  
The key fingerprint is:  
43:c5:5b:5f:b1:f1:50:43:ad:20:a6:92:6a:1f:9a:3a schacon@agadorlaptop.local  
</code>  

It first asks you to confirm the position of saving the public key (.ssh / id_rsa), then it will let you repeat a password twice, if you do not want to enter a password when using the public key can be left blank. 

Now, all users have done this step had to put their public key to your server administrator or Git (assuming the service is configured to use SSH public key mechanism). They only need to copy. Pub file contents and then e-email it. The way the public key as follows: 

<code>
$ cat ~/.ssh/id_rsa.pub  
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAklOUpkDHrfHY17SbrmTIpNLTGK9Tjom/BWDSU
GPl+nafzlHDTYW7hdI4yZ5ew18JH4JW9jbhUFrviQzM7xlELEVf4h9lFX5QVkbPppSwg0cda3
Pbv7kOdJ/MTyBlWXFCR+HAo3FXRitBqxiX1nKhXpHAZsMciLq8V6RjsNAQwdsdMFvSlVK/7XA
t3FaoJoAsncM1Q9x5+3V0Ww68/eIFmb1zuUFljQJKprrX88XypNDvjYNby6vw/Pb0rwert/En
mZ+AW4OZPnTPI89ZPmVMLuayrD2cE86Z/il8b+gw3r3+1nKatmIkjn2so1d01QraTlMqVSsbx
NrRFi9wrf+M7Q== test@agadorlaptop.local  
</code>  

## Set up the server
First, create a 'git' user and to create a. Ssh directory in the user home directory:  

<code>
$ sudo adduser git  
$ su git  
$ cd  
$ mkdir .ssh  
</code>  

Next, the developer of the SSH public key added to the user's authorized\_keys file. Suppose you received via e-mail co-existence of several public key to a temporary file. Authorized\_keys file as long as they are added 

<code>
$ cat /tmp/id_rsa.curtis.pub >> ~/.ssh/authorized_keys  
$ cat /tmp/id_rsa.dylan.pub >> ~/.ssh/authorized_keys  
</code>  

Now you can use the-bare option to run git init to set an empty warehouse, which will initialize a repository does not contain a working directory. 

<code>
$ cd /opt/git  
$ mkdir project.git  
$ cd project.git  
$ git –bare init  
</code>  

At this time, developers can add it to remote storage, push a branch to the first version of the project was uploaded to the warehouse. It should be noted that every time you add a new project requires the host through the shell login and create a pure warehouse. We may wish to gitserver as the git user and the warehouse where the host name. If you run the host within the network, and set gitserver in the DNS to point the host, then the following commands are available. 

<code>
$ cd myproject  
$ git init  
$ git add .  
$ git commit -m ‘initial commit’  
$ git remote add origin git@gitserver:/opt/git/project.git  
$ git push origin master  
</code>  

Thus, cloning and push other people the same become very simple: 

<code>
$ git clone git@gitserver:/opt/git/project.git  
$ vim README  
$ git commit -am ‘fix for the README file’  
$ git push origin master  
</code>  

Using this method can be very quick to set up a small number of developers can read and write Git services. 

As an extra precaution, you can use Git comes with a simple tool to git-shell git users to limit the activities associated only with Git. Make it git user login shell, then the user can not have a normal shell access to the host. To achieve this, the need to specify the user's login shell is a git-shell, rather than bash or csh. You may have to edit / etc / passwd file  

`$ sudo vim /etc/passwd`

In the end of the file, you should be able to find the line like this:  

`git:x:1000:1000::/home/git:/bin/sh`

The bin / sh to / usr / bin / git-shell (or, which git-shell to see its location). Modified the way the bank is as follows:  

`git:x:1000:1000::/home/git:/usr/bin/git-shell`

Now git user can only use SSH connection to push and get Git repository, and can not directly use the host shell. Try to log on, you will see the following information about this rejection: 

<code>
$ ssh git@gitserver  
fatal: What do you think I am? A shell?  
Connection to gitserver closed.  
</code>  

via: http://www.codeweblog.com/git-server-group-level-structures/

