## Hello everyone!

Today, we will explore *Popcorn*, a medium-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Popcorn/Popcorn_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     popcorn.htb`   to your local `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we will perform an aggressive nmap scan to gather as much information as possible about the target:

![image](/HTB/Popcorn/Popcorn_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 5.1p1 | 22)

**HTTP** (Apache httpd 2.2.12 (Ubuntu) | 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Popcorn/Popcorn_images/3.png)

The root webpage is a static webpage which doesn’t contain anything, either in the source code. We can use **Gobuster** to enumerate available web directories:

![image](/HTB/Popcorn/Popcorn_images/4.png)

We can identify three interesting endpoints: `test`, `test.php` and `torrent`. The first two confirms that **PHP** is deployed. The last one remains mysterious. Let’s inspect it:

![image](/HTB/Popcorn/Popcorn_images/5.png)

It appears that the web server hosts a **Torrent Hoster** application. We can authenticate through a `login.php` API endpoint. Attempting a couple of default credentials combination such as `admin:admin` does not work. But what happens if we attempt to inject a classic SQL Injection bypass payload such as `admin' OR '1' = '1 --` in the *username* field?

![image](/HTB/Popcorn/Popcorn_images/6.png)

![image](/HTB/Popcorn/Popcorn_images/7.png)

We successfully bypassed the authentication mechanism using a classic SQL injection payload.




## Initial Access


After a thorough enumeration of the dashboard, i couldn’t identify an entrypoint to gain initial foothold on the target system. Researching about potential exploit affecting this application led me to a Python script hosted in this GitHub Repository which allows to achieve **remote code execution** (RCE).

We can download and execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/Popcorn/Popcorn_images/8.png)

It appears that the only need to provide the target URL to the script. Moreover, we need to execute it via `python2` to make it functional:

![image](/HTB/Popcorn/Popcorn_images/9.png)

We successfully obtained initial access on the target machine as the **www-data** user.

Listing the `/home` directory reveals a **george** user. However, we can’t access this directory because of the shell instability. Furthermore, we really need to upgrade our current shell in order to execute system command properly. Executing classic `python -c 'import pty; pty.spawn("/bin/bash")'` or `script -qc /bin/bash` does not work. Moreover, `Busybox` is not installed on the target, so we cant initiate a reverse shell from the target.

After trying a couple of techniques, I decided to use a PHP reverse shell from PentestMonkey.

We can transfer it to the target through a Python server we could set up locally on our machine using `python3 -m http.server 8000`. The file can then be downloaded from the target with the `wget` utility:

![image](/HTB/Popcorn/Popcorn_images/10.png)

(You need to configure the PHP file to your respective own IP and assigned port for your netcat listener. Here, i chose the port 4444)

The file has successfully been uploaded. Since we downloaded it from an accessible `/upload` directory, we can access it via the following URL to execute it:

`http://popcorn.htb/torrent/upload/<REVSHELL_FILE>`

![image](/HTB/Popcorn/Popcorn_images/11.png)

The request should hang in your web browser, resulting in a successfull initiation of the reverse shell!

We are now able to run `python -c 'import pty; pty.spawn("/bin/bash")'` in order to stabilize the shell without causing it to crash as before.

The `/home/george` directory can now be accessed, allowing us to retrieve the user flag:

![image](/HTB/Popcorn/Popcorn_images/12.png)





## Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag.

The target is vulnerable to **CVE-2010-0832** which leverage a vulnerability in the Message Of The Day (MOTD).

`pam_motd` (aka the MOTD module) in `libpam-modules` before **1.1.0-2ubuntu1.1** in **PAM** on `Ubuntu 9.10` and `libpam-modules` before **1.1.1-2ubuntu5** in PAM on **Ubuntu 10.04 LTS** allows local users to change the **ownership of arbitrary files** via a symlink attack on `.cache` in a user's home directory, related to "user file stamps" and the `motd.legal-notice` file.

