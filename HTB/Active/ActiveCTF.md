**Welcome everyone !**

Today, we will explore the “Active” machine from **HackTheBox**. This CTF is rated easy and is well-rated by the community.

![image](/HTB/Active/Active_images/1.png)

Please take note that i added `10.129.36.19    active.htb` in the `/etc/hosts` in order to do a DNS resolution of the host. If you see this domain name in my commands, it only represents the target IP itself. It is a good practice to do this as we can follow when needed with a subdomains / virtual hosts discovery.

## Information Gathering

As always, we start with an **Nmap** scan:

![image](/HTB/Active/Active_images/2.png)

We can identify a **domain** which is named `active.htb` and various services. The fundamentals here are:

**DNS (53)**

**Kerneros (88)**

**RPC (135)**

**NetBios (139)**

**LDAP (389)**

**SMB (445)**

We can deduce that we’re dealing with a Windows machine within an Active Directory environment. Our task will probably consist to compromise the domain, escalating from an user to another.

Let’s enumerate the SMB shares with the `smbclient` Linux utility:

![image](/HTB/Active/Active_images/3.png)

We found seven shares. After testing them, it seems that we are able to access to the `Replication` share without credential since **anonymous login** is enabled on this share:

![image](/HTB/Active/Active_images/4.png)

There is an `active.htb` directory. After inspecting the various directories in it, we can found an interesting XML file:

![image](/HTB/Active/Active_images/5.png)

We can download the file on our machine with `get Groups.xml` and inspect it locally:

![image](/HTB/Active/Active_images/6.png)

## Initial Access

There is an interesting `cpassword` field with the following value: `edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`. 

`cpassword` values are not hashes. They are **AES-encrypted** passwords used by **Group Policy Preferences** (GPP). The issue is that Microsoft published the AES key used for encryption, meaning anyone with access to the XML file can **decrypt** the password. We can use `gpp-decrypt`, a preinstalled utility on Kali Linux, to recover the plaintext password:

![image](/HTB/Active/Active_images/7.png)

It works! We successfully decrypted the `GPPstillStandingStrong2k18` password.

We are encouraged to use this password with the service account **SVC_TGS**.

## Privilege Escalation

Instead of using it directly, we can leverage this account to perform a **Kerberoasting attack**.

Kerberoasting targets accounts that have a **Service Principal Name** (SPN). Any authenticated domain user can request a Kerberos service ticket (TGS) for these accounts from the **KDC**.

The important point is that these service tickets are encrypted using material derived from the target account’s password. This allows attackers to request tickets and crack them offline in order to recover plaintext credentials.

We can enumerate SPNs and request service tickets using `GetUserSPNs.py` from the Impacket suite:

![image](/HTB/Active/Active_images/8.png)

We successfully got the response from the KDC which represent the hash for the user **Administrator** ! As we authenticated legitimately as **SVC_TGS**, we obtained a **Kerberos service ticket** (TGS) for a service associated with the Administrator account, encrypted using its NTLM hash.

We will use **hashcat** with the mode 13100 (Kerberos 5, etype 23, TGS-REP) on Hashcat which correspond to the current hash. We paste the hash in a `hash.txt` file and run the following command:

`hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Active/Active_images/9.png)

We successfully retrieve the password:

`Ticketmaster1968`

 Let’s use `psexec.py` to authenticate as the user **Administrator:**

![image](/HTB/Active/Active_images/10.png)

We successfully gained access to the target machine with **highest privileges**!

We can now retrieve both flags:

![image](/HTB/Active/Active_images/11.png)

After gaining **SYSTEM** access, we used standard enumeration techniques to locate the user flag on the system. We can use the **Meterpreter** `search` built-in command:

![image](/HTB/Active/Active_images/12.png)

## Conclusion

Today, we exploited an Active Directory environment by leveraging a combination of misconfigurations and Kerberos weaknesses.

Starting from a low-privileged domain account, we were able to perform Kerberoasting against service accounts with SPNs, allowing us to recover and crack a service ticket offline.

This demonstrates how privilege escalation in Active Directory environments often relies on chaining multiple small misconfigurations rather than a single vulnerability.

Thank you for reading !
