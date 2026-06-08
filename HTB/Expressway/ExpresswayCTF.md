## Hello everyone!






Today, we will explore *Expressway*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/Expressway/Expressway_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP     expressway.htb` to your local `/etc/hosts` file to enable DNS resolution)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:

![image](/HTB/Expressway/Expressway_images/2.png)

We identify only one open TCP port on the target which is 22 (**SSH**). We will launch a UDP scan to see whether we can find other exposed services:

![image](/HTB/Expressway/Expressway_images/3.png)

We identify a service running on port **500**: **ISAKMP**.

**ISAKMP** typically works with IKE (Internet Key Exchange) to automate the process of setting up secure IP communications. We will use the `ike-scan` tool to scan the target through the service:

![image](/HTB/Expressway/Expressway_images/4.png)

As we can see, there is a stored hash attached to the user **ike** (ike@expressway.htb). We can attempt to dump it using the following command: 

`ike-scan -A -pskcrack expressway.htb`

![image](/HTB/Expressway/Expressway_images/5.png)




## Initial Access


We retrieved the **PSK SHA-1** hash. We can save it into a `hash.txt` file and pass it to hashcat in order to crack it offline using the following command:

`hashcat -m 5400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/Expressway/Expressway_images/6.png)

We successfully crack the hash, retrieving the `freakingrockstarontheroad` cleartext password. Since the password is owned by the user **ike**, we can assume the password is reused through SSH. Let’s attempt to authenticate via SSH as the user **ike** with the following credentials:

`ike:freakingrockstarontheroad`

![image](/HTB/Expressway/Expressway_images/7.png)

It works! We obtained initial access on the target leveraging a password reuse for the user **ike**, which demonstrates a very bad security practice. We can now list the current directory and retrieve the user flag:

![image](/HTB/Expressway/Expressway_images/8.png)




## Privilege Escalation


Since we’re the only non-root user present on the system, we now need to escalate our current privileges to **root** in order to retrieve the final flag. We start local privilege escalation enumeration by running `sudo -l` to see whether we are able to run any command with **root privileges**:

![image](/HTB/Expressway/Expressway_images/0.png)

Unfortunately, we can’t do that.

Checking potential cron jobs or capabilities doesn’t reveal anything useful for us. However, listing binaries with the SUID bit set reveals an uncommon configuration:

![image](/HTB/Expressway/Expressway_images/10.png)

The `exim4` binary has an uncommon SUID bit set on it. Researching about potential only reveals vulnerability affecting exim before **v4.86.2**, but our current version is **v4.98.2**. However, we observe that the user is part of the `proxy` group when we run the `id` command.

We use again the `find` command again to list readable files by this particular group:

![image](/HTB/Expressway/Expressway_images/11.png)

As we can see, **Squid** is used as an HTTP proxy.

Inspecting the `/var/log/squid/access.log.1` reveals that the host `offramp.expressway.htb` was accessed through the `squid` proxy:

![image](/HTB/Expressway/Expressway_images/12.png)

We keep this host in mind for further analysis.

Inspecting the `sudo` version, we can identify an outdated version:

![image](/HTB/Expressway/Expressway_images/13.png)

The binary is still running under the **v.1.9.16** version when it should be updated to **1.9.17p1** or **1.9.17.p2** (as may be your Kali machine configuration). The utility has a vulnerability caused by a bug in its `-h` argument allowing to execute arbitrary commands on the system with **root privileges**. The vulnerability is referenced as the **CVE-2025-32462**. We can run this basic command to spawn a root **Bash shell:**

`sudo -h offramp.expressway.htb /bin/bash`

![image](/HTB/Expressway/Expressway_images/14.png)

We successfully escalated our privileges to **root**, granting us full system compromise! We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/Expressway/Expressway_images/15.png)

Challenge completed!

![image](/HTB/Expressway/Expressway_images/16.png)

## Conclusion

In this machine, we followed a realistic attack chain combining network misconfigurations, credential reuse, and privilege escalation weaknesses.

We began with reconnaissance, where an initial scan revealed only SSH exposed on the target. Further UDP enumeration led to the discovery of an ISAKMP service running on port 500, which allowed us to extract a pre-shared key using `ike-scan`. This key was successfully cracked offline, revealing credentials that were reused for SSH access.

This granted us initial access as the user ***ike***, highlighting the risks of weak password hygiene and credential reuse across services.

For privilege escalation, we performed local enumeration and identified multiple potential attack surfaces, including an unusual SUID configuration and proxy-related artifacts. Further investigation of Squid proxy logs revealed internal hostnames accessible through the system, providing valuable insight into the internal network structure.

Finally, by exploiting a vulnerable sudo configuration, we were able to escalate privileges to root and fully compromise the system.

Overall, this machine demonstrates how combining service enumeration, credential cracking, and careful local enumeration can lead to full system compromise.

**Thank you for reading!**
