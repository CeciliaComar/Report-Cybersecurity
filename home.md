# REPORT DEMO CYBERSECURITY: *Vulnhub Napping*
*Student:* **CECILIA COMAR**  
*Student ID number:* **IN2000243**  
*Study program:* **Computer Engineering**


## Credits
I realized my demo using the Napping virtual machine available at [this link](https://www.vulnhub.com/entry/napping-101,752/), in particular following the provided walkthrough.
## Introduction
Napping is a Vulnhub machine created for highlighting the exploit of Tab Nabbing.
Tab Nabbing is a type of phishing attack where a malicious website or script alters the content of a browser tab after 
the user has navigated away from it, typically to another tab or application. The altered content often mimics a 
legitimate website, tricking users into entering their credentials or other sensitive information.
In this report a Tab Nabbing attack will be performed against the Napping machine, to phish some credentials from an administrator.
Then, thanks to the bad practice followed by this user to use the same credential also for SSH, it will be possible to 
enter into the machine using the SSH protocol.
Due to a sudo misconfiguration it will be possible also to perform privilege escalation obtaining root access to the machine.

## Scan of the network
The attack will be performed from the kali Linux machine.  
First of all the command *hostname -I* is performed from the kali Linux machine in order to discover the IP address of the kali machine itself.  
Then the network has to be scanned in order to find other machines in it: this can be done with the command *sudo nmap -v --min-rate 10000 10.0.2.3-254 | grep open*.  
The output of the scan tells that in the network there is a system with IP address *10.0.2.15* which has as open ports:
* port 80 on which is running a http service
* port 22 on which is running an ssh service

This is the targeted vulnerable machine.
A more accurate analysis of the IP address *10.0.2.15* is performed with the command *sudo nmap -v -sV -sC -oN nmap 10.0.2.15 -p-* where:
* thanks to the option *-sV* *nmap* will try to determinate the exact version of the services running on the open ports
* the option *-oN nmap* saves the output of the scanning in a file named nmap and in this way there is a register of the results of the scan.

From the output of this command 2 relevant things can be observed:
1. on the port 22 is running the process OpenSSH
2. on the port 80 is running the process Apache http

Since much can't be done with SSH, a look to the Website is taken.

## Web Page
The web page related to the IP address *10.0.2.15* shows a Login page:
once an account is created (with any username and password), after the login, a free blog promotions site appears.  
The username *test* and  password *password* have been chosen.   
Any link can be here submitted: if as link is inserted the IP address of the machine from which the attacker is working (in my case *10.0.2.7*) a new page is opened at a new tab.  
Analysing the source code of the page, it is showed that this link is vulnerable to Tab Nabbing due to the presence of *target='_blank'*:
![img_3.png](img_3.png)

## Tab Nabbing
A malicious website opened through *target='_blank'* can change the *window.opener.location* to a different page, potentially misleading users. 
Since users usually trust the page that is already opened, they will not get suspicious.  
The idea of the attack is this: since the homepage indicates that the page administrator will review the links, 
I can assume that he will click on the links I submit. 
If I submit a link appropriately constructed, by making the page inactive at the moment it opens the 
new tab, the page inactive will be modified into a page controlled by me that appears identical to the login page: 
when the administrator returns to this page, he will re-enter the credentials without noticing the redirection, 
and thus I will be able to obtain them.  
The next step to perform in order to implement this idea, is writing the exploit.

## Exploit construction
The first thing to do is to create a file coping the source code of the login page.  This file is named *index.html*.  
The second step to perform is to write the payload in a different file:  
![img.png](img.png)

## Phished Credentials
In order to phish some credentials, need to be started:
* a TCP server on port 8000: done with the command *sudo nc -nlvp 8000* 
* an HTTP server on port 80:  once the command *sudo python3 -m http.server 80* has been performed, the HTTP server will serve the files in the 
current directory on port 80 to the users that will connect on this system's port.  

The operations that need to be performed are as follows: 
1. into the page obtained after the login, submit the link pointing to the file indez.html
![img_7.png](img_7.png)  
When the user clicks on this link, he/she will receive the index file from the HTTP server that was started, which will 
redirect the login page to the index.html page that appears identical to the login page
but is actually under the control of the attacker. When the user enters his/her credentials, these will be captured by the TCP server.
2. click on the link: it opens the malicious webpage 
3. after a few moments looking at the output of the *nc* command the username and password of a user
are received:  
![img_8.png](img_8.png)  
This credential can be used for login into the vulnerable machine via ssh.

# Discovery
At this point a good idea is looking around in the system: some useful information that can be obtained is
* groups to which the user daniel belong: with the command *groups* is discovered that daniel is part of the administrators group!
* interesting files at which daniel has access: with the command *find / group -administrators -type f 2>/dev/null*  
where the option *'2>/dev/null'* redirects any error messages to */dev/null*, which is 
a special device in Linux that discards everything sent to it. This prevents 
error messages related to directories that are not accessible from being displayed. With
this command all files owned by the administrators group are being searched starting 
from the root of the file system.  
An interesting file found is the Python script *query.py* which appears to check the status of the 
Web Server and then writing it in the file *site_status*.
By checking the rights of this file with the command *ls -l /home/adrian/query.py* it is discovered
that the user daniel has write permissions on it because he belongs to the administrators group
and by analysing the content of the file *site_status.txt* with the *cat* command it appears that this file is executed every 2 minutes.  

# Privilege Escalation
All this gathered information can be used to perform Privilege Escalation. The idea is the 
one that follows:
1. in the directory */dev/shm* will be created a reverse shell bash script. This directory
is chosen because it is a system directory in Linux that serves as shared memory temporary 
file system: files created in this folder are visible to all system processes, which makes
them useful for temporary data exchange between processes  
![img_14.png](img_14.png)  

2. the query *query.py* is modified to execute this reverse shell:  
![img_1.png](img_1.png)
3. a TCP server needs to be started from kali machine: after a few minutes the reverse shell is obtained
![img_17.png](img_17.png)  

Checking the permissions of the user Adrian, what is discovered is that *Vim* can be executed as root without
needing to know the password (case of violation of the principle of Least Privilege) and according to [Gitfobins](https://gtfobins.github.io/gtfobins/vim/) the root access can be obtained with the command *sudo /usr/bin/vim -c ':!/bin/sh'*:  
![img_2.png](img_2.png)

      



