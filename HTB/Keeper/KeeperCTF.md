## Hello everyone!

Today, we will explore “*Keeper*”, an easy-level machine released by HackTheBox.

Without further ado, let’s get started!

![image](/HTB/Keeper/Keeper_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP     keeper.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Keeper/Keeper_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (nginx 1.18.0 | 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Keeper/Keeper_images/3.png)

The website prompts us to visit a `tickets.keeper.htb` subdomain in order to raose an IT support ticket. In order to access the subdomain, we will need to add it to our `/etc/hosts` file to perform DNS resolution:

![image](/HTB/Keeper/Keeper_images/4.png)

This allows us to access the subdomain:

![image](/HTB/Keeper/Keeper_images/5.png)

We are redirected to a **Request Tracker CMS** login panel. Researching default credentials for this application reveals the following pattern:

`root : password`

![image](/HTB/Keeper/Keeper_images/6.png)

We can try to use them to authenticate:

![image](/HTB/Keeper/Keeper_images/7.png)

![image](/HTB/Keeper/Keeper_images/8.png)

Wonderful! The default credentials were not changed, allowing us to access the restricted dashboard as the **root** user.

Inspecting the current tickets, we can identify another user present on the system: `lnorgaard` 

![image](/HTB/Keeper/Keeper_images/9.png)

## Initial Access

Navigating to the user’s settings for this particular user reveals us something interesting:

![image](/HTB/Keeper/Keeper_images/10.png)

This user seems to be new on the system and has as default password `Welcome2023!`.

Since this is a default password, It may have been reused through the **SSH** configuration of the user. Let’s verify it by authenticating via SSH using the following credential:

`lnorgaard : Welcome2023!`

![image](/HTB/Keeper/Keeper_images/11.png)

It works! The SSH default password for the user `lnorgaard`  has **not been changed**, allowing us to obtain **initial access** on the target machine. We can list the current directory and retrieve the user flag:

![image](/HTB/Keeper/Keeper_images/12.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag. We can identify an interesting `.zip` file in our directory. Let’s extract it to view its content:

![image](/HTB/Keeper/Keeper_images/13.png)

It appears that the target system uses KeePass to manage users passwords. If we find a way to read the database, we could find a way to retrieve sensitive information. All passwords are stored into a single file protected by a master password.

There is a known vulnerability allowing an attacker access to the database's master password from a memory dump: **CVE-2023-32784**. To leverage it, we first need to transfer the `KeePassDumpFull.dmp` dump file to our local machine:

![image](/HTB/Keeper/Keeper_images/14.png)

We can then use the Python script located in this GitHub Repository:

![image](/HTB/Keeper/Keeper_images/15.png)

It looks like the master password contains broken characters. Pasting the output in our browser reveals us a Danish dessert called `rødgrød med fløde`:

![image](/HTB/Keeper/Keeper_images/16.png)

The master password may be `rødgrød med fløde`. 

We now need to transfer the `passcodes.kdbx` file to our local machine.

![image](/HTB/Keeper/Keeper_images/17.png)

We then open KeepassXC running the `keepassxc` command. We click on *Open database* and we select the file we just transferred. We are then prompted to enter the master password for KeePass, we will enter the following : `rødgrød med fløde`.

![image](/HTB/Keeper/Keeper_images/18.png)

![image](/HTB/Keeper/Keeper_images/19.png)

It works! We successfully accessed the KeePass manager using the master password.

The access reveals a `.ppk` RSA private key owned by the **root** user:

![image](/HTB/Keeper/Keeper_images/20.png)

We can also identify a password which could represent the passphrase for the key, we will keep it aside. For now, we will paste the entire key into a `key.ppk` file and convert the key in order to authenticate using `puttygen`:

![image](/HTB/Keeper/Keeper_images/21.png)

We then set the required permissions and use it via SSH to gain access on the system as the **root** user:

![image](/HTB/Keeper/Keeper_images/22.png)

We successfully escalate our initial access to **root**, granting us full access on the target system! We then navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Keeper/Keeper_images/23.png)

Challenge completed!

![image](/HTB/Keeper/Keeper_images/24.png)




## Conclusion


Through this machine, we explored a realistic attack chain built on common misconfigurations and poor credential hygiene. Starting from basic reconnaissance, we quickly identified exposed services and leveraged default credentials on a misconfigured Request Tracker instance to gain initial access.

The challenge highlighted the risks of password reuse, allowing us to pivot from web access to SSH using weak default credentials. During privilege escalation, we encountered a KeePass database, which introduced an interesting twist by requiring us to exploit a memory dump vulnerability (CVE-2023-32784) to recover the master password.

Finally, by extracting sensitive data from the password manager, we obtained a private SSH key that led us to full system compromise as root.

Overall, this machine emphasizes the importance of:

- Changing default credentials
- Enforcing strong password policies
- Securing sensitive files such as memory dumps and password databases
- Avoiding credential reuse across services

*Keeper* is a great example of how small security oversights can be chained together to achieve complete system compromise.

**Thank you for reading!**
