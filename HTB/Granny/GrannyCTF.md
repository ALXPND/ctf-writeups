## Hello everyone!

Today, we will explore “Granny”, an easy-level machine from **HackTheBox**.
Without further ado, let’s get started!

![image](/HTB/Granny/Granny_images/1.png)

(ensure that you have added the following line  `TARGET_IP     granny.htb`  to your local `/etc/hosts` file)

First, we will perform an aggressive **Nmap** scan to collect as much information as possible about the target:

![image](/HTB/Granny/Granny_images/2.png)

It seems that there is only one service running on the target, which is HTTP, serving a Microsoft IIS 6 server on port 80. So let’s inspect the website hosted on this port:

![image](/HTB/Granny/Granny_images/3.png)

As we can see, the website appears to be under construction. But from our Nmap scan, we identified that WebDAV is configured. We can use **Gobuster** to ****perform web directory enumeration:

![image](/HTB/Granny/Granny_images/4.png)

We found multiple folders, including “`_vti_bin`”, which confirms that **WebDAV** is enabled.

Since our **Nmap** scan revealed several dangerous **WebDAV methods** (such as PUT), we will now focus on Microsoft IIS 6 exploits related to **WebDAV** to identify potential attack vectors.

After researching potential exploits, we identified **CVE-2017-7269**, a buffer overflow in the **ScStoragePathFromUrl** function in the WebDAV service in IIS 6.0. We can use a Metasploit module (`windows/iis/iis_webdav_scstoragepathfromurl`) to leverage this vulnerability and obtain initial access:

![image](/HTB/Granny/Granny_images/5.png)

And we successfully obtained initial access on the machine! Moreover, we obtained an elevated session as NT AUTHORITY\SYSTEM, which means that we have full control on the target. This means that we can directly retrieve both flags, `user.txt` and `root.txt`:

![image](/HTB/Granny/Granny_images/6.png)

Challenge completed!

![image](/HTB/Granny/Granny_images/7.png)

## Conclusion

“Granny” is a very beginner-friendly HackTheBox machine that provides a great introduction to exploiting legacy Microsoft IIS services and WebDAV vulnerabilities.

During this challenge, we started by enumerating the target with Nmap, which revealed an IIS 6.0 web server with WebDAV enabled. After identifying potentially dangerous HTTP methods and discovering the `_vti_bin` directory through directory fuzzing, we researched known vulnerabilities affecting IIS 6.0 and found the well-known CVE-2017-7269 buffer overflow vulnerability.

Using the corresponding Metasploit module, we successfully gained remote code execution on the target and obtained a SYSTEM shell, giving us full control over the machine and allowing us to retrieve both flags immediately.

This machine highlights several important lessons:

- The importance of proper enumeration
- Risks associated with outdated software such as IIS 6.0
- Dangerous WebDAV configurations
- How public vulnerabilities can quickly lead to full system compromise

Overall, “Granny” is an excellent box for beginners who want to practice Windows enumeration and learn how vulnerable legacy web technologies can be exploited in real-world scenarios.

**Thank you for reading!**
