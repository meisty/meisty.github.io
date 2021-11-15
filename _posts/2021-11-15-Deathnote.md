---
layout: post
title: Deathnote Vulnhub VM Walkthrough
---

## A walkthrough of the [Deathnote](https://www.vulnhub.com/entry/deathnote-1,739/) VM from Vulnhub

This is labelled as an easy box on vulnhub.  

Initially I ran 

[<img src="../images/deathnnote/nmap.png"
  style="width: 800px;"/>](../images/deathnote/nmap.png)

There was port 22 SSH and port 80 HTTP open.  I took a look at the website hosted on port 80 and added the domain name it mentioned to my /etc/hosts file deathnote.vuln.  

[<img src="../images/deathnote/home.png"
  style="width: 800px;"/>](../images/deathnote/home.png)
  
First thing I noticed was that it was a wordpress site (noticed from the url and also from the layout) and that there was a potential username mentioned from a post `Kira`. So I headed over to the wp-admin page and tried to login with the default credentials admin:password. 

[<img src="../images/deathnote/wp-admin.png"
  style="width: 800px;"/>](../images/deathnote/wp-admin.png)
  
No luck.  Then I attempted to use kira and got a incorrect password response.  After taking a look at the homepage again I saw a line of text "my fav line is iamjustic3" posted by the user `L`.

[<img src="../images/deathnote/fav_word.png"
  style="width: 800px;"/>](../images/deathnote/fav_word.png)
  
I tried logging into wp-admin with kira:iamjustic3 , success!

[<img src="../images/deathnote/wp.png"
  style="width: 800px;"/>](../images/deathnote/wp.png)
  
I had a look in the media library section and noticed a file, notes.txt.  I downloaded the file and took a look at it. 

[<img src="../images/deathnote/notes.png"
  style="width: 800px;"/>](../images/deathnote/notes.png)

Looks like a potential list of passwords.  So I fired up Hydra and pointed it to port 22 SSH and tried with the username kira.  No luck.  But then I remembered the user `L` and tried again.  Success.

[<img src="../images/deathnote/hydra.png"
  style="width: 800px;"/>](../images/deathnote/hydra.png)

I found user.txt in l's home directory.

[<img src="../images/deathnote/user.png"
  style="width: 800px;"/>](../images/deathnote/user.png)
  
But l is not able to run sudo.  So I did some poking around and found something interesting in /opt/L/fake-notebook-rule

[<img src="../images/deathnote/sudo.png"
  style="width: 800px;"/>](../images/deathnote/sudo.png)
  
[<img src="../images/deathnote/wav.png"
  style="width: 800px;"/>](../images/deathnote/wav.png)
  
From the hint I could see I needed to decode the value in case.wav using cyberchef.  I decoded the first hex value and then decoded the base64 value I got to reveal kiras password.

[<img src="../images/deathnote/cyberchef.png"
  style="width: 800px;"/>](../images/deathnote/cyberchef.png)
  
After logging into ssh with kira I checked what kira could run as sudo, all :) 

[<img src="../images/deathnote/kira-sudo.png"
  style="width: 800px;"/>](../images/deathnote/kira-sudo.png)
  
I then ran /bin/bash with sudo to get a root shell and then I could read the root flag.  
  
[<img src="../images/deathnote/priv_esc.png"
  style="width: 800px;"/>](../images/deathnote/priv_esc.png)

[<img src="../images/deathnote/root.png"
  style="width: 800px;"/>](../images/deathnote/root.png)

This was a fun machine which just required you to follow the clues. 
