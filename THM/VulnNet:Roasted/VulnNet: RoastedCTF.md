## Welcome everyone!

Today, we will explore **VulnNet: Roasted**, an easy-rated machine from TryHackMe.

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/1.png)
(Please ensure that you have added the following line `TARGET_IP vulnnet-rst.thm vulnnet-rst.local` to your local `/etc/hosts` file before starting the challenge to avoid any domain name resolution issues.)

## Information Gathering

First, we will perform an aggressive scan with **Nmap** to gather comprehensive information about the target:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/2.png)

Multiple services are running on the target:

**DNS** (Port 53)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**WinRM** (Port 5985)

Moreover, the `vulnnet-rst.local` **domain** has been identified by the Nmap Script Engine. Based on these findings, we can reasonably assume that we are facing an **Active Directory** environment.

Since we don’t have any credentials, let’s attempt to enumerate SMB shares anonymously using `smbclient` and the `-N` switch:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/3.png)

We can identify multiple available shares. However, only the `SYSVOL` share is accessible anonymously, and directory listing is disabled, so there is nothing interesting to retrieve for now.

Since we don’t have any valid domain usernames that we could leverage to initiate further attacks, let’s attempt to enumerate domain users through their **RID** (Relative Identifier), which is part of their **SID** (Security Identifier). This can be done using **NetExec (`nxc`)** with the `--rid-brute` option:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/4.png)

We successfully extracted multiple valid usernames using this technique. We can create a `users.txt` file containing the following domain user accounts (the `SidTypeUser` entries identified above):

`a-whitehat`

`t-skid`

`j-goldenhand`

`j-leet`

`enterprise-core-vn`

## Initial Access

Now that we have valid usernames, we can target them with multiple attacks. Since we don’t have any authenticated access yet, we can start by checking whether any of the identified users are vulnerable to an **AS-REP Roasting** attack.

AS-REP Roasting is possible when Kerberos pre-authentication is disabled for an account, which corresponds to the `DONT_REQUIRE_PREAUTH` flag being enabled. In this situation, an attacker can request an **AS-REP** (Authentication Service Reply) from the **KDC** (Key Distribution Center) without first proving knowledge of the user's password.

The response contains encrypted client-specific material derived from the user's long-term key. This material can be extracted and cracked offline to recover the user's password. The AS-REP also contains a **TGT** (Ticket-Granting Ticket), but the TGT itself is not what is cracked during AS-REP Roasting.

This can be checked with the `impacket-GetNPUsers` tool from the Impacket suite:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/5.png)

It appears that the `t-skid` user has the required flag set, allowing us to extract AS-REP material derived from its password. We can save it to a `t-skid_hash.txt` file and attempt to crack it offline using **Hashcat**:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/6.png)

The `t-skid` password has been successfully recovered in plaintext.

Now that we have valid credentials, we can use **BloodHound** to visualize relationships between domain objects. This can be achieved with `bloodhound-python` using the following command:

`bloodhound-python -u 't-skid' -p 'PASSWORD' -d vulnnet-rst.local -ns TARGET_IP -c All --zip`

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/7.png)

Once the ZIP file has been generated, let’s import it into BloodHound and visualize the AD graph concerning the user we have just compromised in the *Explore* section:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/8.png)

It appears that the user does not have any interesting privileges or group memberships.

However, since we now have authenticated access to the domain, we can attempt to perform **Kerberoasting**.

This attack is somewhat similar to AS-REP Roasting because the goal is also to obtain crackable material and recover a user's password offline. However, instead of targeting accounts without Kerberos pre-authentication, Kerberoasting targets accounts associated with **SPNs** (Service Principal Names).

An SPN allows a service to be associated with a specific account in Active Directory. When an authenticated user requests access to a service identified by an SPN, the KDC issues a **TGS** (Ticket-Granting Service ticket). The service ticket contains encrypted material protected using the key associated with the account that owns the SPN. When that account is a user/service account with a crackable password, an attacker can extract this material and attempt to crack it offline.

Therefore, we can query the domain for accounts associated with SPNs and request their service tickets. This can be achieved with another tool from Impacket: `impacket-GetUserSPNs`.

`impacket-GetUserSPNs vulnnet-rst.local/t-skid@tj072889* -dc-ip 10.129.160.134 -request`

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/9.png)

We successfully extracted crackable material for the `enterprise-core-vn` user from the requested TGS. As we can observe, the `CIFS/vulnnet-rst.local` SPN is associated with the `enterprise-core-vn` account, making it a potential Kerberoasting target.

Let’s attempt to crack it offline using Hashcat once again:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/10.png)

We successfully cracked the user's password.

