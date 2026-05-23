## Hello everyone!

Today we will explore “Broker”, an easy level **HackTheBox** machine.

![image](/HTB/Broker/Broker_images/1.png)

(ensure that you have added the following line  `TARGET_IP     broker.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to enumerate as much information as possible:

![image](/HTB/Broker/Broker_images/2.png)

We can identify two services running on the target:

**SSH** (OpenSSH 8.9p1 | 22)

**HTTP** (nginx 1.18.0 | 80)

We can inspect the website hosted on port 80:

![image](/HTB/Broker/Broker_images/3.png)

And we are prompted to enter credentials. We can access the protected page with common default credentials :

`admin : admin`

![image](/HTB/Broker/Broker_images/4.png)

We now have access to an ActiveMQ CMS. This service runs alongside port **61616**, which can help us identify the ActiveMQ version.

![image](/HTB/Broker/Broker_images/5.png)

We can learn that ActiveMQ has the version 5.15.15.

## Initial Access

Doing some research about vulnerabilities concerning the version led us to the **CVE-2023-46604**, a vulnerability that allow remote code execution (RCE).

We can use the Python scripts from this [GitHub repository](https://github.com/strikoder/CVE-2023-46604-ActiveMQ-RCE-Python) to achieve our goal:

![image](/HTB/Broker/Broker_images/6.png)

As we can see, there is two scripts: `generate_poc.py` and `main.py`.

The first script will generate a `.xml` file containing our local IP and our listening port (because we will set up a **netcat listener**)

The second script will serve the `.xml` file (hosted by a Python server we need to establish) to the target , which should enable us to obtain a reverse shell. 

So let’s setup this:

![image](/HTB/Broker/Broker_images/7.png)

If everything has been configured correctly, you should obtain a connection back to your listener as above.

We successfully obtained initial access as the **activemq** user. We can stabilize our current shell with the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can now check our directory located to `/home/activemq` and retrieve the first flag:

![image](/HTB/Broker/Broker_images/8.png)

## Post-Exploitation

After that, we need to elevate our privileges to retrieve the last flag. We can then enumerate potential privilege escalation paths by running the `sudo -l` to see whether we can run elevated commands or not:

![image](/HTB/Broker/Broker_images/9.png)

It seems that we can run the `nginx` command with root privileges without providing any password, which is interesting.

The Nginx directive used to define allowed HTTP methods (including for WebDAV setups) is `dav_methods`. We abuse `nginx` running as **root** via `sudo` to load a custom configuration file, allowing us to expose sensitive filesystem paths through a misconfigured web server. Mine is quite simple. `user` will be root. It must have an `events` section to define the number of workers, so I’ll pick something arbitrary. Then I make an `http` section with a server that is hosted from the system root:

```
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1337;
        root /;
        autoindex on;
    }
}
```

(We create the file in the `/dev/shm` directory as it is writable and commonly used for temporary payload storage.)

This configuration exposes the entire filesystem via directory listing enabled by autoindex.

We can then run the following command to handle the config file:

`sudo /usr/sbin/nginx -c /dev/shm/malicious.conf`

![image](/HTB/Broker/Broker_images/10.png)

Now that the `.conf` file has been treated, we can access the restricted resources hosted on port **1337** via curl using the following command:

`curl http://localhost:1337/restricted/file`

![image](/HTB/Broker/Broker_images/11.png)

It works! We can now browse sensitive files on the system, including the `root.txt` flag.

![image](/HTB/Broker/Broker_images/12.png)

Challenge completed!

![image](/HTB/Broker/Broker_images/13.png)

## Conclusion

“Broker” is a classic easy-level HackTheBox machine that highlights how **exposed services combined with outdated software and weak privilege separation can quickly lead to full system compromise**.

The initial foothold was achieved by identifying an exposed Apache ActiveMQ instance and leveraging a known vulnerability (CVE-2023-46604), which allowed us to gain remote code execution as the **activemq** user. This step demonstrates the importance of keeping messaging and broker services properly updated, as they often run with high privileges and are frequently exposed to the network.

Once inside the system, post-exploitation enumeration revealed a critical misconfiguration in sudo privileges: the ability to execute nginx as root without a password. Instead of a direct exploit, privilege escalation was achieved through configuration abuse. By crafting a malicious nginx configuration file, we were able to manipulate the server’s behavior to expose the filesystem and ultimately access sensitive data, including the root flag.

This machine reinforces several key security lessons:

- Always patch exposed services such as ActiveMQ to prevent known RCE vulnerabilities
- Never allow unrestricted sudo execution of powerful services like nginx
- Misconfigurations can be just as dangerous as vulnerabilities
- Service-level abuse (like nginx config manipulation) can be a reliable privilege escalation vector
- Enumeration after initial access is critical to identify misconfigured privilege boundaries

Overall, “Broker” is a great example of a real-world attack chain combining **public exploit usage, service misconfiguration, and privilege escalation through legitimate software features**, emphasizing that security failures often come from multiple small mistakes rather than a single flaw.

**Thank you for reading!**
