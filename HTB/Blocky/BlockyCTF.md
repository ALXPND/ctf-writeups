**Welcome everyone !**

This is my third write-up, today we will cover “Blocky” from **HackTheBox.**

Let’s start !

![image](/HTB/Blocky/Blocky_images/1.png)

## Information Gathering

We can start with an **Nmap** scan with service detection:

![image](/HTB/Blocky/Blocky_images/2.png)

There are three running services:

**FTP (21)**

**SSH (22)**

**HTTP (80)**

Curiously, we can’t determine the FTP version. Using the **ftp_version** module from **Metasploit** doesn’t help either:

![image](/HTB/Blocky/Blocky_images/3.png)

We can try to interact with the FTP server without credential if **anonymous login** is enabled on the FTP server: 

![image](/HTB/Blocky/Blocky_images/4.png)

Well, i had to wait a bit before getting prompted to enter any credentials. Although I initially thought it was broken, my impatience was in this case useful, because we got the version for the FTP server when I pressed CTRL+C multiple times. Hence, our problem is resolved. Now, let’s not interrupt the process and try to access this FTP server anonymously:

![image](/HTB/Blocky/Blocky_images/5.png)

Anonymous access seems to be disabled. We will keep the version aside for now and investigate the website:

![image](/HTB/Blocky/Blocky_images/6.png)

We are dealing with a blog under construction, We can do further enumeration with **ffuf**:

![image](/HTB/Blocky/Blocky_images/7.png)

 

It looks like we’re dealing with a WordPress website. A very useful tool for enumerating this CMS is `wpscan`, a powerful program that can help us during the enumeration phase. Let’s enumerate potentials users:

![image](/HTB/Blocky/Blocky_images/8.png)

We identified an user : notch 

Attempting to brute force the wp-admin page as notch doesn’t give anything.

Let’s continue to enumerate the website. We can check the `/plugins` directory:

![image](/HTB/Blocky/Blocky_images/9.png)

We have two .jar files, which we can download and decompile using the `jd-gui`Linux utility. We will do this with the `BlockyCore.jar`file:

![image](/HTB/Blocky/Blocky_images/10.png)

## Initial Access

We found useful informations ! There is a username and a password we could use to interact with a SQL database. But since there is no MySQL listening port on the target or any database service, we can try directly this password to login via SSH as the user we found, **notch**:

![image](/HTB/Blocky/Blocky_images/11.png)

It works! We successfully gained initial access to the target. Although the password we obtained was intended for local SQL access, it was **reused** for SSH access, illustrating a common **password reuse issue**. We can list the content of the notch’s directory, leading to the first flag:

![image](/HTB/Blocky/Blocky_images/12.png)

## Post-Exploitation / Privilege Escalation

In order to get the `root.txt`  flag, we need to elevate our privileges. First, let’s do some local enumeration of our current privileges, we can check what command we can run with root privileges as notch using `sudo -l` :

![image](/HTB/Blocky/Blocky_images/13.png)

And it’s done… the post-exploitation phase is free as we can run everything we want as root via sudo without providing any credential, this is a critical misconfiguration. We can simply run the command `sudo su` to gain **root access.** Therefore we can fetch the `root.txt`flag:

![image](/HTB/Blocky/Blocky_images/14.png)

## Conclusion

Nothing much more to say. The privesc was free as `sudo -l` is usually the first thing we check after authenticating as a legitimate user. However, the initial access was interesting, we had to discover and decompile a .jar file hosting credentials for a database running locally on the machine.

Thank you for reading !
