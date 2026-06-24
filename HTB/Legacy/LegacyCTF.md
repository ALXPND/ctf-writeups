## Welcome everyone!





Today, we will explore *Legacy*, an easy-level machine from HackTheBox.

Without further ado, let’s get started! 

![image](/HTB/Legacy/Legacy_images/1.png)





## Information Gathering


First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Legacy/Legacy_images/2.png)

Three different services are exposed:

**RPC** (135)

**NetBIOS** (139)

**SMB** (445)

The scan reveals that the target is running a very old Windows system (Windows XP / LAN Manager) which is highly insecure from a security perspective. Researching this OS version indicates that the target is likely vulnerable to the critical **MS-08-067** vulnerability:





## Exploitation

![image](/HTB/Legacy/Legacy_images/3.png)

Given the age of the operating system, MS08-067 immediately becomes a likely attack vector.

An attacker can exploit this vulnerability without authentication. Since this is a very well-known vulnerability, there is a **Metasploit** module that leverages this vulnerability. We will select it and configure the `RHOSTS` and `LHOST` values correctly:

![image](/HTB/Legacy/Legacy_images/4.png)

The module can then be run executed to exploit the vulnerability:

![image](/HTB/Legacy/Legacy_images/5.png)

We successfully exploited the outdated Windows version running, which allowed us to obtain initial access to the target machine through a **Meterpreter** session with **SYSTEM-level privileges**.





## Post-Exploitation


We can now use the built-in Meterpreter `search` command to retrieve both flags:

![image](/HTB/Legacy/Legacy_images/6.png)


Machine completed!

![image](/HTB/Legacy/Legacy_images/7.png)




## Conclusion

Overall, *Legacy* was honestly pretty straightforward. The target machine was very old and exposed a classic SMB vulnerability (**MS08-067**), so once you identify the OS and spot the attack surface, there’s not much complexity left. Exploitation is basically plug-and-play with **Metasploit**, and privilege level is already high from the start. I still wanted to show through this challenge how a single critical vulnerability can lead to a full system compromise very easily.

It’s a good introductory box for understanding how dangerous outdated systems can be, but in terms of difficulty it stays really easy and doesn’t require much chaining or deeper analysis.


**Thank you for reading!**
