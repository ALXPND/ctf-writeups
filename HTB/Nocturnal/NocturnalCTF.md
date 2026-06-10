## Hello everyone!

Today, we will explore *Nocturnal*, an easy-level machine from **HackTheBox**.
Without further ado, let’s get started!

![image](/HTB/Nocturnal/Nocturnal_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP    nocturnal.htb`  to your local `/etc/hosts` file)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Nocturnal/Nocturnal_images/2.png)

We can identify two open TCP ports:

**22** (SSH)

**80** (HTTP)

As you can see, the `http.title` script indicates us that we need to enable DNS resolution for the target IP to `nocturnal.htb`, so ensure to add this domain to your `/etc/hosts` file)

We can now inspect the website hosted on port **80**:

![image](/HTB/Nocturnal/Nocturnal_images/3.png)

The website appears to use a web application for file upload purpose. To do this, we need to get authenticated at `login.php`. After trying a couple of common default credentials combinations, we were unable to authenticate as any existing user. We need to register in order to access the dashboard:

![image](/HTB/Nocturnal/Nocturnal_images/4.png)

We can now authenticate with the created account:

![image](/HTB/Nocturnal/Nocturnal_images/5.png)

We are now able to upload some files. If we attempt to upload a webshell file as `webshell.php`, we get an error:

![image](/HTB/Nocturnal/Nocturnal_images/6.png)

As we can see, we are only able to upload the following file extensions: `pdf`, `doc`, `docx`, `xls`, `xlsx`, and `odt`.

After uploading a legitimate file and accessing it, we can observe two GET parameters used by the web application: `username` and `file`:

![image](/HTB/Nocturnal/Nocturnal_images/7.png)

We can use `ffuf` ****to perform brute force on the `username` parameter in order to retrieve other users on the system:

![image](/HTB/Nocturnal/Nocturnal_images/8.png)

While fuzzing the username parameter, we noticed that the application

returns different responses depending on whether the user exists.

- Existing user → different response size / valid file access
- Non-existing user → generic error

To isolate valid usernames, we filtered responses based on their size.

We can identify the user **amanda**, which seems to host some uploaded files. We can find that an interesting `privacy.otd` file was uploaded:

![image](/HTB/Nocturnal/Nocturnal_images/9.png)

We can click on it to download it.

ODT files are actually ZIP archives containing XML files. Let’s extract the file using `unzip` to view what it contains:

![image](/HTB/Nocturnal/Nocturnal_images/10.png)

We can spot something very interesting in the `content.xml` extracted file:

![image](/HTB/Nocturnal/Nocturnal_images/11.png)

We can retrieve the cleartext password attributed to the user **amanda**: `arHkG7HAI68X8s1J`.

Since the password appears to be associated to the web application, we can attempt to authenticate with the following credentials:

`amanda:arHkG7HAI68X8s1J`

![image](/HTB/Nocturnal/Nocturnal_images/12.png)

![image](/HTB/Nocturnal/Nocturnal_images/13.png)

We successfully authenticated as **amanda**. Moreover, we can now access an *admin panel*, so let’s do this so see what we can do on it:

![image](/HTB/Nocturnal/Nocturnal_images/14.png)

The panel allows us to view web application’s files content. We can also backup the current folder providing a password in a field. Inspecting the `admin.php` code, we can observe that a blacklist has been used to restrict some characters:

![image](/HTB/Nocturnal/Nocturnal_images/15.png)

First, it checks for disallowed characters in the password, and returns `false` if such a character is found. Generally speaking, using a deny-list like this is poor practice, because the list may not be complete (as in this case, which we’ll see later).

The application constructs a system command using user input:

`zip -x './backups/*' -r -P <password> <backupFile> .`

Since the password is directly concatenated into the command, we can break out of the intended argument using a double quote ("). For example:

`password"ls`

This results in command injection, allowing us to execute arbitrary commands.

![image](/HTB/Nocturnal/Nocturnal_images/16.png)

In the output, we can observe the directory listing caused by the `ls` command we injected. However, if we attempt to inject a command with whitespaces we receive an error:

![image](/HTB/Nocturnal/Nocturnal_images/17.png)

## Initial Access

Inspecting the `dashboard.php` code reveals the SQLite3 database configuration file:

![image](/HTB/Nocturnal/Nocturnal_images/18.png)

The file is located at `../nocturnal_database/nocturnal_database.db`

The application blocks spaces, but does not filter tab characters (`%09`).

We can use `%09` as a space bypass.

Final payload:
`password%09backups%2fbackup.zip%09..%2fnocturnal_database%22`

This effectively injects additional arguments into the zip command,
forcing it to include the database file in the generated archive.

We can intercept the request in Burp Suite and inject the following payload in order to backup this specific file and retrieve its `.zip` archive:

![image](/HTB/Nocturnal/Nocturnal_images/19.png)

![image](/HTB/Nocturnal/Nocturnal_images/20.png)

We can then navigate to `http://nocturnal.htb/backups/backup.zip` to download the `.zip` archive containing the database configuration file we pointed:

(accordingly to the used payload, the password for file extraction is `password`)

![image](/HTB/Nocturnal/Nocturnal_images/21.png)

![image](/HTB/Nocturnal/Nocturnal_images/22.png)

We can use  the `nocturnal_database.db` through the `sqlite` utility to navigate in the database of the target in order to retrieve sensitive information that could help us to gain initial access on the machine. Inspecting the `users` table, we can identify a user we didn’t discovered through our previous web enumeration with `ffuf`:

![image](/HTB/Nocturnal/Nocturnal_images/23.png)

We can identify a user named **tobias** and his **password’s hash**. Since the user  does not seem to be part of the web application registry, we can assume that he has a SSH account, and the credential we found might allows us to obtain an initial access from this service. But to do this, we first need to crack the hash. Since this is a 32 characters hash which seems to represent **MD5**, we should be able to crack it relatively easy if the password isn’t reinforced. 

We will paste the hash in a **hash.txt** file and pass it to **John The Ripper** in ****order to crack it offline**:**

![image](/HTB/Nocturnal/Nocturnal_images/24.png)

We successfully cracked the MD5 hash, this allows to retrieve the following cleartext password: `slowmotionapocalypse` (which is a quite weak password).

Since this password is owned by the user **tobias**,  we can attempt to authenticate us via SSH with the following credential:

`tobias:slowmotionapocalypse`

![image](/HTB/Nocturnal/Nocturnal_images/25.png)

We successfully obtained initial access on the machine as **tobias**.

We can list the current directory and retrieve the first flag:

![image](/HTB/Nocturnal/Nocturnal_images/26.png)





## Privilege Escalation



Since there is no other non-root user on the system, we need to escalate our current privileges to **root** in order to retrieve the final flag.

It is a good practice to start enumerating privilege escalation vectors by running `sudo -l` if we have a valid password for our current user. It allows us to list potential command we could run with **root privileges**:

![image](/HTB/Nocturnal/Nocturnal_images/27.png)

But here, it appears that we can’t do this. 

Looking at the cron jobs, capabilities and binaries with the SUID bit set doesn’t reveals anything uncommon misconfiguration we can abuse. However, inspecting the `/var/www` web root directory reveals an interesting web interface we didn’t identify before:

![image](/HTB/Nocturnal/Nocturnal_images/28.png)

From the `/usr/local/ispconfig/interface/web` symlink, we can deduce that there is an **ISPConfig** web application that is running locally on the target machine. Moreover, we can observe that the web application is **root-owned**. It means that if we find a way to execute arbitrary command from this web interface, we will execute system commands with **root privileges**, granting full access and control on the machine.

We can verify its presence using `ss -tulnp` to list the active connections on the target. As expected, it reveals an HTTP webserver hosted on port **8080** that we didn’t discover with **Nmap** because the web application is running locally:

![image](/HTB/Nocturnal/Nocturnal_images/29.png)

We can access the website from our local machine via **port forwarding**. We will start a new SSH instance with the following command:

`ssh -L 1234:127.0.0.1:8080 tobias@nocturnal.htb`

After the authentication established, the command allows us to port forward port **8080** to us, making it accessible at the assigned port (here, it’s **1234**) at the following URL:

`http://127.0.0.1:1234`

![image](/HTB/Nocturnal/Nocturnal_images/30.png)

We successfully reached the webserver running locally on the target from our own machine. Indeed, the web application deployed here is **ISPconfig**, granting us with a login panel. After attempting a couple of credentials combination, we successfully authenticate with `admin:slowmotionapocalypse`, reusing the SSH password for the user **tobias**:

![image](/HTB/Nocturnal/Nocturnal_images/31.png)

The *Help* section allows us to identify the web application version, which is  **v3.2.10p1**.

![image](/HTB/Nocturnal/Nocturnal_images/32.png)

Let’s research for potential vulnerability related to it:

![image](/HTB/Nocturnal/Nocturnal_images/33.png)

There is a vulnerability referenced as **CVE-2023-46818** which leverage an improperly sanitized POST parameter to `/language_edit.php` file. This allows an authenticated user to inject and execute arbitrary PHP code. The vulnerability affects version 3.2.11 and prior versions. Since we got an authenticated access to the dashboard, this vulnerability matches perfectly with our situation. There is a GitHub repository which hosts a **Python** script exploiting the concerned vulnerability. We can download and execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/Nocturnal/Nocturnal_images/34.png)

