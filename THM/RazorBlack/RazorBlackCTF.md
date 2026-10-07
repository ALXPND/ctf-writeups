## Welcome everyone!

Today, we will explore “*RazorBlack*”, a medium-rated CTF from **TryHackMe**.

![image](/THM/RazorBlack/RazorBlack_images/1.png)

## Information Gathering

First, we will perform an aggressive scan with Nmap using the -A switch to obtain comprehensive results of the system we’re targeting:

![image](/THM/RazorBlack/RazorBlack_images/2.png)

Multiple services have been identified. Fundamentals here are:

**DNS** (Port 53)

**HTTP** (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**RDP** (Port 5985)

It could be deduced from the port scanning that we face an Active Directory environment. The `raz0rblack.thm` **AD domain** and the `HAVEN-DC.raz0rblack.thm` FQDN **Domain Controller** has been identified by Nmap Script Engine’s default scripts (NSE).

(Add it in your local `/etc/hosts` file to avoid any issues)

Since we start this box without any credentials, we could attempt to access the **SMB shares** through `smbclient` as anonymous with the `-N` switch. However, directory listing appears to be disabled without an authenticated access:

![image](/THM/RazorBlack/RazorBlack_images/3.png)

Moreover, attempting to access standard `C$`, `SYSVOL` and `NETLOGON` share is not possible either.

Initial valid usernames is a need to attempt any form of attack.

After going around in circles for quite a while (and before having to restart the target VM), I realized that my port and service reconnaissance phase using Nmap was misleading: the unusual port 111 is actually active as well, hosting an **NFS share**…

![image](/THM/RazorBlack/RazorBlack_images/4.png)

The following command could be executed to list available NFS share on the target machine:

`showmount -e TARGET_IP`

![image](/THM/RazorBlack/RazorBlack_images/5.png)

A `/users` share which appears to be publicly accessible was found. Indeed, that’s way much better with this. 

To access this local share from our machine, we have to link it to a local folder through the following command (**as root**):

`mount -o rw TARGET_IP:/users /LOCAL/SHARE`

![image](/THM/RazorBlack/RazorBlack_images/6.png)

We successfully mounted up the NFS share to our local machine, permiting ourselves to access it and identify two interesting files: `sbradley.txt` and `employee_status.xlsx` which is a ZIP archive containing XML files. We could open it using `libreoffice` from our terminal, after which you'll see this pop-up window

![image](/THM/RazorBlack/RazorBlack_images/7.png)

Because we don’t own this file, we could simply access it in Read Only mode through clicking on the *Open R/O* button:

![image](/THM/RazorBlack/RazorBlack_images/8.png)

We are greeted with a **CTF PLAYER** dashboard storing several names. We can identify the steven bradley which appears to be associated to the s.bradley.txt file found in the same share. Accessing it reveals the Steven’s flag required for the room. We could construct a solid users list at this stage from the dashboard we just discovered. It contains the twelve usernames identified based on the `sbradley` strings AD object pattern previously identified, as follows:

![image](/THM/RazorBlack/RazorBlack_images/9.png)

We now have the weapon needed to lead our Active Directory attacks, although no authenticate access is granted yet.

Since the **Kerberos authentication flow** start from an ***AS-REQ*** (Request) made from an authenticated user to the **Authentication service** (AS) in order to recieve a ***Ticket-Granting-Ticket*** (TGT) from the ***Key Distribution Center*** (KDC), owned by the Domain Controller (previously identified as `ad.thm.local`). This step is essential for users to access a service legitimately; it is crucial from a security perspective for obvious reasons.

However, there is a flag in the ***userAccountControl*** attribute of an Active Directory account. When this bit is set on an account, the KDC skips the **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an ***AS-REP*** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The Impacket suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use **impacket-GetNPUsers**. The following command could be executed to query the KDC for domain users 

`impacket-GetNPUsers thm.local/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/THM/RazorBlack/RazorBlack_images/10.png)

It appears that Tyson Wiliams has the `DONT_REQUIRE_PREAUTH` flag set: AS-REP responses have been recovered from the tool for the **twilliams** domain users!

We could paste the extracted AS-REP Roast hashes derived from users passwords in order to attempt to perform offline cracking on them through **Hashcat**. The hash mode to provide with the `-m` switch could be identified with the `—-identify` option as below:

![image](/THM/RazorBlack/RazorBlack_images/11.png)

Hence, the following command could be executed to attempt hash cracking on the file:

`hashcat -m 18200 -a 0 TysonWilliams_hash.txt /usr/share/wordlists/rockyou.txt`

![image](/THM/RazorBlack/RazorBlack_images/12.png)

The user’s password has successfully been cracked,allowing us to recover it in plaintext!

Now that we have an authenticated access within the Active Directory domain, let’s use **BloodHound** **to visualize Active Directory objects relations** in order to learn more about the user we just compromise, and, secondly, to see if an interaction with any entity in the domain could lead to a more severe compromise.

`bloodhound-python` could be used to generate a ZIP file that would be uploadable from the BloodHound UI. The following command can be used:

`bloodhound-python -u twilliams -p <PASSWORD> -d raz0rblack.thm -ns TARGET_IP -c All --zip`

![image](/THM/RazorBlack/RazorBlack_images/13.png)

Now, let’s import the file in BloodHound:

![image](/THM/RazorBlack/RazorBlack_images/14.png)

Once the file ingest completed, click the *Explore* button and select the compromised twilliams user to visualize privileges and relationships.

Looking at the Outbound Object Control section, nothing useful for us could be identified. The user no has any node with any users present on the domain. However, listing Domain Users group members reveals another domain user that was not exposed in the XML dashboard we extracted from the NFS share previously discovered: **xyan1d3**.

![image](/THM/RazorBlack/RazorBlack_images/15.png)

Interesting. We could add this user in our user list for further attacks.

Our authenticated access also grants us the ability to perform authenticated AS-REQ to the KDC in order to query service ticket; this is legitimate Kerberos behavior. However, as an attacker, we could leverage the Kerberos authentication flaw to extract the derived key from the service account password targeted through the TGS-REQ: this is called **Kerberoasting**.

Although our user list does not appears to contain any service domain account, we just extracteed a mysterious **xyan1d3** newly discovered user, so why not give it a try? It could be automate account or something like this — The important thing here is that the domain account we are targeting has a valid SPN so that the KDC will recognize it and treat it as a service account within the AD domain in order to authenticate properly.

We could use `impacket-GetUserSPNs` to try to request, as the **twiliams** user a TGS for each user present in the user list through the following command:

`impacket-GetUserSPNs raz0rblack/twilliams -dc-ip TARGET_IP -request`

(Then, provide twilliams’s password)

![image](/THM/RazorBlack/RazorBlack_images/16.png)

The `HAVEN-DC/xyan1d3.raz0rblack.thm:60111` SPN associated with the xyan1d3 user has been identified, suggesting that a service has been configured on local port 60111 through this domain user. This configuration allowed us to successfully lead our Kerberos attack in a password hash recovery concerning the xyan1d3 domain user. We could use the same method through Netcat to attempt hash cracking, but since we are manipulating a TGS-REP hash this time, use `-m 13100`.

![image](/THM/RazorBlack/RazorBlack_images/17.png)

The user’s password has successfully been cracked, allowing to retrieve it in plaintext, again.

We could get gack in BloodHound in order to inspect what configurations within the AD domain does our newly compromised user allow him to make:

![image](/THM/RazorBlack/RazorBlack_images/18.png)

It appears that the xyan1d3 domain user is a member of the ***Remote Management Users*** group. Similarly to the *Remote Desktop Users* group with RDP, this group allows a user to access remotely the system through the WinRM protocol (Windows Remote Management). We could leverage this group membership to obtain our initial access over the target system.

Hence, we could attempt to use `evil-winrm` to legitimately authenticate through WinRM as the xyan1d3 user as below:

![image](/THM/RazorBlack/RazorBlack_images/19.png)

As expected, we successfully obtained our initial foothold on the target system as the xyan1d3 user through WinRM by using its recovered password, but the first flag is not granted yet. Inspecting user’s directory, I could identify an intriguing `xyan1d3.xml` XML file that was taunting me:

![image](/THM/RazorBlack/RazorBlack_images/20.png)

The strings contained in the `<Password>` tag present in the file don't seem to correspond to anything; I think we're just being trolled here.

Then, I remembered that the user we had just compromised was a member of a group other than “*Domain Users*” and “*Remote Management Users*”:

 the “***Backup Operators***” group.

![image](/THM/RazorBlack/RazorBlack_images/21.png)

Backup Operators is a built-in Windows group which has privileges that allows you to perform backups and restores, including files that the user would not normally have access to, such as `SYSTEM` and `SAM` Windows **registry hives**, located in the restricted `C:\Windows\system32` system directory. 

An attacker who has compromised a user belonging to this group, or who possesses the `SeBackupPrivileges` and `SeRestorePrivileges` local privileges on the machine, could, as we are about to do here, use the **BootKey** (also known as SysKey) stored in the `SYSTEM` hive to decrypt the data present in the `SAM` hive, namely the password hashes of the machine’s local users (not to be confused with `NTDS.dit`, which stores all the **domain’s secrets**; its compromise may therefore be significantly **more critical**). We could then perform a Pass-the-Hash attack in order to authenticate as the `Administrator` local user. These effective privileges can be confirmed through `whoami /priv`:

![image](/THM/RazorBlack/RazorBlack_images/2.png)

Hence, the following command could be executed to copy `SYSTEM` and `SAM` local hives in our current folder:

`reg save HKLM\SAM SAM`

`reg save HKLM\SYSTEM SYSTEM`

![image](/THM/RazorBlack/RazorBlack_images/23.png)

Now, we have to set up a SMB server in order to transfer the local registry hives from the target system to our local machine. This can be achieved through `impacket-smbserver` as below:

![image](/THM/RazorBlack/RazorBlack_images/24.png)

 Let’s transfer both SYSTEM and SAM hives through the `copy` command (from the target system):

![image](/THM/RazorBlack/RazorBlack_images/25.png)

![image](/THM/RazorBlack/RazorBlack_images/26.png)

Once both files successfully transfered to our local machine, we have the key (SYSTEM) and the chest (SAM), that’s great, but how do we extract any user hashes ?
The SAM decryption could be achieved, again through an Impacket tool. We could pass both registry hives to `impacket-secretsdump` in order to dump password hashes of all local users, including **Administrator’s hash**:

![image](/THM/RazorBlack/RazorBlack_images/27.png)

SAM’s hive decryption and recovery has been successful, allowing us to retrieve the NTLM hash of the `Administrator` user. 

At this point, the stronger password would not resist to `evil-winrm`: the tool allows to lead a **Pass-the-Hash** attack through the `-H` switch, which means that we don’t have to worry about password cracking, we could directly use Administrator NT hash to authenticate with the following command:

`evil-winrm -i TARGET_IP -u Administrator -H <NT_HASH>`

![image](/THM/RazorBlack/RazorBlack_images/28.png)

We successfully escalated our privileges to the `Administrator` Domain Admin, on the Domain Controller. This concludes in a total access and control of the domain and its entities. This can be verified through the `whoami /priv`, `whoami /groups` and `hostname` commands.

![image](/THM/RazorBlack/RazorBlack_images/29.png)

![image](/THM/RazorBlack/RazorBlack_images/30.png)

![image](/THM/RazorBlack/RazorBlack_images/31.png)

The root flag can be claimed in the `C:\Users\Administrator` directory in the `root.xml` file, but it’s hex-encoded.

![image](/THM/RazorBlack/RazorBlack_images/32.png)

Pasting it into CyberChef may be the faster you can do:

![image](/THM/RazorBlack/RazorBlack_images/33.png)

## Thank you for reading!
