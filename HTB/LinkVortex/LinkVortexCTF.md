## Welcome everyone!

Today, we will explore *LinkVortex*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/LinkVortex/LinkVortex_images/1.png)

(Please ensure that you have added the following line   TARGET_IP     linkvortex.htb   to your local `/etc/hosts` file in order to enable DNS resolution)





## Information Gathering



First, we will peform an aggressive Nmap scan to obtain as much information as possible about the target:

![image](/HTB/LinkVortex/LinkVortex_images/2.png)

We can identify two exposed services:

**SSH** (22)

**HTTP** (80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/LinkVortex/LinkVortex_images/3.png)

The root page is static and doesn’t contain anything that could help us. All we know is that the web application is owned by a company named *BitByBit Hardware* which provides technology-related information. We can use `ffuf` to enumerate hidden web directories:

![image](/HTB/LinkVortex/LinkVortex_images/4.png)

The web application doesn’t seem to contains anything we can inspect. Maybe this web interface is a decoy and is hidding something else? 

Let’s use `ffuf` again to perform **subdomain enumeration** of the target:

![image](/HTB/LinkVortex/LinkVortex_images/5.png)

We can identify a valid subdomain: `dev.linkvortex.htb`.The subdomain appears to be a web application in development, it may contain interesting content for us. 

We need to add it to our `/etc/hosts` file to access it:

![image](/HTB/LinkVortex/LinkVortex_images/6.png)

We should now be able to reach the subdomain:

![image](/HTB/LinkVortex/LinkVortex_images/7.png)

As expected, we face a web interface that is under construction.

Let’s enumerate it using **Gobuster**:

![image](/HTB/LinkVortex/LinkVortex_images/8.png)

The website appears to host a `.git` folder. This folder usually contains source code for the website concerned. We can use the `git-dumper` to download this folder to our local machine to facilitate enumeration:

(you can install it with pip3 install git-dumper)

![image](/HTB/LinkVortex/LinkVortex_images/9.png)

We now have command-line access to the web folder.

After an in-depth inspection of the various files present in the folder, we can identify a cleartext password that is used through a Ghost application. The file is accessible at `/.git/Ghost/core/test/regression/api/admin/authentication.test.js`:

![image](/HTB/LinkVortex/LinkVortex_images/10.png)

Attempting to access the `/ghost` endpoint does not work on the subdomain, However, we can reach it via the main website at the following URL: `http://linkvortex.htb/ghost`

![image](/HTB/LinkVortex/LinkVortex_images/11.png)

We can now authenticate through a login panel. But wait… we need to provide a valid email address. 

After attempting a couple of email address, we can successfully authenticate using the following credentials:

`admin@linkvortex.htb:OctopiFociPilfer45`

![image](/HTB/LinkVortex/LinkVortex_images/12.png)

![image](/HTB/LinkVortex/LinkVortex_images/13.png)

Access to the **Ghost** dashboard is obtained as **admin**.
Getting back to the root page, we can identify the Ghost version in the source code:

![image](/HTB/LinkVortex/LinkVortex_images/14.png)

The web application is running **Ghost 5.58**. Searching for potential vulnerabilities associated to it leads to **CVE-2023-40028**, a weakness that allows an attacker to perform an authenticated arbitrary file read.




## Initial Access


There is a GitHub repository which hosts a Bash script leveraging the vulnerability affecting the web application. We can download and execute it without providing anything to learn more information about its usage:

![image](/HTB/LinkVortex/LinkVortex_images/15.png)

It appears that we need to provide valid credentials with the target IP . We can execute the following command:

`./CVE-2023-40028 -u admin@linkvortex.htb -p OctopiFociPilfer45 -h http://linkvortex.htb`

![image](/HTB/LinkVortex/LinkVortex_images/16.png)

We successfully exploited the vulnerability. We now have a shell prompting us to enter a file path to read: this is because we achieved an **arbitrary file read**, not **remote code execution**. We can for example provide `/etc/passwd` to verify that the exploit is working:

![image](/HTB/LinkVortex/LinkVortex_images/17.png)

Since we can read local files, We may need to look for an SSH private key or plaintext credentials in configuration files. We can find the Ghost configuration file located at `/var/lib/ghost/config.production.json`

