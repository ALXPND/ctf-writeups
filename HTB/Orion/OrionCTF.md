## Hello everyone!




Today, we will explore a newly released machine from **HackTheBox**: *Orion.*

Without further ado, let’s get started!

![image](/HTB/Orion/Orion_images/1.png)




## Information Gathering


First, we will perform an aggressive Nmap scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Orion/Orion_images/2.png)

There are two exposed services:

- SSH (OpenSSH 8.9p1 | 22)
- HTTP (nginx 1.18.0 | 80)

Let’s start by simply inspecting the website hosted on port **80**:

![image](/HTB/Orion/Orion_images/3.png)

By loading the website with **Wappalyzer**, we identified that it was running **Craft CMS**. Running tools like **Gobuster** or **ffuf** doesn’t reveal anything, we can assume that directory listing is disabled:

![image](/HTB/Orion/Orion_images/5.png)

But fortunately, we could identify that **Craft CMS** was running. Since the machine was released recently, we could attempt to research about any recent **vulnerability exposure** affecting this content manager CMS:

![image](/HTB/Orion/Orion_images/6.png)




## Initial Access


It appears that a vulnerability affecting Craft CMS was disclosed last year: **CVE-2025-32432**. If successfully exploited, it allows a threat actor to achieve remote code execution without being authenticated. 

We can use a Python script hosted on this GitHub repository. But there is also a **Metasploit** module (`exploit/linux/http/craftcms_preauth_rce_cve_2025_32432`) that covers the vulnerability. We will use it for the sake of simplicity. After configuring the module correctly with the `LHOST` and `RHOSTS` variables, we can run it:

![image](/HTB/Orion/Orion_images/8.png)

After a couple of seconds, we successfully receive a **Meterpreter session**, granting initial access on the target system as the **www-data** user!

We can now run the Meterpreter built-in `shell` command to spawn a standard Linux terminal in order to start inspecting the local filesystem: 

![image](/HTB/Orion/Orion_images/9.png)

Make sure to execute the commands as above to stabilize the shell, otherwise you won’t be able to execute system commands properly.

We can start our local enumeration by checking the `/home` directory to identfy other users present on the system:

![image](/HTB/Orion/Orion_images/10.png)




## Horizontal Privilege Escalation


We can identify a single non-root user: **adam**. Unfortunately, we cannot access his home directory. This means that we need to perform horizontal privilege escalation to this particular user in order to retrieve the first flag.

Listing the environment variables with the `env` utility, we can identify valid **database credentials** stored in plaintext:

![image](/HTB/Orion/Orion_images/11.png)

This strongly suggests that a database is running locally on the target machine, which could explain why we couldn’t identify it with our previous Nmap scan. Let’s verify it by checking active connections using `ss -tulnp`:

![image](/HTB/Orion/Orion_images/12.png)

Indeed, that is the case! We can deduce from the port that the database is either **MySQL or MariaDB**. We now have all the needed information to access the local database in order to access any user-related table that could store sensitive information. The following command can be executed to authenticate ourselves: `mysql -u root -pSuperSecureCraft123Pass!` (no markspace between the `-p` switch and the password)

After enumerating the `orion` database, as expected, we can identify a `users` table containg the password hash of **adam**:

![image](/HTB/Orion/Orion_images/13.png)

Based on its prefix, we can deduce that hash uses **bcrypt**, which is based on **Blowfish**. Although this algorithm is very slow, we could crack the password if it’s weak. The next logical step is to attempt offline cracking on this hash, hoping that the password has been reused through **SSH**. We can paste it in an `hash.txt` file and crack it using John The Ripper with the following command:

`john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt` 

![image](/HTB/Orion/Orion_images/15.png)

We successfully recovered the password hash associated to **adam**! This allows us to recover the cleartext password `darkangel`, which is not a good security practice. Since the password is weak, there is a reasonable chance that it has been reused for **SSH** authentication. Let’s attempt to authenticate as adam via SSH using the following credentials:

`adam:darkangel`

![image](/HTB/Orion/Orion_images/16.png)

We successfully pivoted our access from **www-data** to **adam** by abusing a weak password that was reused. Listing the current directory allows us to retrieve the user flag:

![image](/HTB/Orion/Orion_images/17.png)




## Vertical Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag. It is a good practice when you have a valid password to start enumerating privilege escalation vectors by running `sudo -l` to see whether the compromised user can execute any command with **root privileges**:

![image](/HTB/Orion/Orion_images/18.png)

Unfortunately, we can’t do this.

Listing cron jobs, capabilities and SUID binaries doesn’t reveal anything useful for us. However, getting back to the active connection on the target system reveals an interesting service running locally on the machine:

![image](/HTB/Orion/Orion_images/19.png)

The port 23 is open and listening at the **loopback address** (127.0.0.1), which means that **Telnet** is running locally on the target system. Searching for potential vulnerabilities related to it leads us to the **CVE-2026-24061:**

![image](/HTB/Orion/Orion_images/20.png)

`telnetd` improperly passes client‑controlled environment variables to `login(1)`.
By setting `CREDENTIALS_DIRECTORY=<attacker directory>` and placing `login.noauth = yes`, authentication is bypassed, potentially granting a root shell without a password.

Affected versions are up to and including version **2.7**. We can check the current binary version by running `telnetd --version` to check the current **Telnet Daemon** version installed:

![image](/HTB/Orion/Orion_images/21.png)

It appears that the binary is vulnerable to the discovered vulnerability! Since the executable has not been updated, we can download the **Python script** attached to this Github repository found earlier. Furthermore, Since the Telnet service is running locally and unreachable from our own machine, it is required to transfer it to the target through a **Python server** in an accessible folder, such as `/tmp`, in order to execute the malicious script locally and exploit the service properly:

![image](/HTB/Orion/Orion_images/22.png)

Once transferred, we can execute the script with the `-h` switch to obtain more information aboutt its usage:

![image](/HTB/Orion/Orion_images/23.png)

Since the service is running locally on the target system, we need to provide the loopback address (127.0.0.1) and the port associated to Telnet, which is **23**:

![image](/HTB/Orion/Orion_images/24.png)

We successfully escalated our privileges to **root**, granting full access and control over the target machine. The `/root` directory can now be accessed, and the final flag can be claimed:

![image](/HTB/Orion/Orion_images/25.png)

Machine completed!

![image](/HTB/Orion/Orion_images/26.png)




## Conclusion


This machine was very pleasant to complete. Starting from web enumeration, we discovered through the use of **Wappalyzer** that the web server was running **Craft CMS**. Searching for vulnerabilities associated to it led us to the **CVE-2025-32432** which allowed us to achieve **remote code execution** (RCE) without authentication. We leveraged this to obtain an initial foothold on the system as the **www-data** user. Listing the environment variables revealed critical credentials exposed in plaintext used by an internal **MySQL database**. We used these credentials to enumerate the database which contained the **password hash** associated to the user **adam**. Since the password was **weak**, we successfully cracked it offline using **John the Ripper**. Abusing a reused password through **SSH**, we pivoted our access from **www-data** to **adam**. Finally, we found another exposed service which was **Telnet**. Since the `telnetd` binary was outdated, we could exploit it through the **CVE-2026-24061** vulnerability exposure, which led us to escalate our privileges to **root** , granting full access and control. This challenge demonstrates how low-severity vulnerabilities chained can lead to a total compromise of the system.

**Thank you for reading!**
