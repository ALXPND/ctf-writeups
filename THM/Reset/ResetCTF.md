## Welcome everyone!

Today, we will explore “Reset”, a *hard-rated* machine from TryHackMe, our first one together!

For this challenge, I want to be as real as possible. I will demonstrate my method, going through failures and rabbit holes. This is not the absolute way to complete this room. I started all of this 10 months ago, so I definitely make mistakes.

Without further ado, let’s get started!

![image](/THM/Reset/Reset_images/1.png)

## Information Gathering

First, we will perform an aggressive scan with Nmap to gather as much information as possible about the system we’re targeting:

![image](/THM/Reset/Reset_images/2.png)

Multiple services are running. The fundamentals here are:

DNS (Port 53)

Kerberos (Port 88)

RPC (Port 135)

NetBIOS (Port 139)

LDAP (Port 389)

SMB (Port 445)

RDP (Port 3389)

WinRM (Port 5985, unexposed here due to VM issue)

We can deduce from the retrieved information that we’re facing an Active Directory environment. The thm.corp domain and the HayStack.thm.corp FQDN Domain Controller has been identified.

We first need to obtain valuable user candidates. Since we start the box without anything, let’s attempt to list available SMB shares through smbclient anonymously:

![image](/THM/Reset/Reset_images/3.png)

It worked, we listed multiple shares, including a non-default Data share. Attempting to access it anonymously is conclusive. Moreover, we could observe three interesting files in an onboarding folder:

![image](/THM/Reset/Reset_images/4.png)

These files can be downloaded with the get command. The first one exposes a credential that appears to be a default credential for the company employees as an initial access:

![image](/THM/Reset/Reset_images/5.png)

We can even identify a valid Lily.Oneill username in the second PDF file:

![image](/THM/Reset/Reset_images/6.png)

I spent a lot of time attempting to authenticate with a service using these credentials, but they were just rabbit holes… nothing could be leveraged there. Since LDAP required valid credentials for querying any AD object data, I attempted to extract valid usernames as the Guest user through SMB from their RIDs (extracted from the SID (example: Administrator=500)), by using nxc and it worked:

![image](/THM/Reset/Reset_images/7.png)

30 users have been successfully identified. Wonderful. I created a users.txt file containing the following content (.+_ pattern):

Ernesto_Silva Ernesto.Silva Tracy_Carver Tracy.Carver Shawna_Bray Shawna.Bray Cecile_Wong Cecile.Wong Cyrus_Whitehead Cyrus.Whitehead Deanne_Washington Deanne.Washington Elliot_Charles Elliot.Charles Michel_Robinson Michel.Robinson Mitchell_Shaw Mitchell.Shaw Fanny_Allison Fanny.Allison Julianne_Howe Julianne.Howe Roslyn_Mathis Roslyn.Mathis Daniel_Christensen Daniel.Christensen Marcelino_Ballard Marcelino.Ballard Cruz_Hall Cruz.Hall Howard_Page Howard.Page Stewart_Santana Stewart.Santana Lindsay_Schultz Lindsay.Schultz Tabatha_Britt Tabatha.Britt Rico_Pearson Rico.Pearson Darla_Winters Darla.Winters Andy_Blackwell Andy.Blackwell Lily_Oneill Lily.Oneill Cheryl_Mullins Cheryl.Mullins Letha_Mayo Letha.Mayo Horace_Boyle Horace.Boyle Christina_Mccormick Christina.Mccormick Morgan_Sellers Morgan.Sellers Marion_Clay Marion.Clay Ted_Jacobson Ted.Jacobson Augusta_Hamilton Augusta.Hamilton Trevor_Melton Trevor.Melton Leann_Long Leann.Long Raquel_Benson Raquel.Benson

## Initial Access

This valuable information may be helpful for us. Performing password spraying (trying a single password against multiple users) with the ResetMe123! default password doesn’t reveal anything conclusive. But since we now have valid usernames, we could attempt to target these AD objects through multiple features, such as their privileges or relationships. However, we are still missing an authenticated access.

