## Hello everyone!


Today, we will explore “*Underpass*”, an easy-level machine from **HackTheBox**

Without further ado, let’s get started!

![image](/HTB/UnderPass/Underpass_images/1.png)

(Ensure that you have added the following line  `TARGET_IP     underpass.htb`  to your `/etc/hosts` local file in order to resolve the domain name through DNS)



## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/UnderPass/Underpass_images/2.png)

We can identify two running services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (Apache httpd 2.4.52 | 80)

We can start by inspecting the web site hosted on port **80**:

![image](/HTB/UnderPass/Underpass_images/3.png)

We face a Ubuntu’s Apache2 default configuration page, which contains anything that could help us, either in the source code. We can use **Gobuster** to enumerate potential hidden web directories:

![image](/HTB/UnderPass/Underpass_images/4.png)

It didn’t reveal anything we could access directly. Enumerating potential subdomains/virtual hosts isn’t concluent either. Maybe we missed a UDP service in our Nmap scan that could reveals us something interesting, let’s get back to it:

![image](/HTB/UnderPass/Underpass_images/5.png)

It seems that **SNMP** is running on the target too! We can use this protocol to enumerate further information. `snmpwalk` would be a very helpful tool for us here:

![image](/HTB/UnderPass/Underpass_images/6.png)

There is a lot of information here. Most importantlywe discoevered an email address and an application named “*daloradius*” installed on the webserver. What if we try to access this endpoint in our web browser?

![image](/HTB/UnderPass/Underpass_images/7.png)

It exists! However, we cannot access it. After further enumeration of the directory, we can locate a login panel on the `/daloradius/app/operators/login.php endpoint`:

![image](/HTB/UnderPass/Underpass_images/8.png)

We can identify the application’s version which is **2.2 beta**. A quick search revealed that **daloRADIUS** often uses default credentials `administrator:radius`.

![image](/HTB/UnderPass/Underpass_images/9.png)

We can attempt to use it to authenticate us:

![image](/HTB/UnderPass/Underpass_images/10.png)

![image](/HTB/UnderPass/Underpass_images/11.png)

It works! The defaults credentials were not remplaced, which is critical from a security perspective. We can now access the dashboard storing restricted resources, such as users information or configuration files. 




## Initial Access




After moving around, we can identify a password’s MD5 hash for the user **svcMosh**:

![image](/HTB/UnderPass/Underpass_images/12.png)

Since MD5 is very weak, let’s paste the hash in a `hash.txt` file and attempt to crack it offline using **John The Ripper:**

![image](/HTB/UnderPass/Underpass_images/13.png)

Wonderful, we now have a pair of valid credentials:

`svcMosh:underwaterfriends`

What happens if we use it to authenticate through **SSH**? Does the user exists on the system?

![image](/HTB/UnderPass/Underpass_images/14.png)

The user account exists on the system. We successfully gained initial access on the target as the user **svcMosh**.

We can now list our current directory and retrieve the first flag:

![image](/HTB/UnderPass/Underpass_images/15.png)





## Privilege Escalation



We now need to escalate our current privileges in order to retrieve the final flag. We can start to check privilege escalation vectors using `sudo -l` to see whether we are able to execute any command with **root privileges**:

![image](/HTB/UnderPass/Underpass_images/16.png)

As we can see, our current user can run the following command as **root** without providing a password: `/usr/bin/mosh-server`.

It would be a good idea to execute it to learn more about the usage of the binary:

![image](/HTB/UnderPass/Underpass_images/17.png)

The binary appears to initiate a mosh server instance, providing us a token-key which we need to use in order to access the session as **root**.

We can run the following command to initiate a new one:

`sudo /usr/bin/mosh-server 0.0.0.0`

We will then export the key to the MOSH_KEY environment variable using `export MOSH_KEY=<TOKEN>`

![image](/HTB/UnderPass/Underpass_images/18.png)

After we set up the token variable, we can access the session running internally on port 60001 using the following command:

`mosh-client 127.0.0.1 60001`

![image](/HTB/UnderPass/Underpass_images/19.png)

We successfully access to the session initiated with root privileges and obtained our r**oot shell**! We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/UnderPass/Underpass_images/20.png)

Challenge completed!

![image](/HTB/UnderPass/Underpass_images/21.png)




## Conclusion



The **UnderPass** machine is a great example of how multiple small misconfigurations can chain together into a full system compromise.

We started by performing basic reconnaissance, which revealed only standard services such as SSH and HTTP. However, further enumeration of UDP services led us to **SNMP**, which proved to be the key entry point. Through SNMP enumeration, we were able to discover critical information about the internal environment, including the presence of a **daloRADIUS application** running on the web server.

This discovery allowed us to access a hidden administration panel, where default credentials were still in use. This highlights a common but severe security issue: **failure to change default credentials in production systems**. From there, we obtained access to sensitive data, including a hashed password, which we successfully cracked using **John the Ripper**, granting us SSH access as the user **svcMosh**.

Finally, privilege escalation was achieved by abusing a misconfigured `sudo` permission on `/usr/bin/mosh-server`. By leveraging this binary, we were able to spawn a session running with root privileges and ultimately obtain full control of the system.

This machine emphasizes several important security lessons:

- Never expose SNMP without proper access restrictions
- Always change default credentials in web applications
- Protect sensitive dashboards and internal services from unauthenticated access
- Carefully restrict `sudo` permissions, especially for binaries capable of executing commands or spawning sessions

Overall, **UnderPass** demonstrates how a combination of enumeration, weak credentials, and sudo misconfiguration can lead to full system compromise.
