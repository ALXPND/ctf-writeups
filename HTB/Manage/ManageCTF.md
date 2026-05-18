## Hello everyone!

Today, we will cover the “Manage” machine from HackTheBox.

Without further ado, let’s get started!

![image](/HTB/Manage/Manage_images/1.png)

(It is a good practice to add the following line  `TARGET_IP    manage.htb` to the `/etc/hosts` file, replacing `TARGET_IP` with the target machine IP address)

## Information Gathering

We can start with an aggressive Nmap scan to obtain comprehensive results:

![image](/HTB/Manage/Manage_images/2.png)

We can identify three services running on the target:

**SSH** (OpenSSH 8.9p1 | 22)

**RMI** (Java RMI | 2222)

**HTTP** (Apache Tomcat 10.1.19 | 8080)

Let’s visit the website hosted on port **8080**:

![image](/HTB/Manage/Manage_images/3.png)

It appears to be a default Tomcat documentation page. Let’s enumerate the available web directories using **Gobuster**, a powerful web content discovery tool:

![image](/HTB/Manage/Manage_images/4.png)

It seems that we do not have access to the discovered resources.

At this point, there is not much we can do, as we don’t have any authenticated access. But we saw earlier that a Java-RMI service is running on the target. We can use a tool called [BeanShooter](https://github.com/qtc-de/beanshooter) to enumerate resources exposed by this service. 

Here are the installation steps:

1. `git clone https://github.com/qtc-de/beanshooter`
2. `cd beanshooter`
3. `mvn package`

Once this is done, we can enumerate the service with the following command:

`java -jar target/beanshooter-4.1.0-jar-with-dependencies.jar enum 10.129.38.19 2222`

![image](/HTB/Manage/Manage_images/5.png)

We can see at the end of the scan that the tool retrieved valid credentials for two users: **manager** and **admin**

![image](/HTB/Manage/Manage_images/6.png)

There is a way to exploit this service using the “Tonka” functionality built into BeanShooter (you can explore this functionality further if interested), however, it did not work reliably in my testing environment. I rebooted the machine, but the issue persisted.

![image](/HTB/Manage/Manage_images/7.png)

Thankfully, Metasploit provides a `java_jmx_server` module we can use to gain our initial access, even without BeanShooter. So let’s load the module, configure the required options and execute it:

![image](/HTB/Manage/Manage_images/8.png)

We successfully gained access to the target as the user **tomcat**!

We can run the Meterpreter’s built-in `shell` command in order to obtain a Bash shell before performing local enumeration:

![image](/HTB/Manage/Manage_images/9.png)

(We can also run `python3 -c 'import pty; pty.spawn("/bin/bash")'` and `export TERM=xterm` to upgrade/stabilize our shell)

We can identify two additional users present on the system: **karl and useradmin**.

The `/home/karl` directory does not contain anything interesting besides a `.ssh` directory that we are unfortunately not able to access.

However, we discover a `backups` directory inside the **useradmin** home directory containing a `backup.tar.gz` archive:

![image](/HTB/Manage/Manage_images/10.png)

Since extracting the archive locally will be more convenient, we will transfer it to our attacking machine.
After basic enumeration of the root `/` directory, we can retrieve the user flag located in the `/opt/tomcat` directory:

![image](/HTB/Manage/Manage_images/11.png)

## Privilege Escalation

Let’s transfer the archive to our machine in order to extract files. We need to set up a server on the target via `python3 -m http.server 8000`. We can then download the file with the `wget` utility:

![image](/HTB/Manage/Manage_images/12.png)

We can now extract the archive:

![image](/HTB/Manage/Manage_images/13.png)

It looks like the compressed data were simply the /home/useradmin directory. But what interests us here is the `.ssh` directory content, and more specifically the `id_ed25519` file, which appears to be the **SSH private key** for this user. We can also identify a `.google_authenticator` file, indicating that **multi-factor authentication** (MFA) is enabled for this account.

![image](/HTB/Manage/Manage_images/14.png)

(best key ever)

We can try to log in with the private key:

![image](/HTB/Manage/Manage_images/15.png)

We are prompted for a verification code. However, we previously extracted a `.google_authenticator`, a file used by Google Authenticator to store MFA configuration data. Let’s display it:

![image](/HTB/Manage/Manage_images/16.png)

We can identify several emergency codes that may be used for authentication. Since the file belongs to the `useradmin` account, these recovery codes are likely valid for this user. After attempting the first one, we can successfully initiate an SSH session as **useradmin**!

![image](/HTB/Manage/Manage_images/17.png)

## Privilege Escalation (Root)

We can enumerate our current privileges using `sudo -l` to list which commands we can execute with elevated privileges:

![image](/HTB/Manage/Manage_images/18.png)

And it seems that we can run the `adduser` command with root privileges, which can be used to create a new user account on the system. Let’s try to run it and add the **axk8** user:

![image](/HTB/Manage/Manage_images/19.png)

I provided the following password for the user: `password123` (which is definitely not a good security practice).

We can log in to our newly generated account. However, this is not particularly useful since the newly created user only has low privileges. However, there is a specific username that is automatically assigned to a privileged group **by default**.

Interestingly, the `admin` account does not exist on the system.

Because of the machine's custom configuration, creating this specific user automatically assigns it to a privileged group.

![image](/HTB/Manage/Manage_images/20.png)

As we can see, the **admin** user **does not** exist on the system, this means that we can create this specific user and automatically inherit elevated privileges.

![image](/HTB/Manage/Manage_images/21.png)

I chose this time the `FinD@BeTtRP@$$` password (slightly stronger), so I will authenticate with `admin : FinD@BeTtRP@$$`

![image](/HTB/Manage/Manage_images/22.png)

Indeed, full privileges are automatically granted! `sudo -l` reveals that we can run any command on the system as any user, such as the **root** user. So we can simply run `sudo /bin/bash` to spawn a **Bash shell**, but since it is run via sudo, we will obtain a root-privileged shell granting full control over the target system.

![image](/HTB/Manage/Manage_images/23.png)

We have successfully escalated our privileges to **root**.

We can now navigate to the /root directory and retrieve the `root.txt` flag:

![image](/HTB/Manage/Manage_images/24.png)

With that, we have achieved full compromise of the target machine.

![image](/HTB/Manage/Manage_images/25.png)

**Thank you for reading!**