Then, I thought about performing AS-REP Roasting.**

This attack could be accessible for us because this is not about authenticating legitimately to a service account in order to extracting its TGS and recovering a password from its derived-key (like Kerberoasting). Here, we will simply verify whether AD user accounts have the the “PRE AUTHENTICATION DISABLED” flag enabled. This feature allows a user, when requesting a TGT (Ticket-Granting-Ticket) from the KDC (Key Distribution Center), to obtain a response from the Domain Controller without needing Kerberos pre-authentication. This is dangerous because we could obtain a response  from the DC (known as AS-REP) that would allow us to extract an AS-REP hash derived from the user’s password.

This attack can be achieved with impacket-GetNPUsers. The following command has been executed to query the domain user accounts:

impacket-GetNPUsers thm.corp/ -no-pass -usersfile users.txt -dc-ip TARGET_IP

![image](/THM/Reset/Reset_images/8.png)

Three AS-REP hashes have been extracted from the insecure ‘PRE-AUTHENTICATION DISABLED’ setting configured on the following users:

Ernesto_Silva

Tabatha_Britt

Leann_Long

We now have three hashes derived from valid passwords. But are they weak enough to crack? Let’s verify it with Hashcat. To know the hash mode to use for offline cracking, we could use the —-identify option:

![image](/THM/Reset/Reset_images/9.png)

We can then construct the following command to attempt to crack these AS-REP hashes:

hashcat -m 18200 -a 0 <HASH_FILE> /usr/share/wordlists/rockyou.txt

Ernesto_Silva and Leann_Long password recoveries were unsuccessful. However, Hashcat displays something different for Tabatha_Britt:

![image](/THM/Reset/Reset_images/10.png)


The password has been successfully recovered! We just gained valid credentials for digging deeper into this domain exploitation process.

Now that we have valid credentials, we could visualize the AD relations graph concerning the user we just compromised by using bloodhound-python in order to generate and import the file containing information related to it in BloodHound. To do this, we could run the following command:

bloodhound-python -u 'Tabatha_Britt' -p 'REDACTED' -d thm.corp -ns 10.128.171.241 -c All --zip

![image](/THM/Reset/Reset_images/11.png)

Let’s now import it in BloodHound:

![image](/THM/Reset/Reset_images/12.png)

Once the file ingestion was completed, we could inspect objects relations in the Explore section. Inspecting relations concerning the Tabatha.Britt AD object, we can identify that the user is a member of the ‘Remote Desktop Users’ group:

![image](/THM/Reset/Reset_images/13.png)

This group enables Tabatha Britt to access the target system remotely over RDP (Remote Desktop Protocol), which means that we could gain our initial foothold on the target system through this protocol.

We could use xfreerdp to authenticate and access the Windows machine remotely with the following command:

xfreerdp /u:Tabatha_Britt /p:REDACTED /v:reset.thm

![image](/THM/Reset/Reset_images/14.png)

We successfully gained an initial access over the system as the Tabatha_Britt user.

Let’s now run the Command Prompt (cmd.exe) in order to see what we can do locally:

![image](/THM/Reset/Reset_images/15.png)

## Privilege Escalation

Listing our local privileges with whoami /priv doesn’t reveal anything interesting:

![image](/THM/Reset/Reset_images/16.png)

Listing the C:\Users directory allows us to identify multiple user accounts on the local system: automate and Cecile_Wong.

A simple local enumeration led me to check the WinLogon registry in order to see whether any user has credentials stored in plaintext due to an insecure auto-login configuration. This can be done through the following command:

reg query “HKLM\Software\Microsoft\Windows NT\CurrentVersion\WinLogon”

17.![image](/THM/Reset/Reset_images/17.png)


Indeed, that is the case!

We just retrieved the cleartext password of the automate user, allowing us to hijack its Windows session through this simple Living-Off-The-Land command:

runas /user:automate cmd.exe

