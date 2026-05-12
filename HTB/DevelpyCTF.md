**Welcome to my very first CTF walkthrough: Devel from Hack The Box.**

Previously, we played around INE’s Labs. Now, it’s time for serious business! I decided not to cover some CTFs like **Lame** and **Cap**, as I did not find it interesting to explain how to run a simple prebuilt username map script against a service in order to gain full access. However, I will start with a fairly easy CTF so I can track my progress and learn to write them properly.

Let’s get started!



![image](/HTB/DevelpyCTF_images/1.png)


(For the sake of the experience, I preferred to complete this machine in **“Adventure Mode”**, so please note that this is simply the approach I took, and that it could probably be done differently.)

## Information Gathering

We can start from an **Nmap** scan :


![image](/HTB/DevelpyCTF_images/2.png)


We discovered two exposed services:

**- FTP (21)**

**- HTTP (80)**

We can infer that the target is running **Windows**, as IIS is commonly deployed in Windows environments.



## Initial Access

The target exposes an **FTP** service, which is interesting. Can we interact with it using **anonymous authentication**?

![image](/HTB/DevelpyCTF_images/3.png)


We can! Anonymous authentication is enabled on the FTP server. Let’s inspect its contents with `ls`:


![image](/HTB/DevelpyCTF_images/4.png)


It looks like this FTP server hosts content for a website. We can verify this since `welcome.png` appears to be accessible through the web server:

![image](/HTB/DevelpyCTF_images/5.png)


This confirms that we can access these hosted files from our web browser. Are we also able to upload files through FTP?

![image](/HTB/DevelpyCTF_images/6.png)


Uploads are also permitted! The system administrator has made several dangerous misconfigurations. An unauthenticated user can upload via FTP and access files via their web browser. This can be very dangerous as an attacker can leverage this misconfiguration and execute arbitrary code on the machine without authentication. We can try to craft an **msfvenom** payload and upload it to the target and then trigger it via HTTP.

![image](/HTB/DevelpyCTF_images/7.png)


The **x64 Meterpreter** payload failed during execution, likely due to process architecture or environmental constraints. For better compatibility and reliability, a **32-bit** (`x86`) `msfvenom` payload was used instead.

Since the target web server is running Microsoft IIS, an ASP.NET payload using the `.aspx` extension can be uploaded. IIS natively processes ASPX files through the ASP.NET engine, allowing server-side code execution.

We can now upload the generated payload :


![image](/HTB/DevelpyCTF_images/8.png)


We set up a listener using the `multi/handler` module in Metasploit with the appropriate payload, local IP address, and listening port:

![image](/HTB/DevelpyCTF_images/9.png)


Now, it’s time to trigger the payload through the web browser :


![image](/HTB/DevelpyCTF_images/10.png)


And we gained access to the machine with a **Meterpreter session**!

## Post-Exploitation

We can check our privileges with these built-in Meterpreter commands :

![image](/HTB/DevelpyCTF_images/11.png)


The `SeImpersonatePrivilege` privilege is particularly interesting. We can potentially leverage this to **impersonate a token** from a privileged user. Let’s use **Incognito**, a built-in Meterpreter plugin. We can list the available user tokens on the system:


![image](/HTB/DevelpyCTF_images/12.png)


We do not appear to be able to retrieve any token. We can try to escalate our privileges with **PrintSpoofer**, a tool we already used in a previous lab which is very useful during the post-exploitation phase. We need to upload a compatible version as the target environment appears to be more stable with 32-bit binaries (the architecture can be verified with `sysinfo` in Meterpreter or `systeminfo` via the cmd shell).

We start a Python HTTP server and transfer the executable to the target manually:



![image](/HTB/DevelpyCTF_images/13.png)


 The program allows us to spawn a privileged prompt as attempts to exploit the `SeImpersonatePrivilege` privilege in our case. To do this, we can run `.\PrintSpoofer32.exe -i -c cmd` :
 

![image](/HTB/DevelpyCTF_images/14.png)


It didn’t work… multiple attempts with other tools like **JuicyPotato** didn’t help either. Since we obtained an interesting privilege, we can link our current session to a Metasploit module called **local_exploit_suggester** which can be used to locate local privilege escalation techniques, therefore listing every module that could potentially elevate our current privileges.

We set the Meterpreter session we obtained :


![image](/HTB/DevelpyCTF_images/15.png)


Run it:


![image](/HTB/DevelpyCTF_images/16.png)


We can spot many modules we can use in order to gain full access to the machine. After trying **UAC bypass** techniques or various CVEs, we find the **ms16_075_reflection_juicy**. We configure the module accordingly and run it:


![image](/HTB/DevelpyCTF_images/17.png)

We now have **SYSTEM privileges**! We should be able to retrieve the `user.txt` and `root.txt` flags now :


![image](/HTB/DevelpyCTF_images/18.png)

## Conclusion

This straightforward machine demonstrates how dangerous insecure service configurations can be. By combining anonymous FTP access, arbitrary file upload, and local privilege escalation techniques, we achieved SYSTEM privileges on the target.

It also highlights the importance of proper service hardening and the usefulness of Metasploit during both exploitation and post-exploitation phases, particularly for payload handling and privilege escalation enumeration.

Thank you for reading! Any feedback is appreciated.
