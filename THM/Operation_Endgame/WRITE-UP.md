## Welcome everyone!

Today, we will explore “_Operation Endgame_”, a hard-rated CTF from TryHackMe.

![image](/THM/Operation_Endgame/OperationEndgame_images/1.png)

## Information Gathering

First, we will perform an aggressive scan with **Nmap** using the `-A` switch to obtain comprehensive results of the system we’re targeting:

![image](/THM/Operation_Endgame/OperationEndgame_images/2.png)

Multiple services have been identified. Fundamentals here are:

**DNS** (Port 53)

**HTTP** (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**RDP** (Port 3389)

It could be deduced from the port scanning that we face an Active Directory environment. The `thm.local` **AD domain** and the `ad.thm.corp`FQDN **Domain Controller** has been identified by Nmap Script Engine’s default scripts (NSE). (we add it in `/etc/hosts`)

Since we start this box without any credentials, we could attempt to list available **SMB shares** through `smbmap` as the unauthenticated *Guest* user:

!3capté.png

The *Guest* user does not have access to any ressource, excepted for the `IPC$` share, which means that it has minimal privileges to make requests within the domain. Moreover, `IPC$` read permission allows us to perform **RID bruteforcing** using ****`nxc` ****through the ****`—rid-brute` option in order to retrieve valid domain users through their SID entry:

![image](/THM/Operation_Endgame/OperationEndgame_images/3.png)

And… the list is long. We could repeat the command, redirecting its output in a local file with `> usersrid.txt` so that we could cut properly usernames from the entire RID line. The following command can be executed to obtain a valid list of domain users for the current penetration test:

`cat usersrid.txt | grep "SidTypeUser" | awk '{print $6}' | cut -d'\' -f2 > users.txt`

![image](/THM/Operation_Endgame/OperationEndgame_images/4.png)

We now have the weapon needed to lead our Active Directory attacks, although no authenticate access is granted yet.

Since the **Kerberos authentication flow** start from an ***AS-REQ*** (Request) made from an authenticated user to the **Authentication service** (AS) in order to recieve a ***Ticket-Granting Ticket*** (TGT) from the ***Key Distribution Center*** (KDC), owned by the Domain Controller (previously identified as `ad.thm.local`). This step is essential for users to access a service legitimately in Kerberos.

However, there is a flag in the ***userAccountControl*** attribute of an Active Directory account. When this bit is set on an account, the KDC skips the **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an ***AS-REP*** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The **Impacket** suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use **impacket-GetNPUsers**. The following command could be executed to query the KDC for domain users:

`impacket-GetNPUsers thm.local/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/THM/Operation_Endgame/OperationEndgame_images/5.png)

It appears that multiple users have the `DONT_REQUIRE_PREAUTH` flag set: AS-REP responses have been recovered from the tool for five different domain users:

- SHELLEY_BEARD
- ISIAH_WALKER
- QUEEN_GARNER
- PHYLLIS_MCCOY
- MAXINE_FREEMAN

We could paste the extracted AS-REP Roast hash derived from users passwords in order to attempt to perform offline cracking on them through **Hashcat**. The hash mode to provide with the `-m` switch could be identified with the `—-identify` option as below:

![image](/THM/Operation_Endgame/OperationEndgame_images/6.png)

Unfortunately, attempting to crack those hashes isn’t concludent, probably due to their passwords strongness. We need to explore more in depth the target machine.

Since we previously identified port 80 serving HTTP as open, let’s attempt to access the website hosted on it:

![image](/THM/Operation_Endgame/OperationEndgame_images/7.png)

Nothing really interesting could be found on the default root Microsoft IIS’s web page. Attempting to perform **directory fuzzing** using **Gobuster** isn’t conclusive.

I then remembered the *Guest* user. We discovered that this domain account has not been deactivated. Moreover, this unprivileged user was still able to query the SMB protocol to list shared ressources. Since this user still represents a valid domain object, we could attempt to use its “null session” authenticated context in order to request a TGT from the KDC, which allows us to request a **Ticket-Granting Service** (TGS) to each user having a **Service Principal Name** (SPN) set, included in our list. This technique is known as **Kerberoasting**. 

Let’s verify if our unprivileged user could authenticate for requesting any TGS to the KDC through the following`impacket-GetUserSPNs` tool command:

`impacket-GetUserSPNs thm.local/Guest -no-pass -userslist users.txt -dc-ip TARET_IP`

![image](/THM/Operation_Endgame/OperationEndgame_images/8.png)

It appears that the Guest user is able to do it! Moreover, the tool successfully identified that a Service Principal Name (SPN) is set for three domain users: `CODY_ROY`, `CHRISTIAN_SANFORD` and `GLENNA_GRAY`.

Hence, TGS hashes derived from users passwords have been recovered by the tool. Since the two user accounts in the domain appear to correspond to “human” user accounts, that may be a misconfiguration, but this need to be confirmed through checking users SPNs—which we may do later.

For now, since we recovered two valid TGS tickets, let’s attempt to crack extracted TGS-REP hashes derived from the users passwords.

(Mode number for etype23 TGS-REP hashes is `13100` in Hashcat, it can also be achieved through **John the Ripper** with **`**—-format=krb5tgs`)

![image](/THM/Operation_Endgame/OperationEndgame_images/9.png)

![image](/THM/Operation_Endgame/OperationEndgame_images/desole.png)

Christina Sanford and Glenna Gray password hashes cracking attempt turned out to be unsuccessful —probably once again, given how strong passwords were. However, Hashcat returned a different result for **Cody Roy**:

![image](/THM/Operation_Endgame/OperationEndgame_images/10.png)

We successfully cracked the weak Cody_Roy user’s password, allowing us to gain authenticated acces within the target AD domain. We could leverage this to use `bloodhound-python` (`SharpHound` tool for Linux) in order to make authenticated LDAP queries to retrieve several domain objects that could be generate and imported as a ZIP file in **`BloodHound`** for ****misconfigured ****ACLs/relations investigation purposes.

(*Guest* user account was unable to do this because it didn't have sufficient LDAP access rights.)

This could be achieved through the following command:

`bloodhound-python -u 'Cody_Roy' -p '<PASSWORD>' -d thm.local -ns TARGET_IP -c All --zip`

![image](/THM/Operation_Endgame/OperationEndgame_images/11.png)

Now, let’s import it in the BloodHound GUI:

![image](/THM/Operation_Endgame/OperationEndgame_images/12.png)

Once file ingest completed, we could inspect a graph representing object-to-object relationship by clicking on *Explore*. Inspecting relations concerning the compromised domain user object, it can be identified that he is **member of the *Remote Desktop Users* domain group**:

![image](/THM/Operation_Endgame/OperationEndgame_images/13.png)

This group ownship enable Cody_Roy to remotely logon on the Windows system through protocols such as **RDP**. We could leverage this exclusive group access to obtain an initial foothold on the remote system. We could use `xfreerdp` to achieve this, with the following command:

`xfreerdp /v:TARGET_IP /u:'Cody_Roy' /p:'<PASSWORD>'`

![image](/THM/Operation_Endgame/OperationEndgame_images/14.png)

Enter “Yes”.

![image](/THM/Operation_Endgame/OperationEndgame_images/15.png)

However, we are greeted with an unreliable access; we couldn’t run the command prompt from the Start Menu. To address this, we could run Win + R to open the Run process window and then running cmd.exe from it to access a shell properly:  

![image](/THM/Operation_Endgame/OperationEndgame_images/cpasmafaute.png)

However, nothing interesting could be identified but an unusual `Scripts` folder in the `C:\` root directory, which is forbidden to access. The only user directory accessible is `C:\Users\Administrator`, which may contain the flag required for completing the room. Accessing the *Documents* icon in the toolbar is also denied, which could mean that this user has not permissions to navigate in the local file system.

Since the user has no other relevant relationships or permissions within the target domain, we need to find another way to obtain valid authenticated access in order to proceed with our full domain compromise.

Since we now have a valid password candidate, we could attempt performing **Password Spraying** which, unlike brute-forcing, consists in attempting the same password against several users in order to leverage a password reuse issue within the domain.

This could be achieved through `nxc` as follows:

`nxc smb 10.130.166.118 -u users.txt -p 'MKO)mko0' --continue-on-succes`

![image](/THM/Operation_Endgame/OperationEndgame_images/16.png)

The `—-continue-on-success` option is required to lead our password spraying attack. This prevents the command from stopping and SMB requests will continue to be sent after the tool has identified a valid authenticated access, such as the irrevocable Cody_Roy user from which the password was extracted.

After several hundred requests, something interesting:

![image](/THM/Operation_Endgame/OperationEndgame_images/17.png)

Password spraying was successful; **Zachary_Hunt** domain user has been identified having the same password as Cody_Roy, allowing us to compromise its user account right away. We could get back in BloodHound to inspect user object nodes:

![image](/THM/Operation_Endgame/OperationEndgame_images/18.png)

One outband object controll has been identified: the **`GenericWrite`** ACE permission is granted over the **Jerri_Lancaster** domain user**.** Since this sensitive right allows the owner to modify non-protected domain object properties, it could allows us to edit victim’s LDAP `msDS-KeyCredentialLink` object properties. 

This could be leveraged to perform targeted Kerberoasting by adding a SPN temporarily in LDAP object properties in order to request a TGS from the victim user account before removing the SPN, we could then crack the TGS-REP hash derived from user’s password offline.

We could execute the following command to properly modify the domain object attribute against Jerri_Lancaster, specified through the `—-request-user` option to specifically query this domain user through LDAP before performing a Kerberos authentication as the Zachary_Hunt user. (Exclusively specifying the victim user through the `—request-user` option is decisive; I was stuck for hours diving in a Shadow Credentials attack rabbit hole, having give up the Targeted Kerberoasting attack since the tool was not able to add the `servicePrincipalName` user’s object propertie through its basic synthax usage.)

`python3 targetedKerberoast.py -u 'Zachary_Hunt' '<PASSWORD>' -d thm.local --dc-ip TARGET_IP --only-abuse --request-user JERRI_LANCASTER`

![image](/THM/Operation_Endgame/OperationEndgame_images/20.png)

Our attack was successful; a SPN has successfully been added to the victim user, allowing us to authenticate with it through Kerberos for looting it’s TGS-REP hash that we could attempt to crack in Hashcat with -m 13100 (specifying e-type 23 TGS-REP HASH format), as we previously did with **Cody_Roy**:

![image](/THM/Operation_Endgame/OperationEndgame_images/21.png)

Weak user password could be cracked, enabling us to obtain a new authenticate access on the domain. Let’s get back in BloodHound:

Inspecting Jerri_Lancaster objects properties, it can be observed that the Admin Count attribute has its value set to `True` (represented by a **1** in LDAP).

![image](/THM/Operation_Endgame/OperationEndgame_images/bonjmefousdvotregueulemdr.png)

By observing the Outband Object Control tab, we can confirm the user is a member of the intriguing ***Reader Admins*** group.

![image](/THM/Operation_Endgame/OperationEndgame_images/22.png)

Moreover, an interesting **`WriteOwner`** access right has been configured for the compromised Jerri_Lancaster domain user over the newly discovered group. This sensitive permission allows a user to modify the group’s owner, which could be used to take ownership of the targeted object.

To change the ownership of the object, we may use Impacket's `owneredit` tool to access and modify domain object attribute through **LDAP** :

![image](/THM/Operation_Endgame/OperationEndgame_images/23.png)

We can observe that the previous group owner **Christian_Sandford** privileged user (**Tier Zero**) has been replaced by our compromised user.

This can be verified by using the tool again, by ordering the `read` action against the domain group as below:

![image](/THM/Operation_Endgame/OperationEndgame_images/24.png)

The Jerri_Lancaster now owns the *Reader Admins* group. Since i tried to search about this group name in common AD conventions, but I found nothing, it appears to be an “Admin group” customized group for the box purposes.

So far, wee identified RDP as accessible and we accessed it as an unprivileged user.

The ACL Analysis performed against the Jerri_Lancaster user clearly suggests that he is a privileged user of some sort, even though he is not a domain administrator.

We could attempt to access the RDP session associated with this user in order to see whether something new could be achieved.

Inspecting the `C:\` root directory, an unusual `C:\Scripts` folder that was forbidden to access as Cody_Roy could be identified. Inspecting folder permissions through `icacls` reveals something very interesting:

![image](/THM/Operation_Endgame/OperationEndgame_images/25.png)

It appears that the folder is only accessible by domain administrator, but additionally, Jerri Lancaster has read and execute permissions on the it! This allows us to access the folder and identify a mysterious `syncer.ps1` PowerShell script. Opening it in the Notepad reveals the following content:

![image](/THM/Operation_Endgame/OperationEndgame_images/26.png)

The file contains hardcoded credentials concerning the **Sanford_Daugherty** domain user. Let’s verify if that is legitimate with `nxc` before celebrating to soon:

![image](/THM/Operation_Endgame/OperationEndgame_images/27.png)

Indeed, recovered credentials are valid within the domain! Let’s inspect this domain user in BloodHound:

![image](/THM/Operation_Endgame/OperationEndgame_images/28.png)

And… here we are!

We can observe that Sanford Daugherty belongs to **Tier Zero**, which represents the **highest access level** within an Active Directory domain. This strongly suggests that this user is a domain administrator belonging to the **`*Domain Admins*`** group.

By default, domain administrator users have the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` permissions for data replication purposes. We could leverage these excessive privileges to perform a **DCSync attack** in order to dump **`NTDS.dit`** secrets, including all the password hashes for users in the domain.

This highly-privileged attack consists in impersonating DC’s role in a way to replicate sensitive data through the “***DRSUAPI***” ****RPC interface**. Indeed, Active Directory domains need **data replication** between Domain Controllers in order to synchronize everything properly within the AD domain. It is a legitimate feature in and of itself, and it also requires significant privileges. But since we just compromised the Domain Admin user, all privileges are already granted.  

Please note this method is achieved through `-just-dc` because we are querying data replication to the target Domain Controller. Without it, the tool access local Windows registry hives through the `WindowsRegistry` service in order to retrieve hashes stored in the Security Account Manager (SAM) by decrypting it with the local **system bootKey** extracted from the `SYSTEM` hive, without any DRSUAPI method deployed. 

(the `impacket-secretsdump` tool is used here to lead **DCSync** attack with the `*domain/user:password@TARGET_IP*` synthax paired with the `-just-dc` option allowing to perform the attack)

![image](/THM/Operation_Endgame/OperationEndgame_images/30.png)

We successfully dumped all **`NTDS.dit`** secrets, allowing us to retrieve the **`Administrator`** password NTLM hash.

We could now leverage this privileged access to abuse the *Windows Task Scheduler* in order to perform remote code execution under the local **`NT AUTHORITY\SYSTEM`**  account through a **Pass-the-Hash** attack, without needing to crack user’s password, regardless of its strength.

This could be achieved with the `impacket-atexec` tool as below:

![image](/THM/Operation_Endgame/OperationEndgame_images/31.png)

Privilege esclation has been confirmed, which conclude in a full domain compromise.

The room flag can be accessed in the `C:\Users\Administrator\Desktop` directory.

![image](/THM/Operation_Endgame/OperationEndgame_images/32.png)

Challenge completed!

!33.![image](/THM/Operation_Endgame/OperationEndgame_images/33.png)



## Conclusion

This room was very pleasant to complete. Starting without any credentials, we compromised a first domain user through a **Kerberoast** attack as the unprivileged *Guest* user, allowing us to obtain a TGS from the KDC. We cracked the TGS-REP password hash due to insufficient password strength, which allowed us to gain authenticated access within the domain. We also learnt from a **password spray** performed ****against all domain users that the Jerri_Lancaster user has the same password, representing again, a critical failure to comply with safety standards.

We leveraged a RDP access as Jerri Lancaster, which was able to access a restricted `C:\Scripts` folder, which led us to discover aPowerShell script that stored insecurely hardcoded credentials associated with the Domain Administrator & Domain Admin group highly-privileged member `Sanford_Daugherty`, which leads us to conclude that a very poor security practice regarding these employees is ubiquitous.

This challenge perfectly demonstrates how multiple low-severity vulnerabilities could be chained to achieve a complete domain compromise.

**Thank you for reading!!**



## BONUS


So far you're still here?? Do you really want to read about a rabbit hole I fell down while enumerating Zachary_Hunt vulnerable DACL? Alright!
Nevertheless, this provides some very interesting insights into **PKINIT**, the Kerberos extension that enables **asymmetric-key cryptographic authentication**.

In Active Directory, the **Key Trust** mechanism could allows a domain user to authenticate with a **certificate**. A `KeyCredential` structure can be stored in the `msDS-KeyCredentialLink` LDAP attribute in order to associate a public key with a user. An attacker could leverage the `GenericWrite` ACE to make these changes in order to leverage **asymmetric cryptography-based** authentication as the Jerri_Lancaster domain user (through the **PKINIT** Kerberos extension).

This attack can be carried out with the `pywhisker.py` script tool:

(Python script dependancies could be downloaded with the `pip3 -m requirements.txt` command once the repository has been downloaded using the `git clone` command.)

`pywhisker.py -d 'thm.local' -u 'Zachary_Hunt' -p '<PASSWORD>' --target 'Jerri_Lancaster' --action 'add' --filename privatesecret`

!33.![image](/THM/Operation_Endgame/OperationEndgame_images/19.png)

We successfully received a `.pfx` **PKCS#12 cryptographic container**, which stores our cryptographic secret and the auto-signed certificate. It need to be provided paired with the generated password that protect the file.

We could run the following commands to extract separately the private key and the certificate for a better authentication reliability (it helps to avoid the `KDC_ERR_PADATA_TYPE_NOSUPP` unsupported method Kerberos error)

`openssl pkcs12 -in secret.pfx -clcerts -nokeys -out cert.pem`
`openssl pkcs12 -in secret.pfx -nocerts -nodes -out key.pem`
(These commands will prompt for the PKCS#12 container password)

Once both file generated, we could execute the following command to authenticate with the Domain Controller in order to obtain a valid TGT that would allows us to impersonate Jerri_Lancaster user account within the domain:

`python3 PKINITtools/gettgtpkinit.py -cert-pem cert.pem -key-pem key.pem thm.local/Jerri_Lancaster Jerri_Lancaster.ccache`

However, for some reason, this method wont work…

*the write-up continues from there*


**Thank you for reading again<3**