![image](/THM/Reset/Reset_images/18.png)

We successfully pivoted our access to the automate user account. User flag can be claimed in the C:\Users\automate\Desktop directory.

![image](/THM/Reset/Reset_images/19.png)

We could suggest that CECILE_WONG is the highly privileged user containing the final flag. Let’s see if we could find a way to escalate to her:

Inspecting the automate AD object relationship in BloodHound reveals an interesting group membership associated to it:

![image](/THM/Reset/Reset_images/20.png)

It appears that this user is a member of the Remote Management Users group, which works similarly to the Remote Desktop Users group, but for WinRM. We could leverage this protocol to escape the unstable keyboard inputs.

However, after an in-depth enumeration of the local system, we couldn’t find a way to abuse services or any features. Maybe we have to get back in our AD graph, in BloodHound.



## Latteral Movement


Inspecting the outbound object control list concerning the automate user doesn’t reveal anything interesting. However, inspecting the one belonging to the previously compromised Tabatha_Britt user, we could identify that she has the GenericAll ALC permission over two domain users: Shawna_Bray and Raquel_Benson:

![image](/THM/Reset/Reset_images/21.png)

This permission is also known as “Full Control”. This could allow the permission owner to modify domain object features concerning the victim, including its password.

We could run the following command in order to reset users’ passwords:

net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController”

![image](/THM/Reset/Reset_images/22.png)

No error has been returned, we successfully replaced Shawna_Bray and Raquel_Benson passwords with a password we control. Since these users are now compromised, let’s inspect their outbound relations to see what we are able to do now. Raquel_Benson doesn’t have any interesting membership or relationship with other domain objects. However, Shawna_Bray has a very interesting ForceChangePassword ACL permission over the Cruz_Hall domain user:

![image](/THM/Reset/Reset_images/23.png)

This ACL explicitly allows its owner to change the target’s password. We could simly re-run the previous net command to perform an authenticated password reset of Cruz_Hall, but this time, thanks to the permission, as Shawna_Bray.

Once it has been done, we could inspect Cruz_Hall relations to see what comes next:

![image](/THM/Reset/Reset_images/24.png)

We can observe from this node that the domain user Cruz_Hall has the ForceChangePassword ACL configured as well over the domain user Darla_Winters, allowing us to reset its password with the same command that we have done so far (while being authenticated as Cruz_Hall this time).



## DC Compromise


Inspecting Darla_Winters’ AD configuration reveals a critical Constrained Delegation Privilege execution privilege configured.

![image](/THM/Reset/Reset_images/25.png)

This privilege allows Darla_Winters to delegate to a service identified by an SPN (present in the *msDS-AllowedToDelegateTo* LDAP property) The constrained delegation primitive allows a principal to authenticate as any user to specific services. We can identify them in the

The constrained delegation primitive allows a principal to authenticate as any user to specific services (found in the msds-AllowedToDelegateTo LDAP property) on the target computer. A node with this permission can impersonate any domain account (including Domain Admins) to the specific service. This could allow an attacker to extract a Kerberos credential from the ST (Service Ticket).

Indeed this is possible through the S4U2Self/S4U2Proxy processes. These features are Kerberos extensions that allows an authenticated principal presenting a valid TGT to query a TGS as another domain user. Here, combined with our Constrained Delegation Privileges,  we could leverage this to impersonate a user that will then authenticate to a service identified by an SPN allowed for delegation.

We could identify a valid SPN to use in the BloodHound source node tab:

![image](/THM/Reset/Reset_images/26.png)

Let’s use the cifs/HayStack.thm.corp SPN allowed for delegation. To retrieve the NT hash responsible for initial TGT delivery, we could run:

iconv -f ASCII -t UTF-16LE <(printf 'PASSWORD') | openssl dgst -md4

![image](/THM/Reset/Reset_images/27.png)

We could now use impacket-getST from the Impacket suite to delegate the domain Administrator user account through the following command in order to authenticate with the cifs/HayStack.thm.corp service identified as allowed for delegation:

