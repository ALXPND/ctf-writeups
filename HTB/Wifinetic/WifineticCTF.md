## **Welcome everyone!**

Today we will explore the “Wifinetic” machine released by HackTheBox.

Let’s get started!



![image](/HTB/Wifinetic/Wifinetic_images/1.png)



(add the following line `TARGET_IP   wifinetic.htb` at the end of your local `/etc/hosts` file)

## Information Gathering

First; we will perform an aggressive **Nmap** scan in order to obtain comprehensive results:



![image](/HTB/Wifinetic/Wifinetic_images/2.png)



The scan revealed three running services on the target:

FTP (vsftpd 3.0.3 | 21)

SSH (OpenSSH 8.2p1 | 22)

DNS (53)

The Nmap Script Engine tells us that we can access the FTP server via **anonymous login**, since it is allowed. Moreover, its content is directly included in the scan results:



![image](/HTB/Wifinetic/Wifinetic_images/3.png)



Then we can log in using the following credentials: `anonymous : anonymous`



![image](/HTB/Wifinetic/Wifinetic_images/4.png)




This confirms what we saw earlier. We can download the fourth documents using the `get` command:




![image](/HTB/Wifinetic/Wifinetic_images/5.png)




We can see that there is an archive. We can extract its content running the `tar -xvf` command:




![image](/HTB/Wifinetic/Wifinetic_images/6.png)




We extracted all the archive contents into an `etc` directory. After enumerating it, we can retrieve an interesting `wireless` file which stores a **cleartext password** attached to the **wifinet0** and **wifinet1** interfaces:



![image](/HTB/Wifinetic/Wifinetic_images/7.png)




## Initial Access


Since we retrevied the `/etc` directory of the system, we can access the `passwd` file in order to identify other users present on the target:




![image](/HTB/Wifinetic/Wifinetic_images/8.png)




We can identify a **netadmin** user. What happens if we try to log in via SSH as this user using the password we found earlier?



![image](/HTB/Wifinetic/Wifinetic_images/9.png)




It works! We obtained initial access through a password reuse as the **netadmin** user, which allows us to retrieve the first flag:




![image](/HTB/Wifinetic/Wifinetic_images/10.png)




We can check whether we are allowed to run commands with root privileges running `sudo -l`:



![image](/HTB/Wifinetic/Wifinetic_images/11.png)



It seems that we are not allowed to run privileged commands. Privilege escalation is still required to retrieve the final flag.

After an in-depth enumeration, we can identify that a user space daemon software is being used. We can use `iwconfig` to confirm the presence of wireless interfaces:



![image](/HTB/Wifinetic/Wifinetic_images/12.png)



We can identify two interesting network interfaces: `mon0` and `wlan1`.

There is a tool we can use which is already installed on the target: `reaver`

This utility can be used to leverage a potential weakness in the WPA/WPS protocol.

Since we retrieved a legitimate access point through the `wlan1` interface, we can pass it to `reaver` in order to recover the WPA password with the following command:

`reaver -i mon0 -b <ACCESS POINT>`



![image](/HTB/Wifinetic/Wifinetic_images/13.png)



We successfully obtained the WPA password! What happens if we use this password to switch access to root?



![image](/HTB/Wifinetic/Wifinetic_images/14.png)



It works! We gained root access, leading to a complete system compromise.
We can now navigate in the `/root` directory and fetch the remaining flag:



![image](/HTB/Wifinetic/Wifinetic_images/15.png)



## Conclusion

This machine demonstrated a complete penetration testing workflow, starting from initial reconnaissance to full system compromise. The exploitation path began with an **anonymous FTP access**, which exposed sensitive system files and led to the discovery of a **cleartext credential reuse** scenario. This allowed us to obtain an initial foothold as the `netadmin` user via SSH.

Once inside the system, privilege escalation was achieved through deeper enumeration of the wireless configuration. The presence of a misconfigured wireless setup and the availability of tools such as Reaver enabled exploitation of a weak WPS implementation, ultimately leading to root-level access.

Overall, *Wifinetic* highlights several important security lessons: the risks of **anonymous services exposure**, **plaintext credential storage**, and **misconfigured wireless security protocols**. It also reinforces the importance of thorough enumeration and thinking beyond traditional attack vectors during post-exploitation.



![image](/HTB/Wifinetic/Wifinetic_images/16.png)



**Thank you for reading!**