Since this is an authenticated exploit, we need to provide the target URL pointing the web application (remember, we port forwarded it to our local machine on the port **1234**) and valid credentials. We can run the following command:

 `python3 CVE-2023-46818.py http://127.0.0.1:1234 admin slowmotionapocalypse`

![image](/HTB/Nocturnal/Nocturnal_images/35.png)

We successfully escalate our privileges to **root**! We executed arbitrary code with **root privileges**, allowing us to spawn a **root Bash shell**, granting full access.
We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Nocturnal/Nocturnal_images/36.png)

Challenge completed!

![image](/HTB/Nocturnal/Nocturnal_images/37.png)

## Conclusion

This CTF was very pleasent to complete. We started by enumerating a web application that allows users to upload some file. After performing brute forcing an exposed **GET parameters**, we discovered the user **amanda** which has uploaded a `privacy.odt` file insecurely storing a cleartext password for the web application. We could use it to authenticated as **amanda**, which allowed us to access an admin panel that permitted to backup and read application’s code, exposing the local path of a dangerous database configuration file. We fetched the **SQLite3** configuration file, which led us to retrieve the `users` table and the password hash for an SSH user, which allowed us to gain access on the system. From this initial access, we were able to identify a web application running locally on the machine. Via **port-forwarding**, we successfully accessed the website from our own machine and authenticated with legitimate credentials abusing a password reuse. We were then able to identify the web application’s version, which contained a vulnerability that can be exploited with an authenticated access. Since we met all the criteria, we were able to execute **arbitrary commands** on the system with **root privileges** because the web application was owned by the **root** user. Consequently, we spawned a **root Bash shell**, granting full access and control on the target machine.

This challenge demonstrated how **chaining** multiple low-severity misconfigurations can lead to a total system compromise. 



**Thank you for reading!**
