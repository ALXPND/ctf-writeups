## Welcome everyone!

Today, we will explore “*Sendai*”, a medium-rated machine from Hack The Box.


![image](/HTB/Sendai/Sendai_images/0.png)


## Information Gathering

 

First, we will perform a full TCP SYN port scan to enumerate all possible accessible gates using **Nmap** before running an aggressive scan on open ports through the `-A` switch to gather as much information as possible:

![image](/HTB/Sendai/Sendai_images/2.png)
![image](/HTB/Sendai/Sendai_images/3.png)


Multiple services are running. Fundamentals here are:

**DNS** (Port 53)

**HTTP** (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**HTTPS** (Port 443)

**SMB** (Port 445)

**RDP** (Port 3389)

**WinRM** (Port 5985)

We can deduce from the retrieved information that we’re facing an Active Directory environment. The `sendai.vl` domain and the `dc.sendai.vl` FDQN Domain Controller has been identified.

(we will add these information in our `/etc/hosts` file as below)

![image](/HTB/Sendai/Sendai_images/4.png)

We first need to obtain valuable users candidate. Since we start the box without anything, let’s attempt to list available SMB shares through `smbclient` anonymously:

![image](/HTB/Sendai/Sendai_images/5.png)

It appears that it is allowed, permiting us to retrieve several shares. Let’s now use **SMBMap** to learn more about shares permissions before diving in blindly:

![image](/HTB/Sendai/Sendai_images/6.png)

As expected, `ADMIN$` and `C$` shares are forbidden to access as anonymous, just like `config`, `NETLOGON` and `SYSVOL`. 

However, we can observe that we have Read Only permission on two unusual shares: `sendai` and `Users`. Inspecting the first share reveals in the root directory an interesting incident.txt file that we could transfer to our local machine with the `get` command as below:

![image](/HTB/Sendai/Sendai_images/7.png)

It informs employees that password security policy isn’t enough strong among most of them, appending that these users have to reset their passwords; that’s interesting.

Inspecting the  security folder, an interesting guidelines.txt file could be identified; however, opening it or attempting to download was unsuccessful.

Listing the `transfer` directory reveals several users directory:

![image](/HTB/Sendai/Sendai_images/9.png)

Their content are empty, but this allows us to construct a valid users list as below:

![image](/HTB/Sendai/Sendai_images/10.png)

Inspecting the `Users` share, nothing very interesting could be identified, excepted that the **RecordedTV** software is seemingly installed on the target system, observing the `Public` folder:

![image](/HTB/Sendai/Sendai_images/11.png)

The `Default` folder could not be listed.

Our unauthenticated SMB enumeration is not achieved yet; we previously identified that the `IPC$` share left has **read permissions** as anonymous.

![image](/HTB/Sendai/Sendai_images/13.png)

This could allow us to perform **RID brute-forcing** in order to see whether additionals users could be identified through their **SID**. But before, let’s provide the list of users we just created to the tool —without passwords— to see which users are affected by a password reset required by the company's security policies. This could be achieved through the following `nxc` command:

`nxc smb TARGET_IP -u '' -p ''` 

![image](/HTB/Sendai/Sendai_images/14.png)

It appears that elliot.yates and thomas.powell domain users must reset their expired password, but since we doesn’t have any valid password to spray, we couldn’t yet reset their password through a tool like `smbpasswd`; we probably need to take our SMB enumeration a step further.

Let’s attempt to perform the previously mentioned **RID cycling** method in order to retrieve valid users by extracting their RIDs through the SID associated with their domain accounts, through the following command (unauthenticated, as Guest):

`nxc smb 10.129.71.41 -u 'Guest' -p '' --rid-brute`

![image](/HTB/Sendai/Sendai_images/15.png)

Several additionals domain users could be identified as the `SidTypeUser` SID object.

We could repeat the command, redirecting its output in a local file with `> usersrid.txt` so that we could cut properly usernames from the entire RID line. The following command can be executed to obtain a valid list of domain users for the penetration test:

`cat usersrid.txt | grep "SidTypeUser" | awk '{print $6}' | cut -d'\' -f2 > users.txt`

![image](/HTB/Sendai/Sendai_images/16.png)

Although we still didn’t recovered elliot.yates and thomas.powell expired passwords, now have a solid weapon to lead Active Directory attacks, although no authenticate access is granted yet.

The **Kerberos authentication flow** starts from an ***AS-REQ*** (Request) made from an authenticated user to the **Authentication service** (AS) in order to recieve a ***Ticket-Granting-Ticket*** (TGT) from the ***Key Distribution Center*** (KDC), owned by the Domain Controller (previously identified as `dc.sendai.vl`). This step is essential for users to access a service legitimately; it is crucial from a security perspective for obvious reasons.

However, there is a flag in the ***userAccountControl*** attribute of an Active Directory account. When this bit is set on an account, the KDC skips the **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an ***AS-REP*** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The Impacket suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use `impacket-GetNPUsers`. The following command could be executed to query the KDC for domain users potentially vulnerable to **AS-REP Roasting**:

`impacket-GetNPUsers thm.local/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/HTB/Sendai/Sendai_images/17.png)

Unfortunately, this attack is not conclusive here; none user has the `DONT_REQUIRE_PREAUTH` flag set within the domain, which probably would mean that we definitely need to find a way to reset the passwords for the users Elliot.yates and Thomas.Powell.

After wasting some time for a while, I learnt that NetExec was able to reset **blank** password through the`-M change-password` option (although that is not mentionned in the `—-help` manual page), unlike `smbpasswd` that prompts for the old user’s password since it is unable to treat empty string.

The following command could be executed to reset blank users passwords:

 `nxc smb TARGET_IP -u '<USER>' -p '' -M change-password -o USER='<USER>'  NEWPASS='<STRONG_PASSWORD>'`

![image](/HTB/Sendai/Sendai_images/18.png)

Passwords changes have been successful, allowing us to gain two authenticated access within the AD domain. We could leverage this access to use `bloodhound-python` in order to generate a ZIP file that we could export in the **BloodHound** GUI, allowing us to visualize AD domain permissions and relations. This can be achieved through the following command:

`bloodhound-python -u '<USER' -p '<PASSWORD>' -d <DOMAIN> -ns TARGET_IP -c All --zip`

![image](/HTB/Sendai/Sendai_images/19.png)

![image](/HTB/Sendai/Sendai_images/20.png)

We will now import the ZIP file in the BloodHound GUI.

![image](/HTB/Sendai/Sendai_images/21.png)

## Lateral Movement

Once file ingest completed, we could inspect a graph representing objects relations by clicking on *Explore*. Inspecting relations concerning both compromised AD objects, it can be identified that users are **members of** the ***Support*** domain group:

![image](/HTB/Sendai/Sendai_images/22.png)

![image](/HTB/Sendai/Sendai_images/23.png)

Moreover, we can observe that the concerned group has the **`GenericAll`** ACL permission over the ***ADMSVC*** group ****and Organizational Unit (OU) domain objects: 

![image](/HTB/Sendai/Sendai_images/24.png)

This permission grants “**Full Control**” over the target object. Since the target object here is a domain group, we could use this privilege to add the compromised principals into it. But why would we want to do that? Let’s inspect it more in depth; looking at the group’s **outbound object controll** tab, an interesting ACL node could be identified:

![image](/HTB/Sendai/Sendai_images/25.png)

The group *ADMSVC* has the `ReadGMSAPassword` permission that allows to `MGTSVC$` service account’s password through gMSA.

A **gMSA** (Group Managed Service Account) is a special Active Directory account that run Windows services witout having to manage their passwords manually. This has been designed to automatize service account passwords managing within the domain and to avoid password security policies issues. 

A controlled principal having this privilege over the service (in this case, being member of the ***ADMSVC*** group) would be able to dump gMSA password associated with the targeted object. This could be achieved with the `gMSADumper.py` tool, through the following command:

![image](/HTB/Sendai/Sendai_images/26.png)

It can be observed that the tool accepted our authentication as the Elliot.Yates user, but the `ReadGMSAPassword` ACL permission was not detected within the domain, outputting a list of users/groups eligible for the dump: ***ADMSVC*** group.

So let's use our **`GenericAll`** highly privileged rights to add Elliot.Yates to the ADMSVC group with the following command:

`net rpc group addmem 'ADMSVC' 'Elliot.Yates' -U sendai.vl/'Elliot.Yates'%'<RESETED_PASSWORD>' -S dc.sendai.vl`

Afterwards we will attempt to dump `mgtsvc$` service account password hash with the newly privileged user:

![image](/HTB/Sendai/Sendai_images/27.png)

We successfully recovered the NTLM hash associated with the `mgtsvc$` domain account. But without even trying to crack it offline through tools like John the Ripper or Hashcat, le’ts get back in **BloodHound** in order to inspect outband objects controls concerning the user account we just compromised:

![image](/HTB/Sendai/Sendai_images/28.png)

The user is member of the **Remote Management Groups**, which allows members to manage the system remotely through management protocol, such as **WinRM**. This means that we could obtain our initial foothold on the target system through a tool like `evil-winrm` and performing a Pass-the-Hash authentication attack in order to obtain a shell as MGTSVC$:

![image](/HTB/Sendai/Sendai_images/29.png)

We successfully obained our initial access on the system through the WinRM protocol.

The user flag could be retrieved in the `C:\` root directory.

![image](/HTB/Sendai/Sendai_images/30.png)

## Privilege Escalation

Inspecting user’s current local privileges through whoami /priv doesn’t reveal any misconfiguration that we could leverage to escalate our privileges.

After an in-depth local enumeration of the file system, we could retrieve  an intriguing `ImagePath` value description for the `helpdesk.exe` service:

![image](/HTB/Sendai/Sendai_images/31.png)

Harcoded credentials are exposed, compromising the **clifford.davey** domain user, which is a member of the privileged ***CA-Operator*** group, responsible for certificates management.

![image](/HTB/Sendai/Sendai_images/32.png)

We could use the `certipy` tool to inspect available certificates configured on the target domain. **Certipy** is an awesome tool that allows to find, request, forge, relay and do much more on ADCS certificate.

Being member of this group could be critical if any vulnerable certificate with sensitive permissions could be identified. The following command could be executed to list available certificates within the domain:

`certipy-ad find -u 'Clifford.Davey@sendai.vl' -p '<PASSWORD>' -dc-ip TARGET_IP -stdout`

![image](/HTB/Sendai/Sendai_images/33.png)

We could observe above that the sensitive `*SendaiComputer*` certificate template, owned by the Domain Administrator, has insecurely been configured with the **`GenericAll`** sensitive rights set for the *CA-Operators* group members, which could allow an attacker to exploit a **Vulnerable Certificate Template Access Control** vulnerability, which could lead to full domain compromise. 

The **ESC4 attack** (derived from ESC1) consists in **abusing** the excessive ACL permissions owned over a vulnerable certificate template object to **grants** **`GenericAll`** full access permission to a controlled domain entity and set the `User xc` and `Enrollee Supplies Subject` **AD CS certificate template attributes** in order to **impersonate** **any domain users** through their “***userPrincipalName”*** (UPN) so that we could authenticate with the Domain Controller (as the `Administrator` Domain Admin user for privilege escalation) with the user’s **private key (EKA)** paired with ****the ****Certificate Authority’s **signature (**CA’s public key**)**. 

Both properties respectively allows included users to **request** the vulnerable certificate and **usurpate** a ****domain ****user identity through its UPN referenced as a Subject Alternative Name (SAN) that the Domain Controller will trusts, that the domain controller will trust for authentication, since the issued certificate will have been legitimately signed by the CA.

!nrv.png

The following command could be executed to abuse the ACL permission targeting the vulnerable `*SendaiComputer*` certificate template in order to edit template attributes in our advantage: 

`certipy template -u clifford.davey@sendai.vl -p '<PASSWORD>' -template SendaiComputer -dc-ip <TARGET_IP>`
⚠️***WARNING***⚠️(Although that is not required in this lab, you MUST append the `save-old` output option to the command to save out current AD CS settings before exploitation in a `.json` file if you carry out this attack in a corporate penetration test; that is because the command will make the tool **overwrite** the AD CS object configuration.)

![image](/HTB/Sendai/Sendai_images/34.png)

![image](/HTB/Sendai/Sendai_images/35.png)

Certificate’s template properties have been successfully updated. We could now display the template to verify that changes have been effective:

![image](/HTB/Sendai/Sendai_images/36.png)

Indeed, that is the case; All authenticated users in the domain have now full control over the `*SendaiComputer*` certificate template, allowing them to use it for any purpose. But what interests us most is that they have earned **enrollment permissions**, which means that they  `Enrollee Supplies Subject` template attribute flag set to `*True*`, which means that the enrollee (client) could supplies a **Subject Alternative Name** (SAN) in order to represent a domain user using the certificate.

Combined with the ***ExportableKey*** private key flag and the `Client Authentication` propertie template set to `True` that allows the certificate to be used for domain user authentication, Impersonation Attack is possible as we will use user’s UPN that we want to usurpate (such as `Administrator` user). This could be achieved with the following command:

`certipy req -u Elliot.Yates@sendai.vl -p <PASSWORD> -ca sendai-DC-CA -target dc.sendai.vl -template VulnTemplate -upn Administrator@sendai.vl -dc-ip TARGET_IP`

(I demonstrate here that the authentication context concerning the Clifford.Davey group-privileged user isn’t required since the `User Enrollable Principals` template attribute has been set, allowing *Authenticated Users* group members to request a certificate from this template)

![image](/HTB/Sendai/Sendai_images/37.png)

We successfully requested and recieved a legitimate certificate signed by the CA, allowing us to dangerousely impersonate the victim ****user. The user’s private key signed certificate has been looted and outputted in an `administrator.pfx` file, in our local directory. 

This certificate allows us to authenticate with the Domain Controller, as it legitimately trusts the Public Key Infrastructure (PKI). The following command could be executed to authenticate as the `Administrator` user with the signed certificate and request a TGT to the KDC in order to retrieve user’s NTLM hash:

![image](/HTB/Sendai/Sendai_images/38.png)

It appears here that the script encounters a SID mismatch issue concerning the **`Administrator`** user account, which means that we need to impersonate someone else in order to get authenticated properly. Getting back in BloodHound, we could observe that the **Anthony.Smith** user has the **`Tier Zero`** level access, that represents the  “Domain Control” highest permissions within the AD domain. This means that we could impersonate this user in order to take advantage of the excessive trust granted by the Domain Controlelr to carry out a **DCSync attack**. This attack consists in abusing the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` sensitive ACLs that are granted by default to **Domain Admin** users in order to replicate sensitive data from the DC, such as **`NTDS.dit`** secrets.

… I then remembered that certificate could not contain the correct object SID extension required for strong certificate mapping, which would cause authentication to fail. To fix it, we have to specify the user’s SID directly within the command. We can retrieve it through the use of the `impacket-lookupsid` tool from the Impackets suite:

![image](/HTB/Sendai/Sendai_images/39.png)

We could then repeat the certificate request authentication command, appending `-sid` option specifying the recovered domain SID **paired** with the user’s RID so that user account SID will be identified and treated properly (for the **`*Administrator*`** user, it is **500).**

However, PKINIT definetly wont works on this AD machine:

![image](/HTB/Sendai/Sendai_images/40.png)

Fortunately for us, the `ldap-shell` tool option allows us to gain a LDAP channel session as the **Administrator** user.

![image](/HTB/Sendai/Sendai_images/42.png)

Dont underestimate this unusual shell which seems limited, our privileged access allows us to perform critical modifications on the target domain objects through their build-in commands. We could for example set the `set_dontreqpreauth` option as true in order to set the `DONT_REQ_PREAUTH` flag that we previously explained, we could even grant the GenericAll ACL permission to a targeted object through the `grant_control` command. 

But even more simple, we could change **`Administrator`** user password through the `change_password` command, without even having it in plaintext (similarly to the `ForceChangePassword` ACL).

![image](/HTB/Sendai/Sendai_images/43.png)

User password has successfully been modified, allowing us to authenticate on the Domain Controller machine through `evil-winrm`:

![image](/HTB/Sendai/Sendai_images/44.png)

We successfully escalated our privileges to the top-privileged Domain Admin user on the `DC$` Domain Controller machine, resulting in a full compromise of the target Active Directory domain. This can be confirmed with the `whoami /priv` and the hostname command

![image](/HTB/Sendai/Sendai_images/45.png)

Indeed, full privileges are granted to us on the DC machine. The final flag could be retrieved in the `C:\Users\Administrator\Desktop` users’s desktop directory.

![image](/HTB/Sendai/Sendai_images/46.png)

Challenge completed!!

![image](/HTB/Sendai/Sendai_images/47.png)

![image](/HTB/Sendai/Sendai_images/49.png)

## Conclusion

This box was very similar to the *Delegate* challenge: I really appreciated to chain domain object configuration vulnerabilities in order to gradually moving toward compromising the domain: Several ACLs were exploited to carry out a chained attack on the domain's user and machine accounts and several techniques and technical skills were learned. Moreover, I loved to learn more about AD CS attacks through identifying their different types, there are up to eight vulnerability cases concerning this topic, and I strongly encourage you to check the blog from RedFoxSec that helped me a lot during this lab to understand with precision all technologies and concepts we evoked: PKI, `Enrollee Suplies Object` and the others decisive certificate template properties we leveraged to carry out this domain compromise, various Kerberos-related terms and processes,…)

This challenge demonstrates how multiple low-severity domain object underestimated misconfigurations could be chained to lead in a full Active Directory domain compromise. 

# Thank you for reading!

(and for your time, that was long)

![image](/HTB/Sendai/Sendai_images/50.png)