Since we have now compromised another user, let’s return to BloodHound and investigate what this account can access:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/11.png)

It appears that the user is a member of the ***Remote Management Users*** group. This group generally allows users to access systems remotely through the WinRM protocol for remote administration.

Let’s attempt to gain our initial foothold on the target machine by leveraging this access through `evil-winrm`:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/12.png)

It works! We successfully obtained an initial foothold on the target machine through the WinRM protocol.

The first flag can be retrieved from the `C:\Users\enterprise-core-vn\Desktop` directory.

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/13.png)

## Privilege Escalation

We now need to escalate our privileges further in order to retrieve the root flag.

After an in-depth enumeration of the local system, nothing interesting could be identified for local privilege escalation.

However, if you remember, we previously identified multiple SMB shares that were not accessible anonymously. Now that we have valid credentials, we can enumerate them again and potentially find something interesting.

The `ADMIN$` share is still inaccessible. The `C$` administrative share does not contain much more information than what we can already observe from our Windows session. However, the `SYSVOL` share is now accessible and allows directory listing.

A `vulnnet-rst.local` directory can be identified. This folder contains another `scripts` directory which hosts a very interesting password reset script:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/14.png)

Inspecting it with the `more` command reveals a critical password disclosure concerning the **a-whitehat** user:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/15.png)

Let’s verify the account's privileges and relationships within the domain using **BloodHound**:

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/16.png)

It appears that we have just compromised a user who is a member of the **Domain Admins** group.

This is critical because the account therefore has the privileges required to perform **DCSync** against the domain. DCSync abuses Active Directory's replication mechanism to request credential material from a Domain Controller through the **DRSUAPI** replication protocol.

We can use the `impacket-secretsdump` tool to perform this attack:

`impacket-secretsdump 'a-whitehat:PASSWORD@vulnnet-rst.local' -just-dc`

With the `-just-dc` option, `secretsdump` focuses on the domain controller's Active Directory data and uses the domain replication mechanism rather than dumping the local SAM. This allows us to retrieve credential material for domain accounts, including NT hashes and Kerberos keys.

It is important to distinguish this from running `secretsdump` without `-just-dc`. Without this option, Impacket can also use the **RemoteRegistry** service to access Windows registry hives such as `SAM` and `SYSTEM`. The `SYSTEM` hive contains the information required to derive the system **bootKey**, which can then be used to decrypt protected SAM data and retrieve local account hashes.

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/17.png)

All relevant domain secrets have been successfully recovered, including the NT hash of the domain `Administrator` account.

At this point, we do not need to crack the Administrator password. The recovered NT hash can be used directly to authenticate through **Pass-the-Hash**.

`evil-winrm` supports Pass-the-Hash authentication, allowing us to provide the NT hash directly:

`evil-winrm -i TARGET_IP -u Administrator -H <NT_HASH>`

(NT hash only.)

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/18.png)

We successfully escalated our privileges to the `Administrator` account on the **Domain Controller**, resulting in full administrative control over the domain.

The final flag can be retrieved from the `C:\Users\Administrator\Desktop` directory.

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/19.png)

Challenge completed!

![image](/THM/VulnNet:Roasted/VulnNet:Roasted_images/20.png)


## Conclusion

*VulnNet: Roasted* was a good introduction to several common Active Directory attack techniques and, more importantly, to how they can be chained together to progressively compromise a domain.

The attack started with anonymous SMB enumeration and RID brute-forcing, which allowed us to identify valid domain accounts. We then leveraged an account configured without Kerberos pre-authentication to perform AS-REP Roasting and recover the password of `t-skid`. With valid domain credentials, Kerberoasting became possible, allowing us to exploit the `CIFS/vulnnet-rst.local` SPN associated with `enterprise-core-vn` and recover its password.

The `Remote Management Users` membership of `enterprise-core-vn` then provided remote access to the Domain Controller through WinRM. From there, authenticated SMB access exposed the `SYSVOL` share and, more importantly, a password reset script containing credentials for `a-whitehat`.

The newly compromised account was a member of `Domain Admins`, giving us the privileges required to abuse Active Directory replication. Using DCSync through the DRSUAPI protocol, we were able to retrieve credential material for domain accounts through data replication, including the NT hash of the domain `Administrator`. Finally, Pass-the-Hash allowed us to authenticate as `Administrator` through WinRM, concluding in a full compromise of the Active Directory domain targeted.

Overall, the machine demonstrates how several relatively small Active Directory weaknesses — exposed account information, disabled Kerberos pre-authentication, service accounts with SPNs, sensitive credentials stored in SYSVOL, and excessive domain privileges — can be chained together to achieve a complete domain compromise.

**Thank you for reading!**
