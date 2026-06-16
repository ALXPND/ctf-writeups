## Welcome everyone!

Today, we will cover “*Explore*”, an easy-levem machine from HackTheBox.

Without further ado, let’s get started!

![image](/HTB/Explore/Explore_images/1.png)

(Please ensure that you have added the following line   `TARGET_IP     explore.htb`   to your local `/etc/hosts` file to enable **DNS resolution**)

## Information Gathering

First, we will perform a full TCP port **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/Explore/Explore_images/2.png)

We can identify two exposed services:

- **SSH** (2222)
- **HTTP** (59777)

We can identify a **ES File Explorer** application hosted on port 80.

## Initial Access

 Searching for potential vulnerability related to it lead us to **CVE-2019-6447** exposure. The ES File Explorer File Manager application through 4.1.9.7.4 for Android allows remote attackers to **read arbitrary files** or execute applications via TCP port **59777** requests on the local Wi-Fi network. This TCP port remains open after the ES application has been launched once, and responds to unauthenticated application/json data over HTTP.
We can use the Metasploit `es_file_explorer_open_port` module to achieve this. The module has been configured as follow:

![image](/HTB/Explore/Explore_images/3.png)

We can identify an interesting file in the `/storage/emulated/0/DCIM/` directory:

![image](/HTB/Explore/Explore_images/4.png)

It appears that credentials are stored here as a `creds.jpg` file. We can download the file by setting `GETFILE`  to the `ACTION` option and providing the absolute path of the file to the `ACTIONITEM` option:

![image](/HTB/Explore/Explore_images/5.png)

The file has successfully been downloaded and transfered to our local `/home/<USER>/.msf4/loot/` directory. We can view the image using `ristretto`. The following command will be used:

`/home/<USER>/.msf4/loot/20260616234503_default_10.129.238.59_getFile_110817.jpg`

![image](/HTB/Explore/Explore_images/6.png)

We can retrieve the `Kr1sT!5h@Rp3xPl0r3!` cleartext password for the user Kristi. We maybe be able to authenticate via SSH (port 2222) with these credentials. Let’s attempt it. Since the modern SSH remote service doesn’t accept the ssh-rsa algorithm. Consequently, we need to authenticate using the following command:

`ssh -p 2222 kristi@explore.htb -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa`

![image](/HTB/Explore/Explore_images/7.png)

It works! We successfully obtained initial access as the user **kristi**. Since the environment is special, there is no `/home` directory to look at. We can retrieve the first flag in the `/storage/emulated/0` directory:

![image](/HTB/Explore/Explore_images/8.png)




## Privilege Escalation



We now need to escalate our current privileges to root in order to retrieve the final flag.

Inspecting active connections with `ss -tulnp` reveals few interesting ports:

![image](/HTB/Explore/Explore_images/9.png)

We can identify port 5555 that we didn’t found before. 

Searching about this port learn us that its used for the **Android Debug Bridge** (adb) command-line tool.

We can assume that the service is running locally on the target system Since we have valid SSH credentials, we can perform **port-forwarding** to access it from our local machine. This can be done with the following command:

`ssh -p 2222 -L 1234:127.0.0.1:5555 kristi@explore.htb -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa`

The command shown above port forward the remote service to our local port **1234**.

We can now connect to the Android Debug Bridge service using the following command:

`adb connect localhost:1234`

We can then spawn a adb shell by simply running `adb shell`:

![image](/HTB/Explore/Explore_images/11.png)

It works, but our session isn’t elevated. Fortunaly, there is a command that allows to elevate the adb session privilege. We can quit and run `adb root` to upgrade the shell to **root**. We can then run again **adb shell** to spawn the **elevated** adb shell:

![image](/HTB/Explore/Explore_images/12.png)

We successfully escalated our privileges to **root,** granting full access and controll**.** We can use the following `find` command to retrieve the final flag:

`find / -type f -name root.txt 2>/dev/null`

![image](/HTB/Explore/Explore_images/13.png)




## Conclusion

This machine demonstrated how an exposed Android service combined with insecure application design can lead to full system compromise.

We began by discovering an ES File Explorer service vulnerable to **CVE-2019-6447**, which allowed us to read arbitrary files on the device. This led to the discovery of stored credentials inside an image file, giving us access to the system via **SSH** as the user *kristi*.

After initial access, further enumeration revealed an exposed **ADB (Android Debug Bridge)** service running locally. By leveraging SSH port forwarding, we were able to access this service from our machine and obtain a shell on the device.

Finally, by upgrading the ADB session to root, we achieved full privilege escalation and retrieved the root flag, completing the compromise of the target.

Overall, this machine highlights the risks of:

- insecure third-party Android applications exposing debug interfaces,
- sensitive data stored in accessible locations,
- and improperly secured ADB services.

**Thank you for reading!**
