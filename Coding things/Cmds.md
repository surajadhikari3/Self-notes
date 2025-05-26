### 📁 File & Directory Operations

| Command | Description                                                 |
| ------- | ----------------------------------------------------------- |
| `pwd`   | Present working directory                                   |
| `ls`    | List files and folders                                      |
| `cd`    | Change directory                                            |
| `mkdir` | Make directory                                              |
| `rmdir` | Remove empty directory                                      |
| `rm`    | Remove files or folders (recursively for non-empty folders) |
| `cp`    | Copy files/folders (recursive for folders)                  |
| `mv`    | Move or rename file/folder                                  |
| `ln`    | Create symbolic or hard link                                |
| `cat`   | Display file content                                        |
| `less`  | View file content with navigation                           |
| `echo`  | Echo text or variables to console                           |
| `tree`  | Show directory structure in tree format                     |

---

### 🔍 Search & Text Processing

| Command | Description                                      |
| ------- | ------------------------------------------------ |
| `grep`  | Search text in files                             |
| `find`  | Search files/folders by name, size, or timestamp |
| `cut`   | Split text by delimiters                         |
| `sort`  | Sort lines of text                               |
| `uniq`  | Filter duplicate records                         |
| `tail`  | Display last lines of a file                     |

---

### 🔒 Permissions & Ownership

| Command  | Description                                                                                                              |
| -------- | ------------------------------------------------------------------------------------------------------------------------ |
| `chmod`  | Change file/directory permissions                                                                                        |
| `chown`  | Change ownership                                                                                                         |
| `chgrp`  | Change group                                                                                                             |
| `chattr` | Set/unset file attributes (It does not work on mac as linux used the ext file format and mac uses different file format) |
| `umask`  | Set default permissions for new files                                                                                    |

---

### 👤 User Management

| Command    | Description                |
| ---------- | -------------------------- |
| `id`       | Show current user ID       |
| `adduser`  | Create a new user          |
| `addgroup` | Create a user group        |
| `usermod`  | Modify a user account      |
| `passwd`   | Change user password       |
| `groups`   | Show user groups           |
| `who`      | Show logged-in users       |
| `w`        | Show users and system load |

---

### ⚙️ Process Management

| Command | Description                        |
| ------- | ---------------------------------- |
| `ps`    | Show running processes             |
| `kill`  | Kill a process (use `-9` to force) |
| `top`   | Display real-time processes        |

---

### 🌍 Environment Management

| Command  | Description                  |
| -------- | ---------------------------- |
| `env`    | Show environment variables   |
| `set`    | Windows equivalent for `env` |
| `export` | Set environment variable     |

---

### 🌐 Networking & Remote Access

| Command       | Description                          |
| ------------- | ------------------------------------ |
| `ssh-keygen`  | Generate SSH keys                    |
| `ssh-copy-id` | Copy SSH key to remote server        |
| `ssh`         | Open SSH session                     |
| `scp`         | Securely copy files                  |
| `wget`        | Download files from web              |
| `curl`        | HTTP command-line tool               |
| `nc`          | Network tool (netcat)                |
| `ping`        | Check host reachability              |
| `ifconfig`    | Configure network interfaces         |
| `ip`          | Show/manipulate IP info (Linux only) |
| `hostname`    | Show system hostname                 |
| `traceroute`  | Trace network path to host           |
| `mtr`         | Trace route with statistics          |
| `nslookup`    | Query DNS servers                    |
| `netstat`     | Network statistics                   |
| `host`        | DNS lookup tool                      |
| `tcpdump`     | Capture network traffic              |
| `dig`         | DNS lookup and diagnostics           |

---

### 💾 Disk Usage & Archiving

| Command | Description                         |
| ------- | ----------------------------------- |
| `df`    | Show disk space usage               |
| `du`    | Estimate file/directory space usage |
| `tar`   | Archive utility                     |
| `gzip`  | Compress files                      |
| `unzip` | Extract zip archives                |

---

### 🔣 Special Characters

| Symbol | Description                 |
| ------ | --------------------------- |
| `.`    | Current directory           |
| `..`   | Parent directory            |
| `~`    | Home directory              |
| `/`    | Root or directory separator |
| `\\`   | Escape character (Windows)  |
| `>`    | Redirect output (overwrite) |
| `>>`   | Redirect output (append)    |
| `<`    | Redirect input              |
| `:`    | Used in path/env variables  |
| `;`    | Command separator           |
| `\n`   | New line                    |
| `\r`   | Carriage return             |

Links(Hardlinks and Softlinks)

![[Pasted image 20250524204550.png]]


## **Symbolic `chmod` Syntax**


`chmod [who][operator][permission] file`

|Who|`u` (user), `g` (group), `o` (others), `a` (all)|
|---|---|
|Operator|`+` (add), `-` (remove), `=` (set exact)|

### Example Commands

| Command               | Explanation                                        |
| --------------------- | -------------------------------------------------- |
| `chmod +x script.sh`  | Give execute permission to all (user/group/others) |
| `chmod u+x file.txt`  | Add execute permission for user only               |
| `chmod go-w file.txt` | Remove write permission from group and others      |
| `chmod a=r file.txt`  | Set read-only for all                              |
chmod values

execute(x) ->  1
writer (w) -> 2
read  (r) -> 4
Total --> 7

| Scenario                         | Use `netcat` (`nc`)                            | Use `netstat`                           |
| -------------------------------- | ---------------------------------------------- | --------------------------------------- |
| Check if a port is open          | ✅ `nc -zv host port`                           | 🚫 (cannot check reachability directly) |
| Send/receive data between hosts  | ✅ `nc -l` (listener) + `nc host port` (client) | 🚫 Not used for sending/receiving data  |
| Scan for open ports              | ✅ `nc -zv host 20-80`                          | 🚫                                      |
| Monitor open network connections | 🚫                                             | ✅ `netstat -tulnp`                      |
| Show listening services          | 🚫                                             | ✅ `netstat -an                          |

netcat can be used to run the server and client too..
### 🆚 **Quick Analogy**

| **Analogy** | `netcat` is like a **Swiss army knife** for network I/O — it can act like a phone call (send/receive).<br>`netstat` is like a **network auditor** — it only reports what connections already exist. |

---

### ✅ Conclusion

- Use **`netcat`** when you want to **test, debug, or interact** with network services.
    
- Use **`netstat`** when you want to **monitor** or **inspect** network connections and their statuses.


Tar Vs gzip..

![[Pasted image 20250526100353.png]]

tar --> Combined the multiple file into one archive folder 

gzip --> Then compress the combined folder using gzip 


So use the combination of it :

Or else we can directly use the tar with the extra aruments like given below 


`tar -czvf backup.tar.gz folder-name`

use the --help to see the arguments where z is to compress which gzip does by default....

Chattr cmd 

It does not work in the mac as it work with ext4 file system
There is chflags cmd equivalent to the chattr cmd in mac os..

| Linux (`chattr`) | macOS (`chflags`)     | Description         |
| ---------------- | --------------------- | ------------------- |
| `chattr +i file` | `chflags uchg file`   | Make file immutable |
| `chattr -i file` | `chflags nouchg file` | Remove immutability |
