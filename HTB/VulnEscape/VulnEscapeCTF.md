## Welcome everyone!

Today, we will cover “VulnEscape”, an **HackTheBox** easy-level machine.

Without further ado, let’s get started!

![image](/HTB/VulnEscape/VulnEscape_images/1.png)

(Please ensure that you have added the following line `TARGET_IP     vulnescape.htb` to your local `/etc/hosts` file)

## Information Gathering

We can start with an aggressive nmap scan to obtain comprehensive results about the target:

![image](/HTB/VulnEscape/VulnEscape_images/2.png)

We can identify only one service running on the target system:

**RDP** (3389)

We can try to interact with the service without credential in order to obtain more information with the following command:

`xfreerdp /v:TARGET_IP /cert:ignore /sec:tls`

![image](/HTB/VulnEscape/VulnEscape_images/3.png)

We are greeted by a FreeRDP windows that indicates us to login as **KioskUser0** without password, interesting. Let’s do this, we will run the following command to authenticate us as **KioskUser0** without providing any password:

`xfreerdp /u:KioskUser0 /p: /v:TARGET_IP`

![image](/HTB/VulnEscape/VulnEscape_images/4.png)

Now he have gained access to the target system, but it is very restricted since it is running Kiosk Mode. We cannot run cmd to interact with the command line and obtain more information about the target. However, we can access the Edge browser. To do this we can press the *Windows* button on our keyboard to open the *start menu*. We can then input “`msedge.exe`” and hit *Enter* to interact the web browser.

We can access the file system within the browser by using the following URL:

`file:///C:/`

![image](/HTB/VulnEscape/VulnEscape_images/7.png)

From there, we can retrieve the first flag located in the `C:/Users/kioskUser0/Desktop/` directory, which can be accessed by the following URL:

`file:///C:/Users/kioskUser0/Desktop/user.txt`

![image](/HTB/VulnEscape/VulnEscape_images/8.png)

Since the `msedge.exe` binary is **allowed** to be executed, if we rename the `cmd.exe` to `msedge`, we will be able to use it!

![image](/HTB/VulnEscape/VulnEscape_images/9.png)

![image](/HTB/VulnEscape/VulnEscape_images/10.png)

![image](/HTB/VulnEscape/VulnEscape_images/11.png)

It works! We can now navigate interactively through the file system.

We can find an interesting file in the `C:\_admin` directory:

![image](/HTB/VulnEscape/VulnEscape_images/12.png)


It looks like it contains the encrypted password for the admin user. We can learn as well that Remote Desktop Plus is installed on the machine, which can be confirmed by accessing it via `file:///C://Program%20Files%20(x86)/Remote%20Desktop%20Plus/`

![image](/HTB/VulnEscape/VulnEscape_images/13.png)

We can rename it as we did with `cmd.exe` to execute it:

![image](/HTB/VulnEscape/VulnEscape_images/14.png)

We can use [BulletPassView](https://www.nirsoft.net/utils/bullets_password_view.html) to decrypt the password directly. Once downloaded and transferred to the target. We will copy the .xml file and import it into RDP in order to retrieve it in BulletPassView:

![image](/HTB/VulnEscape/VulnEscape_images/15.png)

I copied the `profiles.xml` file into the `Downloads` directory and imported it. After clicking on “Edit”, we can now run BulletPassView to retrieve the hidden password:

![image](/HTB/VulnEscape/VulnEscape_images/16.png)

![image](/HTB/VulnEscape/VulnEscape_images/17.png)

We successfully retrieved the plaintext password: `Twisting3021`. But it seems that the admin user is not allowed to log in via RDP directly. We use `runas` to authenticate as the admin user, then perform privilege escalation. We will start by running `powershell` in order to use `runas`. We will then execute `runas /user:admin powershell`. We will be prompted to enter the admin password, which we found earlier. 

We abuse a UAC auto-elevation mechanism via Event Viewer to spawn an elevated PowerShell session. Within the path bar we just enter **`powershell`** which gives us an elevated powershell session in **`high mandatory level`**:

![image](/HTB/VulnEscape/VulnEscape_images/18.png)

![image](/HTB/VulnEscape/VulnEscape_images/19.png)

![image](/HTB/VulnEscape/VulnEscape_images/20.png)

Since we executed powershell in an elevated environment, we are now able to retrieve the `root.txt` flag in the Administrator’s Desktop directory!

![image](/HTB/VulnEscape/VulnEscape_images/21.png)

Challenge completed!

![image](/HTB/VulnEscape/VulnEscape_images/22.png)

## Conclusion

“VulnEscape” is a great example of a heavily restricted Windows environment where the initial foothold is not achieved through traditional exploitation, but rather through **misconfiguration and weak access control in a kiosk-style setup**.

Starting from a single exposed service, RDP, we were able to gain access using a guest-like account. Although the session was heavily locked down, the ability to use Microsoft Edge provided a critical pivot point, allowing us to explore the underlying file system and retrieve the first flag through browser-based access.

The challenge then shifted from execution to **information discovery and abuse of allowed applications**. By leveraging the installed tools and misconfigurations, we were able to extract sensitive configuration data and recover credentials stored for Remote Desktop Plus. This highlights a common real-world issue: sensitive credentials being stored insecurely in application profiles.

With valid credentials in hand, privilege escalation was achieved through credential reuse combined with weak administrative separation. The final step involved leveraging built-in Windows mechanisms to bypass restrictions and obtain elevated access, ultimately leading to the retrieval of the root flag.

Overall, this machine emphasizes several key lessons:

- Attack surfaces can exist even in “locked-down” environments like kiosk sessions
- Browser access alone can be enough to pivot and gain system-level insight
- Poorly protected configuration files often expose critical credentials
- Credential reuse remains one of the most effective privilege escalation vectors
- Built-in Windows tools can be abused for privilege escalation when misconfigured

“VulnEscape” demonstrates that sometimes the hardest part of a system is not exploitation, but understanding how restrictions can still be bypassed through legitimate functionality.

Thank you for reading!
