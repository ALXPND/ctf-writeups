## **Welcome everyone!**

Today we will cover the Chemistry machine from HackTheBox.



![image](/HTB/Chemistry/Chemistry_images/1.png)



(Ensure that you have added the following line: `10.129.231.170  chemistry.htb` at the end of your local `/etc/hosts` file remplacing your target IP address)

## Information Gathering

We will start with an aggressive Nmap scan to gather as much information as possible:



![image](/HTB/Chemistry/Chemistry_images/2.png)



We can identify two services running on the target:

**SSH** (OpenSSH 8.2p1 | 22)

**HTTP** (Werkzeug 3.0.3 | 5000)

It seems that a web server on port 5000, so let’s visit it:



![image](/HTB/Chemistry/Chemistry_images/3.png)



We are prompted to login or to register. We can deduce that a dashboard is accessible, so let’s register in order to authenticate us:



![image](/HTB/Chemistry/Chemistry_images/4.png)



Then we login with the generated account:



![image](/HTB/Chemistry/Chemistry_images/5.png)



And we now have access to a dashboard through which we can upload some files. But the expected format is a intriguing `.cif` file. 

## Initial Access

After some research, we can learn that the parsing for this particular type of file has a known vulnerability, the CVE-2024-23346. Let’s search about a Proof of Concept for this vulnerability:



![image](/HTB/Chemistry/Chemistry_images/6.png)



We found a particular exploit related to it! Let’s download the python script from the [GitHub repository](https://github.com/Sanity-Archive/CVE-2024-23346) and execute it to see what happens:



![image](/HTB/Chemistry/Chemistry_images/7.png)



(if you get issue with the packages dependencies, you can run `pip3 install <requiered_package>`. If it doesn’t work due to security measures, you will need to set up a virtual python environnement with `python3 -m venv venv` and `source /venv/bin/activate`, after what the `pip3 install` command should works)

It seems that we need to specify some arguments to the script, including the target URL, an username and a password (that we have since we registered earlier), and our IP.

Let’s provide it and execute the python script:



![image](/HTB/Chemistry/Chemistry_images/8.png)



And we successfully obtained our initial access as the **app** user! But since we got a unstable shell which restrict us in our actions, let’s upgrade it using the following command:

`python3 -c 'import pty; pty.spawn(”/bin/bash”)'`

(we can run as well `export TERM=xterm` to get back the `clear` utility functional)

We can enumerate the `/home` directory to see what other users are hosting on the system. It seems that the user.txt located to the `/home/rosa` directory isn’t accessible with our access:



![image](/HTB/Chemistry/Chemistry_images/9.png)



## Privilege Escalation

Since we can’t access the first flag, we need to elevate our privileges. After enumerate our own directory, we can see a interesting `database.db` file:



![image](/HTB/Chemistry/Chemistry_images/10.png)



Let’s take a look:



![image](/HTB/Chemistry/Chemistry_images/11.png)



The file stores **MD5 account hashes**, which one seems to be owned by the user **rosa**! Since MD5 is a **very weak** hashing algorithm, we can attempt to crack it offline using **John**. We need to paste the hash into a hash.txt local file and send it to John with the following command:



`john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt` 



![image](/HTB/Chemistry/Chemistry_images/12.png)



We successfully cracked the rosa’s password, which is `unicorniosrosados`. We will now use it to log in via SSH:



![image](/HTB/Chemistry/Chemistry_images/13.png)



Indeed, the password was reused across SSH, which permit us to pivot our access to the user **rosa**, allowing us to retrieve the first flag:



![image](/HTB/Chemistry/Chemistry_images/14.png)



Let’s check with `sudo -l` if we can run privileged command:



![image](/HTB/Chemistry/Chemistry_images/15.png)



We can’t. 

After enumerating potential cron jobs, capabilities and SUID binaries, we find nothing interesting that we can leverage to gain root access. Maybe we need to access internal resources? Let’s check whether an internal service is running on the system using the `ss` utility:



![image](/HTB/Chemistry/Chemistry_images/16.png)



We can identify a web server running on the internal network of the system on port **8080**!

We will perform port-forwarding in order to access this web server from our local machine. To do this, we will initiate an other SSH session with the following command:

`ssh -L 1234:127.0.0.1:8080 rosa@chemistry.htb`

Here, the `-L` switch specify the local port we will use to access the internal web server from our own web browser at the following URL: `http://127.0.0.1:1234`. After providing the rosa password, we can access to the internal ressource through SSH tunneling:



![image](/HTB/Chemistry/Chemistry_images/17.png)



It works! We reached an internal web server which is hosting a monitoring webiste. But we don’t have much more information about the technology employed. Let’s use `curl` to see if a particular header contains informations:



![image](/HTB/Chemistry/Chemistry_images/18.png)



It looks like the web server is running **AIOHTTP 3.9.1**. After searching for vulnerability associated to it, we can find a [very well documented PoC](https://github.com/z3rObyte/CVE-2024-23334-PoC) for a Directory Traversal vulnerability also know as **CVE-2024-23334**. It arises due to insufficient sanitization when handling static file requests. Attackers can traverse the directory structure to access sensitive files.

We can exploit this LFI from the remote target as well running the following command:

`curl --path-as-is http://localhost:8080/assets/../../../../../root/root.txt`

This payload mount up the file system arborescence until it can reach the root directory, allowing us to retrieve any content within it. Here, we attempt to fetch the `root.txt` file:



![image](/HTB/Chemistry/Chemistry_images/19.png)



And we’re done!



![image](/HTB/Chemistry/Chemistry_images/20.png)