The vulnerability is triggered during user login via PAM (e.g. SSH session authentication), which executes the MOTD scripts. But we can’t trigger it from our webshell. We need to create an SSH key pair and deploy the public key into `authorized_keys` for the **www-data** user to get access through a legitimate authentication in order to activate the MOTD.

We will simply create a private key using `ssh-keygen -t rsa -b 2048` from our local machine (with a passphrase of your choice). This will create a private key and a public key. Then, we need to transfer the public key in the target `/.ssh` directory as a file named `authorized_keys`:

![image](/HTB/Popcorn/Popcorn_images/13.png)

We also need to give the following permissions to make it functional:

`chmod 700 .ssh` (from the target)

`chmod 600 .ssh/authorized_keys` (from the target)

`chmod 600 id_rsa` (from our local machine)

Finally, we can authenticate using the following command:

`ssh www-data@popcorn.htb -i key -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -o KexAlgorithms=+diffie-hellman-group1-sha1`

![image](/HTB/Popcorn/Popcorn_images/14.png)

SSH authentication is now possible to trigger the **MOTD**. We can use the Bash script hosted in this GitHub Repository to leverage the vulnerability. We will transfer it in an accessible directory, such as `/tmp` and give it execute permission to use it:

![image](/HTB/Popcorn/Popcorn_images/15.png)

We need to specify to the script the file we want to target to gain permission. The most interesting file we want to modify is the `/etc/passwd` because it could allow us to add a privileged user. Consequently, we could be able to authenticate as root without password, gaining full access on the machine. Let’s execute the following command:

`./14273.sh /etc/passwd`

![image](/HTB/Popcorn/Popcorn_images/16.png)

The script has successfully been executed. We now need to use the **SSH access** we created to trigger the vulnerable **MOTD** through **PAM**. Please observe carefully the `/etc/passwd` file permissions before we log back: this is owned by **root**, as it should be. However, after a second SSH authentication:

![image](/HTB/Popcorn/Popcorn_images/17.png)

File permissions changed! We now have write permission on it.

This means that we can edit the file with `nano` in order to add a new privileged user with UID 0 via modified password hash entry in `/etc/passwd`.

To do this, we will first create a valid password with the following `openssl` command:

`openssl passwd -6 -salt abc <PASSWORD>`

![image](/HTB/Popcorn/Popcorn_images/18.png)

Then, we will add a new line in the `/etc/passwd` file by pasting the **root configuration line**, altering the username and replacing the `x` character by the hash we just generated. The following line should look as follow:

![image](/HTB/Popcorn/Popcorn_images/19.png)

Finally, we can authenticate to our newly generated user by running `su <USER>` and providing the password chosen:

![image](/HTB/Popcorn/Popcorn_images/20.png)

We successfully escalated our privileges to **root**, granting full access and control over the machine. We can now navigate to the `/root` directory in order to retrieve the final flag:

![image](/HTB/Popcorn/Popcorn_images/21.png)

Challenge completed!

![image](/HTB/Popcorn/Popcorn_images/22.png)

## Conclusion

This challenge was very pleasant to complete. We started from a web enumeration that revealed an outdated **Torrent Hoster** web application deployed, allowing us to execute arbitrary system commands to obtain initial foothold as **www-data**.

During the privilege escalation phase, we identified that the target system was running an **outdated Ubuntu 9.10 environment** with a vulnerable PAM configuration related to **CVE-2010-0832 (MOTD file tampering)**.

The exploitation relied on abusing a **symlink race condition in the PAM MOTD process**, where a `.cache` symbolic link in the user’s home directory could be hijacked to redirect file operations to sensitive system files. By preparing the environment and triggering a new authentication session, it became possible to influence file ownership changes during the MOTD execution.

This allowed us to modify a root-owned file and ultimately escalate privileges by injecting a new privileged user into `/etc/passwd`, leading to full root access.


**Thank you for reading!**
