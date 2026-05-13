## **Hello everyone!**

Welcome to my second Hack The Box walkthrough. In this writeup, we will explore the exploitation of the Valentine machine, an “Easy” box that many users considered closer to Medium difficulty.

Let’s get started!

![image](/HTB/Valentine/Valentine_images/1.png)

## Information Gathering

First, we use Nmap to scan the target and discover the exposed services:

![image](/HTB/Valentine/Valentine_images/2.png)

We found three running services :

**SSH (22)**

**HTTP (80)**

**HTTPS (443)**

  
 
Let’s see what this website is hosting:

![image](/HTB/Valentine/Valentine_images/3.png)

On the root directory there is nothing but a .jpg file, the code source doesn’t contain anything else. HTTPS hosts the same content.

We will do some directory brute-forcing using **Gobuster** in order to discover some directories hosted by the web server :

![image](/HTB/Valentine/Valentine_images/4.png)

We found interesting folders. Let’s check the “decode” directory:

![image](/HTB/Valentine/Valentine_images/5.png)

There is a field in which we can send some inputs that get transformed. It might be used to decode some encoded strings. The encode directory has a similar interface but user input gets encoded in Base64:

![image](/HTB/Valentine/Valentine_images/6.png)

Both websites use a respective .php file to encode/decode user input.

We can also take a look at the “dev” directory:

![image](/HTB/Valentine/Valentine_images/7.png)

We have two intriguing files there. Let’s check them:

![image](/HTB/Valentine/Valentine_images/8.png)

The first file contains hexadecimal strings. We can use [CyberChef](https://cyberchef.io/) to decode it :

![image](/HTB/Valentine/Valentine_images/9.png)

We now have an SSH private key ! This can be useful to gain our initial access if we can find a valid user on the system. As the key is called “key_hype”, we can assume it belongs to the user hype. However, when trying to authenticate with it, we are prompted for the key’s passphrase, which I could not crack using `ssh2john`. We will set that aside for now and continuing to enumerate the target, maybe we can find a password for this user.

Let’s check the second file:

![image](/HTB/Valentine/Valentine_images/10.png)

There is not much information that could help us.

## Exploitation

I came back to the initial Nmap phase to enumerate potential vulnerabilities on the web server that could let us obtain any sensitive information. Nmap is incredibly powerful as it includes the Nmap Scripting Engine (NSE), an extensible framework written in Lua that allows users to automate a wide range of tasks such as network discovery and vulnerability detection. The scan revealed multiple vulnerabilities. The Heartbleed vulnerability is particularly critical because it can leak sensitive information directly from server memory over the SSL/TLS protocol.

![image](/HTB/Valentine/Valentine_images/11.png)

We can use searchsploit to obtain an exploit for this particular vulnerability :

![image](/HTB/Valentine/Valentine_images/12.png)

We can run the script specifying the target:

![image](/HTB/Valentine/Valentine_images/13.png)

We successfully exploited the Heartbleed vulnerability! It allows us to retrieve leaked data from the server’s memory. We can spot an interesting base64 code sent back by `decode.php` here:

![image](/HTB/Valentine/Valentine_images/14.png)

Decoding this string with CyberChef or the Linux `base64` utility gives us the following: `heartbleedbelievethehype`

This could potentially be hype’s password. Let’s test it.

First, we have to crack the passphrase of the key. If you didn’t, copy all the content of the key and paste it locally on a file. Then, give it the required permissions:

![image](/HTB/Valentine/Valentine_images/15.png)

As I said earlier, we can’t log in directly with the private key. We can use OpenSSL to remove the passphrase protection from the private key. It asks for the key’s passphrase, but thanks to the Heartbleed vulnerability, we were able to recover sensitive data from memory.

![image](/HTB/Valentine/Valentine_images/16.png)

It works! We needed the `heartbleedbelievethehype` password to decrypt the key and use it. We can now try again to log in as hype:

![image](/HTB/Valentine/Valentine_images/17.png)

The SSH connection failed because modern OpenSSH clients disable the legacy `ssh-rsa` signature algorithm by default due to security concerns. This issue can be fixed by explicitly enabling the legacy algorithm using the `-o PubkeyAcceptedAlgorithms=+ssh-rsa` option:

![image](/HTB/Valentine/Valentine_images/18.png)

We successfully obtained initial access. We can list the user’s directory and fetch the first flag:

![image](/HTB/Valentine/Valentine_images/19.png)

## Post-Exploitation / Privilege Escalation

Now, we need to elevate our privileges in order to retrieve the `root.txt` flag. Attempting to enumerate sudo privileges using `sudo -l` was unsuccessful, as it required a password. I tried the one we got earlier, but it failed. Checking the crontab file doesn't reveal any interesting running cronjob. If we check the binaries with a root SUID, we find nothing interesting either.

Then, I tried to retrieve the .bash_history for our current user, which could reveal something interesting :

![image](/HTB/Valentine/Valentine_images/20.png)

We discover several `tmux` commands referencing an existing session and a custom socket path. Tmux is a popular Linux terminal multiplexer used to manage multiple sessions within a single terminal.

Let’s try to interact with the session that the user created. The `exit` command is followed by the `tmux -S /.devs/dev_sess` command, we can deduce that the session was started from there. So let’s try this:

![image](/HTB/Valentine/Valentine_images/21.png)

A root shell has spawned! The running session was owned by root. We can now fetch the root.txt flag:

![image](/HTB/Valentine/Valentine_images/22.png)

## Conclusion

This machine demonstrated how small misconfigurations and overlooked artifacts can lead to full system compromise. After gaining initial access, local enumeration revealed the presence of a privileged tmux session exposed through an accessible socket. By attaching to this session, we successfully obtained a root shell on the target.

This challenge highlights the importance of proper session management, secure socket permissions, and thorough post-exploitation enumeration.

Overall, Valentine was an interesting machine that combined web enumeration, cryptographic artifacts, a real-world SSL vulnerability, and Linux privilege escalation techniques into a very enjoyable challenge.

Thank you for reading !