impacket-getST -spn 'cifs/Haystack.thm.corp' -impersonate 'Administrator' -hashes '**:**DARLA_WINTER_PASSWD_NT-HASH' 'thm.corp/Darla_Winters'

![image](/THM/Reset/Reset_images/28.png)

It worked! We can observe that S4U2Self/S4U2Proxy requests were successful. The Administrator Domain Admin user could be delegated, allowing us to receive the Kerberos credential cache (.ccache) containing the Service Ticket.

(to store the credential cache in the KRB5CCNAME environment variable, run export KRB5CCNAME=<.ccache>)

Now that we have a valid Kerberos authentication context as the Administrator user for the specific cifs/HayStack.thm.corp service, we could use impacket-secretsdump to get NTDS.dit secrets through a DCSync attack in order to extract all domain users’ hashes.

This highly-privileged attack consists in impersonating the DC’s role in a way to replicate sensitive data through the DRSUAPI protocol. Indeed, Active Directory domains need data replication between Domain Controllers in order to synchronize everything properly within the AD domain. It is a legitimate feature in and of itself, and it also requires significant privileges. But since we just compromised the Domain Admin user, all privileges are already granted.

Please note this method is achieved through -just-dc because we are querying the target Domain Controller. Without it, the tool accesses the local Windows registry hives through the WindowsRegistry service in order to retrieve hashes stored in the Security Account Manager (SAM) by decrypting it with the local system bootKey extracted from the SYSTEM hive.

Also, don’t forget to use the -k switch to enable Kerberos authentication and use the credential cache previously exported instead of the Administrator’s password.

![image](/THM/Reset/Reset_images/29.png)

All NTDS.dit secrets have been successfully recovered, including domain users hashes.

At this point, even a strong password would not be necessary with evil-winrm: the tool allows to lead a Pass-the-Hash attack, which means that we don’t have to worry about password cracking. We could use Cecile_Wong’s NT hash to authenticate directly to the local Windows session identified earlier with the following command:

evil-winrm -i TARGET_IP -u CECILE_WONG -H <NT_HASH>

(NT hash only)

Since WinRM is enabled and Cecile_Wong is in the Domain Admin group, we are guaranteed that WinRM access is possible:

(It is worth noting that Cecile_Wong and Administrator passwords NTLM hashes are the same)

![image](/THM/Reset/Reset_images/30.png)

We successfully escalated our privileges to the Cecile_Wong Domain Admin user of the Domain Controller machine, resulting in a total compromise of the Active Directory domain targeted with full access and control over it.

We still can verify it with whoami , whoami /priv and hostname commands:

![image](/THM/Reset/Reset_images/31.png)

![image](/THM/Reset/Reset_images/32.png)

Indeed, full privileges have been granted on the target Domain Controller, which is critical from a security perspective.

The final flag can be claimed in the C:\Users\CECILE_WONG\Desktop directory.

![image](/THM/Reset/Reset_images/33.png)


Challenge completed!


![image](/THM/Reset/Reset_images/34.png)


## Conclusion

And that’s it, Reset is completed.

It started with a simple anonymous SMB enumeration, then turned into a long chain of different Active Directory techniques: AS-REP Roasting, BloodHound enumeration, password resets through ACLs, constrained delegation, S4U2Self/S4U2Proxy, DCSync and finally Pass-the-Hash.

Rabbit holes were justified, however i'd like to add that the target VM was very unstable (shutdowns, WinRM port being filtered, …).

BloodHound was especially useful here to understand how one compromised account could lead to another, until reaching Darla_Winters and eventually the Domain Admin level.

This was also my first Hard-rated TryHackMe machine, so completing it was a pretty important milestone for me. I definitely made mistakes along the way, and there are probably cleaner ways to complete some parts, but this writeup reflects how I actually approached the machine

## Thank you for reading!

![image](/THM/Reset/Reset_images/35.png)
