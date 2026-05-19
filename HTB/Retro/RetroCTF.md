HackTheBox - Retro (Walkthrough)

## Welcome everyone!

Today, we will explore the “Retro” machine from HackTheBox, an easy-level box.

Let’s get started!

![image](/HTB/Retro/Retro_images/1.png)

(ensure that you added the following line `TARGET_IP      retro.htb` to your `/etc/hosts` local file)

## Information Gathering

First, we will run **Nmap** to perform an aggressive scan to obtain comprehensive results:

![image](/HTB/Retro/Retro_images/2.png)

The scan revealed multiple running services, but the fundamentals here are:

**DNS** (Port 53)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**Kerberos** Password (Port 464)

**RDP** (Port 3389)

We can deduce from the `DC.retro.vl` domain name and these technologies that we are facing an Active Directory environment.

We can start our enumeration through the usage of `smbclient` in order to enumerate SMB shares:


![image](/HTB/Retro/Retro_images/3.png)



We can see multiple shares available on the target. After testing to access some of them anonymously, we find out that we are able to access and list the `Trainees` share which contains an interesting file:


![image](/HTB/Retro/Retro_images/4.png)




We can download the file via `get` and read its content from our machine:


![image](/HTB/Retro/Retro_images/5.png)




We can learn from there that various users share the same account.

We can use `NetExec` to perform RID brute forcing to enumerate users, group and service accounts as the **Guest** user with the following command:

`nxc smb retro.htb -u guest -p '' --rid-brute`



![image](/HTB/Retro/Retro_images/6.png)




We can identify multiple usersaccounts that we can add into a `users.txt` file:



![image](/HTB/Retro/Retro_images/7.png)




We can suppose that the mentioned shared account is the `trainee` account. As it was explained, the purpose was to facilitate the users to remember their password. After trying a couple of combinations, we can find that the password for the **trainee** user account is simply `trainee`. Getting back to the discovered SMB shares, it seems that we can now access the `Notes` share through the **trainee** user, listing its content reveals interesting files, including the first flag:



![image](/HTB/Retro/Retro_images/8.png)




We can download the ToDo.txt file and see what it contains:



![image](/HTB/Retro/Retro_images/9.png)





We can learn that an old computer is still running on the target, which may be associated with a user account. We can deduce from this file that James refers here to the `BANKING$` account we previously added into `users.txt`. After doing some research about  **pre created machine account**, we can learn that these types of account have their computer name in lowercase as their default password in lowercase (without the $ symbol)

Again, we can attempt to access some shares as this particular user:



![image](/HTB/Retro/Retro_images/10.png)





This error typically means we have guessed the correct password for the computer account, but the system prevents login due to trust limitations (as expected for machine accounts).

After further enumeration, we can find that the target has a vulnerable template which can be exploited by requesting certificates impersonating other users, which can lead us to privilege escalation. 

We can use `certipy-ad`, a very powerful tool used to enumerate certificate-related information.

By extracting a TGT (Kerberos ticket) for the **BANKING$** computer account, we can use it to enumerate certificate services and identify any misconfigurations we can exploit:




![image](/HTB/Retro/Retro_images/11.png)



![image](/HTB/Retro/Retro_images/12.png)




The tool successfully retrieved a lot of information, including the CA and the vulnerable template!

Since we have stored the Kerberos ticket for the **BANKING$** account, we can leverage it in order to request certificates for other users. For example, we can attempt to target the **Administrator** account with the following command:

`certipy-ad req -u 'BANKING$@retro.vl' -k -dc-ip 10.129.38.164 -dc-host DC.retro.vl -ca 'retro-DC-CA' -template RetroClients -upn administrator@retro.vl -dns administrator.retro.vl -key-size 4096`



![image](/HTB/Retro/Retro_images/13.png)



We successfully retrieved a certificate for the Administrator account. We can use it to authenticate via PKINIT and recover the NTLM hash, allowing us to perform a **Pass-the-Hash attack**.



![image](/HTB/Retro/Retro_images/14.png)




We get a SID mismatch. The certificate does not contain the correct object SID extension required for strong certificate mapping, which causes authentication to fail. We can retrieve it through the use of `lookupsid.py` script from the [Impackets](https://github.com/fortra/impacket) suite:


![image](/HTB/Retro/Retro_images/15.png)





We can now run again `certipy-ad` in order to generate a valid certificate providing now the SID via `-sid`:

(we must to provide the complete SID, including the RID related to the account. In Active Directory, the built-in Administrator account always uses the Relative Identifier (RID) 500.)



![image](/HTB/Retro/Retro_images/16.png)




(You may need to run the command multiple times if it doesn’t work directly)

We can now use the certificate to authenticate as Administrator in order to retrieve his NTLM hash:




![image](/HTB/Retro/Retro_images/17.png)





Great, we can now use it through a Pass-the-Hash attack to authenticate ourselves via `evil-winrm`:

(Note that we will NOT take the LM hash)




![image](/HTB/Retro/Retro_images/18.png)




And we successfully gain access to the target as Administrator, which leads us to a total compromise of the system!

We can now navigate in the `C:\Users\Administrator\Desktop` directory and fetch the remaining `root.txt` flag:




![image](/HTB/Retro/Retro_images/19.png)




The machine is now fully **compromised**.




![image](/HTB/Retro/Retro_images/20.png)




## Conclusion

Retro was an excellent introduction to Active Directory Certificate Services (ADCS) abuse and ESC1 exploitation.

This machine demonstrated how small misconfigurations, such as weak account management practices and vulnerable certificate templates, can lead to a complete domain compromise.

Throughout this box, we covered:

- SMB enumeration
- RID brute forcing
- Machine account abuse
- Kerberos authentication
- ADCS enumeration with Certipy
- ESC1 certificate template exploitation
- PKINIT authentication
- Pass-the-Hash attacks

Overall, Retro is a great machine for learning modern Active Directory privilege escalation techniques involving certificates and misconfigured enrollment templates.
