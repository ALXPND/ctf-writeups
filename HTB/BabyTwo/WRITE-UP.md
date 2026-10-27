## Welcome everyone!

Today, we will explore “*BabyTwo*”, a medium-rated machine from **Hack The Box**.

Without further ado, let’s get started!

![image](/HTB/BabyTwo/BabyTwo_images/1.png)

## Information Gathering

 

First, we will perform a full TCP SYN port scan to enumerate all possible accessible gates using **Nmap before** running an aggressive scan on open ports through the `-A` switch to gather as much information as possible:

![image](/HTB/BabyTwo/BabyTwo_images/2.png)

Multiple services are running. Fundamentals here are:

**DNS** (Port 53)

HTTP (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**HTTPS** (Port 443)

**SMB** (Port 445)

**RDP** (Port 3389)

It has been deduced from the retrieved information that we’re facing an Active Directory environment. The `baby2.vl` domain and the `dc.baby2.vl` DC’s FDQN has been identified, which means that our assessment is made against a Domain Controller that exposed several services publicly that we could leverage for harvesting.

(These informations were added in `/etc/hosts`)

Investigating SMB shares permissions as an unauthenticated user (Guest/Null session) with `smbmap` revealed an uncommon `homes` public share allowing write permissions:

![image](/HTB/BabyTwo/BabyTwo_images/3.png)

Inspecting the `apps` share does not reveals anything significant at this stage.

Inspecting the share folders bearing the name of their employee. This has been leveraged to construct a valid users list. 

![image](/HTB/BabyTwo/BabyTwo_images/4.png)

Since we had read access on the `IPC$` share, RID brute-forcing was possible to to potentially retrieve other users. `DC$` machine account and `gpoadm` user account was identified (added in our list). 

![image](/HTB/BabyTwo/BabyTwo_images/5.png)

Since the **Kerberos authentication flow** start from an ***AS-REQ*** (Request) made from an authenticated user to the **Authentication service** (AS) in order to recieve a ***Ticket-Granting Ticket*** (TGT) from the ***Key Distribution Center*** (KDC), owned by the Domain Controller (previously identified as `ad.thm.local`). This step is essential for users to access a service legitimately in Kerberos.

However, there is a flag in the ***userAccountControl*** attribute of an Active Directory account. When this bit is set on an account, the KDC skips the **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an ***AS-REP*** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The **Impacket** suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use **impacket-GetNPUsers**. The following command could be executed to query the KDC for domain users:

`impacket-GetNPUsers thm.local/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/HTB/BabyTwo/BabyTwo_images/6.png)

It appears that no `DONT_REQUIRE_PREAUTH` flag has been detected within domain users configuration.

However, two users insecurely configured **their own username** as their password within the Active Directory domain. This has been demonstrated through NetExec as below:

![image](/HTB/BabyTwo/BabyTwo_images/7.png)

`Carl.Moore` and `library` domain users account session have been compromised at this stage. This allows an attacker to visualize domain object nodes in BloodHound with a tool such as `bloodhound-python`.

```jsx
bloodhound-python -u 'Carl.Moore' -p 'Carl.Moore' -d baby2.vl -ns 10.129.234.72 -c All --zip

```

![image](/HTB/BabyTwo/BabyTwo_images/8.png)

![image](/HTB/BabyTwo/BabyTwo_images/9.png)

![image](/HTB/BabyTwo/BabyTwo_images/10.png)

Inspecting library object relations doesn’t reveals anything anormal. However, it has been observed that the Carl_Moore user is a member of the ***Office*** group, it has been suggested that the user could have more privileges granted. Hence, SMB shares enumeration has been resumed.

![image](/HTB/BabyTwo/BabyTwo_images/11.png)

Both users have write access on the `apps` and `docs` share. We also gained read access on the `SYSVOL` share through their access. Inspecting it revealed in the *scripts* folder a `login.vbs` VBS login script file which appears to be processed by a scheduled task. Since the msfvenom tool is able to craft payload in VBS format, it has been attempted to hijack the legitimate binary through replacing it (a copy has been saved in local). This resulted in a successfull payload execution, granting a shell as the `BABY2\Amelia.Griffiths` domain user on the system.

![image](/HTB/BabyTwo/BabyTwo_images/12.png)


The first flag evidence could be retrieved and accessed in the `C:\` root filesystem’s directory.

![image](/HTB/BabyTwo/BabyTwo_images/13.png)

Getting back in BloodHound to inspect Amelia.Griffiths object relations, it has been revealed that the user is a member of the ***Legacy*** group. Moreover, an interesting object nodes could have been identified; *Legacy* group members can manage the “`gpoadm`” domain object as they want. (GPO administration account we previously discovered from RID brute-forcing). 

![image](/HTB/BabyTwo/BabyTwo_images/14.png)

This could be done because sensitive**`WriteDACL`** and WriteOwner ACE permission have been configured for these group members. 

Each of these two ACEs can be used to make changes to the DACL of the target object.
**`WriteOwner`** allows the inheriting user to take ownership of the target object and grant themselves the <u>**`GenericAll`**</u> ACE, which represents **Full Control** rights. The `impacket-dacledit` tool from the Impacket suit could be used to modify DACL, but since we don’t have any password concerning the Amelia.Griffiths user, it have been done locally on the Windows system, through **PowerView,** a Powershell script extented tool.

![image](/HTB/BabyTwo/BabyTwo_images/15.png)

Since no error has been returned by PowerShell, DACL modification has been successful. Since Full Controll rights have been attributed over the gpoadm object, a password reset could now be achieved in order to hijack the account session.

![image](/HTB/BabyTwo/BabyTwo_images/16.png)

As expected, password change was successful. The domain account responsible for GPOs management have been compromised, which means that `Default Domain Policies` and Default `Domain Controller Policies` Group Policies Objects could be abused in order to grant to ourselves sensitive privileges within the AD domain. This can be confirmed by inspecting the Outband Object Control tab in BloodHound concerning the gpoadm domain object.

![image](/HTB/BabyTwo/BabyTwo_images/17.png)

The GenericAll ACE permission grants Full Control over GPO objects, which represents a critical level of compromise. An attacker could leverage a script such as pygpoabuse.py that will abuse write permissions granted over any vulnerable GPOs in order to configure a scheduled task processed by the **`NT AUTHORITY\System`** privileged account that will create and add a `john` user in the `BUILTIN\Administrators` local group of the machine. Given that this is done on the controller domain, this privileged entity automatically adds the user to the **Tier Zero** group "**`*Administrators*`**" which represents a significant security risk.

![image](/HTB/BabyTwo/BabyTwo_images/18.png)

![image](/HTB/BabyTwo/BabyTwo_images/19.png)

The newly generated scheduled task could then be retrieved from the `SYSVOL` SMB share, In the `ScheduledTask` folder, valid credentials could be retrieved for authenticating as `john` --the newly created domain administrator account--  with the DC .

![image](/HTB/BabyTwo/BabyTwo_images/20.png)

![image](/HTB/BabyTwo/BabyTwo_images/22.png)

It has been confirmed through `ldapsearch` that the fake user has been successfully added in the ***Administrators*** group:

![image](/HTB/BabyTwo/BabyTwo_images/21.png)

We could now simply authenticate, or request a TGT to authenticate through Kerberos with the Domain Controller in order to perform **DCSync** with it.

By default, domain administrator users have the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` permissions for data replication purposes. We could leverage these excessive privileges to perform a **DCSync attack** in order to dump **`NTDS.dit`** secrets, including all the password hashes for users in the domain.

This highly-privileged attack consists in impersonating DC’s role in a way to replicate sensitive data through the “***DRSUAPI***” ****RPC interface**. Indeed, Active Directory domains need **data replication** between Domain Controllers in order to synchronize everything properly within the AD domain. It is a legitimate feature in and of itself, and it also requires significant privileges. But since we just compromised the Domain Admin user, all privileges are already granted.  

Please note this method is achieved through `-just-dc` because we are querying data replication to the target Domain Controller. Without it, the tool access local Windows registry hives through the `WindowsRegistry` service in order to retrieve hashes stored in the Security Account Manager (SAM) by decrypting it with the local **system bootKey** extracted from the `SYSTEM` hive, without any DRSUAPI method deployed to retrieve a copy of all domain secrets stored in `NTDS.dit`.

![image](/HTB/BabyTwo/BabyTwo_images/23.png)

![image](/HTB/BabyTwo/BabyTwo_images/24.png)

As expected, NTDS.dit secrets have been replicated from the Domain Controller, allowing us to retrieve the NTLM hash of the local **`Administrator`** privileged account, on the `DC$` host (Domain Controller), resulting in a full compromise of the `baby2.vl` Active Directory domain.

Since WinRM is not enabled, Evil-WinRM cannot be used for Pass-the-Hash. However, PtH technique is still possible with Impacket's `psexec.py` or `wmiexec.py`. `psexec.py` typically provides a shell running as `NT AUTHORITY\SYSTEM`, whereas `wmiexec.py` executes commands in the context of the authenticated user, allowing us to confirm our privileges with `whoami`.

![image](/HTB/BabyTwo/BabyTwo_images/25.png)

We successfully escalated our privileges to the privileged `Administrator` local user on the Domain Controler, which conclude in a full system compromise

This can be confirmed with the `whoami /priv` and the `hostname` command

![image](/HTB/BabyTwo/BabyTwo_images/26.png)

The root flag could have been retrieved in the `C:\Users\Administrator\Desktop` directory.

![image](/HTB/BabyTwo/BabyTwo_images/27.png)

Challenge completed!

![image](/HTB/BabyTwo/BabyTwo_images/28.png)

## Conclusion

BabyTwo demonstrated how misconfigurations in SMB permissions, weak passwords, and excessive Active Directory privileges can be chained to compromise an entire domain. By abusing writable shares, a vulnerable login script, excessive ACL permissions, and GPO control, we obtained Domain Admin-level access and ultimately compromised the Domain Controller. This challenge highlights the importance of enforcing least privilege, securing credentials, and auditing SMB shares, AD permissions, and Group Policy Objects.

**Thank you for reading!**
