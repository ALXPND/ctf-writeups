## Welcome everyone!

Today, we will explore “Sauna”, an easy-rated machine from **Hack the Box**.

Without further ado, let’s get started!

![image](/HTB/Sauna/Sauna_images/1.png)

(Please ensure that you have added the following line:   `TARGET_IP     sauna.htb EGOISTICAL-BANK.LOCAL`  to your local `/etc/hosts` file to enable **DNS resolution** of the target)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Sauna/Sauna_images/2.png)

We can identify several services running on the target machine. Fundamentals here are:

**DNS** (port 53)

**HTTP** (port 80)

**RPC** (port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

kpasswd (Kerberos auth, Port 464)

**WinRM** (Port 5985)

Moreover, the **Nmap Script Engine** allowed us to identify the target domain name: `EGOTISTICAL-BANK.LOCAL`.

It can be deduced from the port discovery phase that we’re facing a **Windows environment**, deploying **Active Directory**, suggesting a wide surface for enumeration along these discovered services.

Let’s first inspect the **website** hosted on port **80**:

![image](/HTB/Sauna/Sauna_images/3.png)

We are greeted with a static HTML web page containing multiple external links. However, it may contains hidden web resources. So we could perform some web fuzzing using a tool such as **Gobuster**

![image](/HTB/Sauna/Sauna_images/4.png)

Several HTML files are identified. These files could contain sensitive information such as employee names, allowing us to enumerate the Active Directory surface before attempting to exploit it. We could attempt to access the `about.html` file from our web browser to see what it could offer:

![image](/HTB/Sauna/Sauna_images/5.png)

A standard static HTML page is presented. However, we can identify multiple employees name at the very bottom of the page:

![image](/HTB/Sauna/Sauna_images/6.png)

This piece of information could be useful: we could attempt to target these user accounts to gain our initial access over the target domain.

## Initial Access

I created a `users.txt` file containing employees name in multiple forms:

![image](/HTB/Sauna/Sauna_images/7.png)

We could use a tool from the famous Impacket pentesting: `GetNPUsers.py`. This could allow us to see whether any user account that has the ‘`Does not require Pre Authentication`’ privilege enable. Hence, let’s attempt to  query **ASReproastable** accounts from the Key Distribution Center (KDC) by using the following command:

`spookysec.localpython /opt/impacket/examples/GetNPUsers.py / -no-pass -usersfile users.txt -dc-ip 10.129.56.115`

![image](/HTB/Sauna/Sauna_images/8.png)

Our ASReproasting attack has been successful: it appears that the `fsmith` AD user has ‘`Does not require Pre Authentication`’ privilege enabled, allowing us to query an AS-REP Roast hash, corresponding to the user’s password. If Fergus Smith uses a weak password, it could be relatively easy to recover it in plaintext by cracking it offline using a tool such as **John the Ripper** or **Hashcat**.

To know the hash mode to provide, we will paste the whole line into an `hash.txt` file that we will provide to hashcat with the `—-identify` option:

![image](/HTB/Sauna/Sauna_images/9.png)

Now that we have identified the hash mode (which is **Kerberos 5, etype23, AS-REP**) to provide, let’s attempt to crack the Fergus’s password with the following command:

`hashcat -m 18200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Sauna/Sauna_images/10.png)

because I got a few issues with my kali machine, i did it with **John the Ripper**:

![image](/HTB/Sauna/Sauna_images/11.png)

Fergus’s password has successfully been recovered in plaintext: `Thestrokes23`.

Since we now have valid credentials (`fsmith:Thestrokes23`), we could attempt to authenticate ourselves. But where?

Indeed, our previous Nmap scan allowed us to identify that WinRM was running on the target. Since this service offers a way to gain a shell access with valid credentials (such as with **SSH**), we could use evil-winrm to authenticate through the following command:

`evil-winrm -H sauna.htb -U fsmith -P Thestrokes23`

![image](/HTB/Sauna/Sauna_images/12.png)

We successfully obtained our initial foothold on the target system as the `fsmith` user!

The first flag can now be accessed in the `C:\Users\FSmith\Desktop\` directory:

![image](/HTB/Sauna/Sauna_images/13.png)



## Privilege Escalation

Now we have an initial physical access on the target machine, let’s enumerate our current **privileges** using the `whoami /priv` command:

![image](/HTB/Sauna/Sauna_images/14.png)

It appears that we need to escalate our current privileges in order to retrieve the protected root flag.

After leading a local system enumeration, an insecure configuration can be identified when we read the **WinLogon registry key:**

![image](/HTB/Sauna/Sauna_images/15.png)

A bad auto-logon configuration allows us to retrieve the password associated to the `svc_loanmanager` service account in plaintext: `Moneymakestheworldgoround!`.

Since this represents the name of the service account along the domain, we will run the `net user` command to see which username is associated to this account:

![image](/HTB/Sauna/Sauna_images/16.png)

The discovered account is associated to the `svc_loanmgr` user on the remote system. We can attempt to pivot our access to this particular service account to see whether some privileges can be gained in order to ultimately get `SYSTEM` access over the machine.

Let’s use evil-winrm again to authenticate ourselves with the following credentials: `svc_loanmgr:Moneymakestheworldgoround!`

![image](/HTB/Sauna/Sauna_images/17.png)

It works! We successfully pivoted our access from F**Smith** to **svc_loanmgr**. But it appears that it doesn’t offer more privileges:

![image](/HTB/Sauna/Sauna_images/18.png)

We may need to use **BloodHound** in order to enumerate permissions that the user we just found herits within the domain. These permissions could be critical from a security perspective.

Since we obtained valid credentials for the **svc_loanmanager** domain user, we could exfiltrate useful information about relationships and permissions within the AD domain.

This can be done with **SharpHound**, ****a powerful tool written in **PowerShell** that can be used directly on the target machine. After transferring the `Sharphound.exe` binary to the target, we could use the following command  to create a **ZIP** file that we could import in BloodHound in order to observe interesting data:

`.\SharpHound.exe --CollectionMethods All --Domain EGOTISTICAL-BANK.LOCAL`

![image](/HTB/Sauna/Sauna_images/19.png)

![image](/HTB/Sauna/Sauna_images/20.png)

The file has successfully been generated, it can be transfered to our local machine in order to load it into BloodHound and process our investigation. Thanks to *Impacket* again, the following command can be used to transfer the file through **SMB**:

`impacket-smbserver share <LOCAL_FOLDER> -smb2support`

![image](/HTB/Sauna/Sauna_images/21.png)

![image](/HTB/Sauna/Sauna_images/22.png)

Once the file ingestion completed, we could click on Explore and select the user we compromised to analyze what is related about it:

![image](/HTB/Sauna/Sauna_images/23.png)

Inspecting the relationship between the svc_loanmgr user and the domain, we can observe an intriguing permission:

![image](/HTB/Sauna/Sauna_images/24.png)

This critical misconfiguration is a big issue from a security perspective. These permissions allow us to perform a **DCSync** attack ;  The `svc_loanmgr` account has both the `GetChanges` and `GetChangesAll` replication permissions over the domain. These permissions are normally used by Domain Controllers to replicate Active Directory data.

However, when assigned to an unintended account, they can be abused to perform a **DCSync attack**. DCSync abuses the Active Directory replication protocol to request credential material from a Domain Controller through **DRSUAPI**, without directly accessing the `NTDS.dit` file.

We can leverage these permissions with `impacket-secretsdump` to retrieve domain credential hashes.Now that we know **svc_loanmgr** has critical permissions, we could use `impacket-secretsdump` to leverage these permissions in order to drop users password hashes:

![image](/HTB/Sauna/Sauna_images/25.png)

It works! We successfully retrieved domain credential hashes through the Active Directory replication protocol (DRSUAPI), including **Administrator's NTLM hash**. At this point, we could paste the NT hash to a file in order to crack it offline, but we are not warranty that the password is weak. However `evil-winrm` allows to perform a **Pass-the-Hash** attack with the `-H` switch, so let’s do this:

(Remember the NTLM hash structure: `USER:RID:LM_HASH:NT_HASH:::`; NT HASH only is required)

![image](/HTB/Sauna/Sauna_images/26.png)

We successfully escalated our privileges to **Administrator**, granting full access and controll over the target system. The final flag can now be accessed in the `C:\Users\Administrator\Desktop\` directory.

![image](/HTB/Sauna/Sauna_images/27.png)


Challenge completed!

![image](/HTB/Sauna/Sauna_images/28.png)



## Conclusion

This challenge was very pleasant to complete. From a basic web enumeration, we identified an Active Directory environment containing multiple employees belonging to a company. An insecure misconfiguration was detected concerning the **j.smith** user: he was configured not to require Kerberos preauthentication allowing us request the AS-REP response and obtain a crackable AS-REP response derived from the user's password. This response can be cracked offline to recover the user's plaintext password. This misconfiguration led us to obtain an initial foothold on the target machine, allowing us to perform a local enumeration of the system. We learnt that a svc_loanmgr user was configured to autologin, this allowed to recover the password in plaintext in order to pivot our initial access. Then, we discovered that this service account had a critical permissions over the domain, enabling data replications. This allowed us to query database secrets to the DC, leading to a Administrator’s NT hash recovery. Finally, we performed a Pass-the-Hash attack to obtain an elevated access as the Administrator user, concluding in a full compromise of the target machine.

This box demonstrates how low-severity misconfigurations chained can lead to a total compromise of a system.

Thank you for reading!
