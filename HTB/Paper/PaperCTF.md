## Hello everyone!

Today, we will explore *Paper*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Paper/Paper_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     paper.htb`   to your local `/etc/hosts` file)





## Information Gathering



First, an aggressive **Nmap** scan can be performed to obtain comprehensive results about the target:

![image](/HTB/Paper/Paper_images/2.png)

We can identify three exposed services:

**SSH** (OpenSSH 8.0 | 22)

**HTTP** (Apache httpd 2.4.37 | 80)

**HTTPS** (Apache httpd 2.4.37 | 443)

The hosted website on port HTTP/HTTPS appears to be the same.

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Paper/Paper_images/3.png)

The web root page is presented as a default static webpage. Enumerating web endpoints with **Gobusted** isn’t conclusive, we are only able to find a `/manual` folder informing about Apache configuration. Inspecting the response header with `curl`, we can identify an existing domain: `office.paper`.

![image](/HTB/Paper/Paper_images/4.png)

It may contain something interesting. Let’s add it to our `/etc/hosts` file in order to enable DNS resolution:

![image](/HTB/Paper/Paper_images/5.png)

We should now be able to access the domain:

![image](/HTB/Paper/Paper_images/6.png)

**Wappalyzer** indicates us that there is a WordPress CMS running on the domain. Inspecting the source code, we can identify its version, which is **5.2.3**:

![image](/HTB/Paper/Paper_images/7.png)

We can perform a quick google research for potential vulnerabilities affecting the targeted application:

![image](/HTB/Paper/Paper_images/8.png)

We discovered an information disclosure in this WordPress version allowing access to private/draft content using specific GET parameters. This weakness is also referenced as **CVE-2019-17671**. In my case, `order=asc` didn’t work, i had to use `order=desc` instead and it worked:

![image](/HTB/Paper/Paper_images/9.png)

This allows us to see private users information. It appears that there is a secret registration link for new employee at the following URL: `http://chat.office.paper/register/8qozr226AhkCHZdyY`.

To access it, we again need to enable DNS resolution of the `chat.office.paper` subdomain by adding it to our `/etc/hosts` file. Once it is done, we can access the secret link to register on the chat system:

![image](/HTB/Paper/Paper_images/10.png)

We proceed to create an account for the **rocket.chat** web application. I registered with the following credentials:

`Name: testuser`

`Email: testuser@mail.com`

`Password = P@$$w0rd`

After a few second, a **general** chat channel spawn to us. We can learn from it that a recyclops bot has been configured and can execute some commands:

![image](/HTB/Paper/Paper_images/11.png)

Sending the `help` direct message to the **recyclops** bot reveals what the bot cat actually do. For example, we can list the current directory as the `ls` command would do:

![image](/HTB/Paper/Paper_images/12.png)




## Initial Access


There is also a way to view the content of a file with providing `file` to the bot. Can we for example retrieve the `/etc/passwd` file content?

![image](/HTB/Paper/Paper_images/13.png)

It appears that we are located in the `/home/dwight/sale` directory. Since the command works exclusively on this directory, what happens if we attempt to perform Path Traversal using `../../../../` to mount up the file system tree?

![image](/HTB/Paper/Paper_images/14.png)

The bot command was restricted to a base directory, but path traversal allowed escaping this restriction. We can identify two non-root users present on the system: **rocketchat** and **dwight**. Since there is a kind of Local File Inclusion vulnerability present on the target, we can read arbitrary files on the system. Inspecting thoroughly the file tree, we can identify an interest .env file in the `/home/dwight/hubot` directory. It can be accessed by using `file ../hubot/.env`:

![image](/HTB/Paper/Paper_images/15.png)

We can retrieve the cleartext password for the rocketchat userbot **recyclops**:`Queenofblad3s!23`.

Since we found this password in the `/home/dwight` directory, we can assume that the password has may been reused over SSH for this user. Let’s attempt to authenticate via SSH using the following credentials: `dwight:Queenofblad3s!23`:

![image](/HTB/Paper/Paper_images/16.png)

It works! We observed a password leak in configuration files, which was successfully reused for SSH authentication, granting initial access. Listing the current directory allows us to retrieve the first flag:

 

![image](/HTB/Paper/Paper_images/17.png)





## Privilege Escalation



We now need to elevate our current privileges to retrieve the final flag. It is a good practice to start enumerating privilege escalation vectors by running `sudo -l` when a password has been obtained. It allows to see whether the user we just compromised is able to run any command with **root privileges**:

![image](/HTB/Paper/Paper_images/18.png)

But here, it seems that we can’t do this. Checking cron jobs, capabilities and SUID doesn’t reveal anything interesting. However, we can identify with the `rpm -qa polkit` a vulnerable version affecting the **polkit** component installed on the target system:

![image](/HTB/Paper/Paper_images/19.png)

The 0-115.6 polkit version is vulnerable to **CVE-2021-3560** which leverage a present bug in the program. The vulnerability relies on a race condition in polkit authentication handling, allowing local privilege escalation under specific interaction timing. This allows an attacker to elevate its privileges to **root**. We can use the **Python** script hosted on this GitHub Repository to escalate access. We need to download it and transfer it to the target in a accessible directory, for instance `/tmp`:

![image](/HTB/Paper/Paper_images/20.png)

We can then execute it to exploit the polkit vulnerability:

![image](/HTB/Paper/Paper_images/21.png)

We successfully escalated our current privileges to **root**, granting full access and control over the target machine. We’re now able to navigate to the `/root` directory in order to retrieve the last flag:

![image](/HTB/Paper/Paper_images/22.png)

Challenge completed!

![image](/HTB/Paper/Paper_images/23.png)




## Conclusion

The compromise resulted from chaining information disclosure, path traversal, credential reuse, and a local privilege escalation in polkit. We started by enumerating a standard domain that reveals a subdomain in its `X-Backend-Server` **response header**, which was hosting a WordPress application. The WordPress installation had a known vulnerability allowing to view unauthorized content by inputting `static` and `order` GET parameters. We found a secret link reserved to employees allowing to create an account for a chat platform. We identified a bot user allowing to perform system command. By abusing a Path Traversal vulnerability, we retrieved the cleartext password for the chatbot user for the **RocketChat** web application that has been reused through the SSH configuration of the **dwight** user. Finally, we identified a vulnerable **polkit** version installed on the target machine that allowed us to escalate our privileges to **root** to obtain full access over the machine.


**Thank you for reading!**