![image](/HTB/LinkVortex/LinkVortex_images/18.png)

There is a cleartext password stored in the file! The password is associated to the user **bob**. Let’s assume that the password may have been reused through SSH, we can attempt to authenticate with the following credentials: `bob:fibber-talented-worth`

![image](/HTB/LinkVortex/LinkVortex_images/19.png)

It works! We successfully obtained initial access on the target system as the user **bob**. Listing the current directory, we are able to retrieve the first flag:

![image](/HTB/LinkVortex/LinkVortex_images/20.png)





## Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag.

It is a good practice to start enumerating local privilege escalation vectors by running `sudo -l` when we have a valid password for our current user. It allows us to see whether we are able to run any command with **root privileges**:

![image](/HTB/LinkVortex/LinkVortex_images/21.png)

It seems that we can run the following command as root without providing any password: `/usr/bin/bash /opt/ghost/clean_symlink.sh *.png`.

Moreover, `env_keep+=CHECK_CONTENT` means that we can pass a environment variable that will be conserved in the sudo environment

First, let’s inspect the Bash script:

![image](/HTB/LinkVortex/LinkVortex_images/22.png)

The script is a security filter for `.png` file. If a file is considered as suspicious, it is redirected in the `/var/quarantined` folder. For instance, if the path contains '**etc**' or '**root**', it will be banned.

the `cat` command follows symlinks (the pointed file). Since we can execute the command with **root privileges**, it will be readed as root.

The trick consists in creating an intermediate symlink that not contains “root”:

`ln -s /root /tmp/r`
 →`/tmp/r` target is the `/root` chain.

We will then dissimulate it into another symlink:

`ln -s /tmp/r/.ssh/id_rsa exploit.png`

→ `exploit.png` target is the `/tmp/r/.ssh/id_rsa` chain, this is what we will provide to the script. As we can see, this chain does not contain “etc” or “root”, so the grep filter will let it pass.

The `CHECK_CONTENT` environment variable is preserved through sudo (`env_keep+=CHECK_CONTENT`) and directly influences the execution flow of the script. When set to `true`, it triggers a **root-executed** `cat` on the quarantined file, which becomes the main primitive exploited in combination with symlinks.

Finally, we can execute the following command to retrieve the SSH private key associated to the **root** user located in the `/root/.ssh/id_rsa` directory:

 `CHECK_CONTENT=true sudo /usr/bin/bash /opt/ghost/clean_symlink.sh exploit.png` 

![image](/HTB/LinkVortex/LinkVortex_images/23.png)

We successfully bypassed the script filtering and retrieved the SSH private key for the **root** user!

We now need to paste it to our local machine in a `id_rsa` file and give it the necessary permissions:

![image](/HTB/LinkVortex/LinkVortex_images/24.png)

We can then authenticate us as **root** providing the SSH private key with the following command:

`ssh -i id_rsa root@linkvortex.htb`

![image](/HTB/LinkVortex/LinkVortex_images/25.png)

Root access is achieved! We successfully escalated our privileges to **root**, granting us full access and control over the target system. We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/LinkVortex/LinkVortex_images/26.png)

Challenge completed!

![image](/HTB/LinkVortex/LinkVortex_images/27.png)

## Conclusion

This CTF demonstrates several common misconfigurations. Starting from a web enumeration, we discovered a subdomain hosted on the target representing a web interface under construction. Due to inadequate security measures, the .`git` folder containing the web source code was exposed. Inspecting it revealed the cleartext password associated to the **admin** user that we leveraged to access the restricted dashboard of a **Ghost** web application. We identified the version, revealing an Arbitrary File Read vulnerability. We were able to retrieve the cleartext password for the user **bob** that has been reused over **SSH**, a very bad security practice. This led us to an initial access over the machine granted, allowing us to enumerate in-depth the target system. We were able execute a Bash script with root privileges via `sudo`. The script was poorly configured, using a `grep` command to filter unallowed strings. We could bypass this mechanism using the power of symlinks. This allowed us to retrieve the SSH private key attached to the superuser that we used to authenticated, leading to full system compromise.


**Thank you for reading!**
