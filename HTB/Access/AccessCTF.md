## Welcome everyone!

Today, we will explore “*Access*”, a Windows machine from Hack The Box.

![image](/HTB/Access/Access_images/1.png)



## Information Gathering


First, we will perform a full **TCP SYN port scan** to enumerate all possible accessible gates using **Nmap**:

![image](/HTB/Access/Access_images/2.png)

Three services could be identified:

**FTP** (Port 21)

**Telnet** (Port 23)

**HTTP** (Port 80)

Let’s run an aggressive scan on them through the `-A` switch to gather as much information as possible:

![image](/HTB/Access/Access_images/3.png)

It appears that the FTP server accepts to be accessed without credentials, as **anonymous**. Let's verify it directly with the `ftp` utility:

![image](/HTB/Access/Access_images/4.png)

Indeed, that is the case; two folders are accessible: `Backups` and `Engineer`. The first folder contains an interesting file, `backup.mdb`, which appears to be a Microsoft Access database.

The second folder contains an intriguing `Access Control.zip` ZIP file. Both files could be downloaded to our local directory through the `get` command (please ensure to do this under the `binary` FTP mode, otherwise metadata will cause issues as we will attempt to dump the database tables.)

![image](/HTB/Access/Access_images/5.png)



## Initial Access

Trying to extract the contents of the ZIP file prompts us to enter a password, which we don't have at the moment

![image](/HTB/Access/Access_images/6.png)

But we could remember that a database file has been recovered, we could list available database tables with the `mdb-tables -1 backup.mdb` command:

![image](/HTB/Access/Access_images/7.png)

Tables dump has been successful, we could identify an interesting `auth_user` table that may contain valid credentials to grant us an initial access, let’s inspect its content with `mdb-export`:

![image](/HTB/Access/Access_images/8.png)

Indeed, the tool recovered three sets of credentials; we extracted valid credentials for the `admin`, `engineer` and `backup_admin` user.  

Let’s move on the second previously identified Telnet protocol Attempting to access it  through the `telnet` utility prompts for a login:

![image](/HTB/Access/Access_images/9.png)

Attempting to authenticate with the admin and backup_admin credentials is unsuccessful. However, a different message is displayed when we attempt to authenticate as the engineer user:

![image](/HTB/Access/Access_images/10.png)

Credentials are valid, however, the users is not a member of the TelnetClients group that grants shell access over this protocol. We may need to use this authenticated access somewhere else. We identified one last service as active on the target machine: **HTTP**. So let’s attempt to access the website hosted on port 80 from our web browser:

![image](/HTB/Access/Access_images/11.png)

The root page greets us with a static web  page that doesn’t contain anything useful in itself.

We could attempt to perform web directory brute-forcing with **Gobuster** in order to dig deeper in our web enumeration phase; we may be able to authenticate as `engineer` in order to move something. However, nothing useful could be found.

Since we have valid passwords, we could get back to our ZIP file, let’s attempt to use one of the passwords we found against it to check whether any passwords have been reused:

![image](/HTB/Access/Access_images/12.png)

The file could be extracted with the engineer’s password, allowing us to access its content as below:

![image](/HTB/Access/Access_images/13.png)

![image](/HTB/Access/Access_images/14.png)

Inspecting extracted files reveals harcoded credentials for the `security` account:

![image](/HTB/Access/Access_images/15.png)

It strongly suggests that our initial access could be obtained from there; let’s verify if this user account is a member of the *TelnetClients* group able to access the remote system through Telnet:

![image](/HTB/Access/Access_images/fail.png)


It worked! We successfully obtained our initial foothold on the system as the security local user.

The first flag could be found in the `C:\Users\security\Desktop` user’s desktop directory.

!16.![image](/HTB/Access/Access_images/16.png)



## Privilege Escalation

We now need to find a way to escalate our current privileges in order to retrieve the final flag.

Enumerating the current user's privileges through `whoami /priv` doesn’t reveal any unusual privileges that we could leverage to perform our vertical privilege escalation.

![image](/HTB/Access/Access_images/17.png)

Checking for auto logon credentials stored in the WinLogon Windows registry doesn’t reveal anything either. However, inspecting more in depth the filesystem allows us to discover that the ZKTeco ZKAccess software is installed on the target system. This can be seen in the `C:\` root directory. Moreover, it appears that the software is installed as version **3.5**:

ZKTeco ZKAccess Professional 3.5.3 contains an insecure file permissions vulnerability that allows “*Authenticated Users*” group members to replace executable binaries with malicious code through the **(M)** modifications `icacls` permission, which that could be executed by a scheduled task in order to escalate privileges. 

This vulnerability is also known as CVE-2016-20025.

However, in this room, the solution was much easier: I forgot to check for saved credentials in the *Windows Credentials Manager* through `cmdkey /list`.

![image](/HTB/Access/Access_images/19.png)

It appears that the privileged `Administrator` user has stored credentials on the local machine…

Although this misconfiguration is pretty basic, the way to leverage it in the Telnet shell makes the privilege escalation a bit more tricky; `cmd.exe` executable won’t return a shell as Administrator due to the unstable TTY, using a `nc.exe` (Netcat) isn’t conclusive either due to system restrictions and PowerShell scripts are strictly blocked. 

The remaining solution is a `.bat` script (`cmd.exe` syntax) that will use Netcat through its `runas` execution in order to initiate a reverse shell to our attacker machine by using administrator‘s saved with the `/savecred` option.

This way, scripts could be executed in a stealthier way, making the `nc.exe` executable effective.

 Make sure that you have set up your Netcat listener through `nc -lvnp PORT` as below:

![image](/HTB/Access/Access_images/20.png)

![image](/HTB/Access/Access_images/21.png)

We successfully escalated to the administrator user through saved credentials, a critically bad configuration from a security perspective that results in a total compromise of the target system.

The root flag could be fetched in the `C:\Users\administrator/Desktop` administrator’s desktop directory:

![image](/HTB/Access/Access_images/22.png)

For the bonus question, download `mimikatz` on the target machine and run `sekurlsa::logonpasswords` in order to dump the administrator’s password stored by the *Windows Data Protection API*.



## Conclusion

Although I had hoped for much more from the privilege escalation phase, I enjoyed completing this room and learnt a lot concerning several Linux tools. Starting with a FTP share that was accessible without credentials, we recovered sensitive files that couldn’t be accessible by an unauthenticated users; this allowed us to chain several credentials, which ultimately enabled us to establish a foothold on the target system as the `security` user. Performing a local enumeration of the filesystem led us to identify a *Windows Credentials Manager* configuration set for the privileged administrator user that allowed us to use a `.bat` script that spawned a reverse shell through `runas.exe` with administrator’s saved credentials, evoked with `/savecred`.

This machine demonstrates how multiple low-severity security misconfigurations could be underestimated and chained to achieve a full compromise.

**Thank you for reading!**
