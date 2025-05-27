<<<<<<< HEAD
# DNS_EMAIL_WEBSITE
=======
# Project: Custom DNS Server on VPS with BIND9 for domain <YOUR_DOMAIN> ## Table of
Contents

1. [VPS and operating system selection](#1-select-vps-and-operating-system)
	1.1 [VPS Providers](#11-vps-providers)
	1.2 [CPU architecture and type](#12-architecture-and-type-cpu)
	1.3 [SSH keys and login](#13-key-ssh-i-login)
2. [BIND9 installation and configuration](#2-installation-and-configuration-bind9)
	2.1 [Package Installation](#21-install-packages)
	2.2 [BIND9 file configuration](#22-config-files-bind9)
3. [DNS zone creation](#3-create-dns-zone)
	3.1 [File named.conf.local](#31-file-namedconflocal)
	3.2 [Zone file db.<YOUR_DOMAIN>](#32-file-zone-dbnoinputdev)
4. [Domain Configuration in Porkbun](#4-domain-configuration-in-porkbun)
	4.1 [Domain Registration](#41-register-domains)
	4.2 [Glue Records](#42-glue-records)
	4.3 [Nameservers](#43-nameservers)
5. [Testing and Implementing DNSSEC](#5-testing-and-implementing-dnssec)
	5.1 [Generating KSK and ZSK keys](#51-generate-key)
	5.2 [Zone Signing and RRSIG Generation](#52-signing-zone)
	5.3 [Reconfiguring BIND9 under DNSSEC](#53-bind-i-dnssec)
	5.4 [Entering DS record in Porkbun](#54-record-ds-w-porkbun)
6. [Installing a mail server](#6-install-mail-server)
	6.1 [Postfix-installation and configuration](#61-postfix-installation)
	6.2 [Creating Maildir Directories](#62-maildir)
	6.3 [Dovecot installation](#63-installation-dovecot)
7. [Testing, Security, and Tools](#7-testing-security-and-tools)
	7.1 [Email and IMAP performance tests](#71-tests-mail)
	7.2 [SPF Diagnostics, DKIM, DMARC](#72-diagnostics-spf-dkim-dmarc)
	7.3 [TLS, certificates, Apple Mail](#73-tls-apple-mail)
	7.4 [Typical Problems and Troubleshooting](#74-problems-and-troubleshooting)
	7.5 [Backup and monitoring](#75-backup-i-monitoring)
	7.6 [Send by relayhost (Gmail/SendGrid)](#76-send-by-relayhost)
	7.7 [Errors and Solutions](#77-errors-and-solutions)
	7.8 [Mail client settings](#78-settings-customers-email)
8. [Project Status and Next Steps](#8-project-status-and-next-steps)
	8.1 [What Works](#81-what-works)
	8.2 [What's Next](#82-what's Next)
9. [Website with WordPress](#9-website-with-wordpress)
	9.1 [Apache installation and VirtualHost configuration](#91-install-apache)
	9.2 [Let's Encrypt SSL Certificate Installation](#92-install-ssl)
	9.3 [WordPress installation and database](#93-wordpress-i-mariadb)
	9.4 [Themes & Plugins - Portfolio & Resume](#94-motifs-and-plugins)
	9.5 [WordPress security](#95-security-wordpress)


--


## 1. VPS and operating system selection 

### 1.1 VPS providers

A VPS from Hetzner with the following features was selected:

* x86 architecture
* CPU type: Shared vCPU
* Operating system: Debian 12
* Panel: none - manual configuration via terminal 

### 1.2 CPU architecture and type


A **Shared vCPU** VPS on **x86 (Intel/AMD)** architecture was chosen, which provides adequate power for the DNS server at low cost.


### 1.3 SSH keys and login 

At the stage of creating a VPS instance:

* SSH key pair generated (with 'ssh-keygen' command)
* The public key has been added to the VPS instance
* The first login was through:

```bash
ssh root@<YOUR_IP> ```

* A new user '<your_USER>' was created, added to the 'sudo' group, and the public key was copied to it:

```bash
adduser <your_USER>
usermod -aG sudo <your_USER>
rsync --archive --chown=<your_USER>:<your_USER> ~/.ssh /home/<your_USER> ```


--


## 2 BIND9 installation and configuration

### 2.1

Installing packages

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc dnsutils -y ```'


###2.2 BIND9 file configuration:

File: '/etc/bind/named.conf.options' 

```conf
options {
	directory "/var/cache/bind";
	recursion no;
	allow-query { any; };
	allowtransfer { none; };
	dnssecvalidation auto;
	authnxdomain no;
	listen-on { any;};
	listen-on-v6 { none; };
};
```

--

## 3. Create a DNS zone

### 3.1 File: '/etc/bind/named.conf.local' 

```conf
zone "<YOUR_DOMAIN>" {
	type master;
	file "/etc/bind/zones/db.<YOUR_DOMAIN>";
	allowtransfer { none; };
};
```


### 3.2 File: '/etc/bind/zones/db.<YOUR_DOMAIN>'

```dns
$TTL 86400
@   IN  SOA ns1.<YOUR_DOMAIN>. admin.<YOUR_DOMAIN>. (
            2024051501 ; Serial
            3600       ; Refresh
            1800       ; Retry
            1209600    ; Expire
            86400 )    ; Minimum TTL

    IN  NS  ns1.<YOUR_DOMAIN>.
    IN  NS  ns2.<YOUR_DOMAIN>.

ns1 IN  A   <YOUR_IP>
s2 IN  A   <YOUR_IP>
@   IN  A   <YOUR_IP>
www IN  A   <YOUR_IP>

@     IN MX 10 mail.<YOUR_DOMAIN>.
mail  IN A   <YOUR_IP>

@     IN TXT "v=spf1 a mx <YOUR_IP> -all"
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:postmaster@<YOUR_DOMAIN>"
```


--

## 4. Domain configuration in Porkbun 

### 4.1 Domain registration

* Domain: '<YOUR_DOMAIN_NAME'
* Registrar: **Porkbun**.
* Seamless process, no requirements for country of residence

### 4.2 Glue Records


| Hostname        | IP Address  |
| --------------- | ----------- |
| ns1.<YOUR_DOMAIN> | \<YOUR\_IP> |
| ns2.<YOUR_DOMAIN> | \<YOUR\_IP> |


### 4.3 Nameservers

Changing domain nameservers:

```
ns1.<YOUR_DOMAIN>
ns2.<YOUR_DOMAIN> 
```

--

## 5 Testing and implementing DNSSEC 

### 5.1 Generating keys

```bash
cd /etc/bind/keys
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE <YOUR_DOMAIN>
sudo dnssec-keygen -f KSK -a RSASHA256 -b 2048 -n ZONE <YOUR_DOMAIN>
```


### 5.2 Signature zone 


```bash
sudo dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -c1-16) \
-o <YOUR_DOMAIN> -t db.<YOUR_DOMAIN> /etc/bind/keys/K<YOUR_DOMAIN>.+008+*.key
```


### 5.3 Reconfigure BIND9 

In the 'named.conf.local' file:

```bash
zone "<YOUR_DOMAIN>" {
    type master;
    file "/etc/bind/zones/db.<YOUR_DOMAIN>.signed";
    allow-transfer { none };
};
```

### 5.4 Entering a DS record in Porkbun

* Used 'dig DNSKEY <YOUR_DOMAIN>' to obtain KSK.
* Calculated 'ds-record' using 'dnssec-dsfromkey'.
* After pasting the full 'digest' (64 characters) DNSSEC was accepted and tested at
[https://dnsviz.net](https://dnsviz.net).

--

## 6. installation of mail server

### 6.1 Postfix - installation and configuration

```bash
sudo apt install postfix 
```

Select: 'Internet Site' 
File '/etc/postfix/main.cf':

```bash
home_mailbox = Maildir/
smtpd_tls_cert_file=/etc/ssl/certs/ssl-certsnakeoil.pem 
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may 
smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database= btree:${data_directory}/smtp_scache 
myhostname = mail.<YOUR_DOMAIN>
mydomain = <YOUR_DOMAIN>
myorigin = /etc/mailname
mydestination= $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8 [::1]/128
inet_interfaces = all
inet_protocols= ipv4 
```



### 6.2 Creating Maildir directories

```bash
sudo mkdir -p /home/<your_USER>/Maildir/{cur,new,tmp}
sudo chown -R <your_USER>:<your_USER> /home/<your_USER>/Maildir
```



### 6.3 Dovecot installation

```bash
sudo apt install dovecot-imapd dovecot-pop3d
```


--

## 7. Testing, security and tools 

### 7.1 Email and IMAP performance tests

* Checking the existence of the Maildir directory:

```bash
sudo -u [USER] maildirmake /home/[USER]/Maildir
sudo chown -R [USER]:[USER] /home/[USER]/Maildir 
```


* Shipping Test:

```bash
echo "Test mail"| mail -s "Test message" [USER] sudo ls -l
/home/[USER]/Maildir/new
```


* Dovecot Login Test:

```bash
sudo doveadm auth test [UKRYTO_USER]
```


* IMAP connection test (openssl):

```bash
openssl s_client -connect mail.<YOUR_DOMAIN>:993 
# IMAP: a LOGIN [USER] [PASSWORD] 
```


### 7.2 SPF, DKIM, DMARC diagnostics

* SPF check:
```bash
dig TXT <YOUR_DOMAIN> +short
# Or https://mxtoolbox.com/spf.aspx
```

* DKIM (opendkim) configuration:
```bash
sudo apt install opendkim opendkim-tools
```

* DKIM keys: '/etc/opendkim/keys/<YOUR_DOMAIN>/'

* Sample TXT record:
```
selector201._domainkey IN TXT ("v=DKIM1; k=rsa; p=[PUBLIC_KEY]"; t=s; s=email)
```

* DMARC:

```
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@<YOUR_DOMAIN>" 
```
--

### 7.3 TLS, certificates, Apple Mail

* Installing Let's Encrypt:

  ```bash
  sudo apt install certbot
  sudo certbot certonly --standalone -d mail.<YOUR_DOMAIN> 
  ```

* Dovecot configuration with full chain:

  ```conf
  ssl_cert= </etc/letsencrypt/live/mail.<YOUR_DOMAIN>/fullchain.pem
  ssl_key = </etc/letsencrypt/live/mail.<YOUR_DOMAIN>/privkey.pem
  ```

* Certificate test:


  ```bash
  openssl s_client -showcerts -connect mail.<YOUR_DOMAIN>:993 </dev/null| sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p'
  ```

* Apple Mail problems:

	* If "Verification Impossible" appears, check the certificate chain configuration. 


### 7.4 Common problems and troubleshooting

* Maildir permission errors - fix directory ownership, restart Dovecot.
* No new mail - check logs: 'journalctl -u postfix -n 50', 'journalctl -u dovecot -n 50', '/var/log/mail.log'.
* TLS errors - verify the paths to the certificates and whether the certificate is installed correctly.
* Check port availability on the VPS firewall. 


### 7.5 Backup and monitoring

* Configuration backup (cron/manual example):

  ```bash
  tar czf /root/backup/$(date +%F)_bind.tgz /etc/bind /etc/dovecot /etc/postfix
  ```

* Monitoring and testing:

* Logs: 'journalctl -u postfix -n 50', 'journalctl -u dovecot -n 50'
* External tests: [mxtoolbox.com](https://mxtoolbox.com), [intodns.com](https://intodns.com), [SSL Labs](https://www.ssllabs.com/ssltest/)

### 7.6 Sending mail from VPS with blocked port 25 (relayhost, Gmail/SendGrid) 

#### Problem:

* The VPS provider (e.g., Hetzner) blocks port 25/465 on new servers by default, preventing the direct sending of emails.


#### Solution:

* Send mail via external SMTP relay on port 587 (e.g. Gmail, SendGrid, Mailgun). 

#### Postfix configuration for relay via SendGrid SMTP (port 587):

* **SendGrid domain verification:**.

  * Add the CNAME records shown by SendGrid to the zone file (db.<YOUR_DOMAIN>), remembering the period at the end:

    ```dns
    em<YOUR_NUMBERS> IN CNAME <NO_SENDGRID>.wl005.sendgrid.net. 
    s1._domainkey IN CNAME s1.domainkey.<NO_SENDGRID>.wl005.sendgrid.net.
    s2._domainkey IN CNAME s2.domainkey.<NO_SENDGRID>.wl005.sendgrid.net.
    ```

* Re-sign the zone if you have DNSSEC.
* Do a BIND9 reload.

* **File '/etc/postfix/main.cf' - minimum relayhost configuration:**.


```conf
relayhost= [smtp.sendgrid.net]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps= hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous smtp_tls_security_level = encrypt
smtp_tls_CAfile= /etc/ssl/certs/ca-certificates.crt 
```


* ** File '/etc/postfix/sasl_passwd':**.

  ```
  [smtp.sendgrid.net]:587 apikey:<YOUR_SENDGRID_API_KEY>
  ```

* 'apikey' - literally that word!
* 'YOUR_SENDGRID_API_KEY' - your key generated in SendGrid.

* **Secure file and update maps:**.

  ```bash
  sudo postmap /etc/postfix/sasl_passwd
  sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db 
  ```


* Postfix's **Restart:**.
  
  ```bash
  sudo systemctl restart postfix
  ```


* ** Shipping test:**.
  ```bash
  echo "Test by SendGrid SMTP"| mail -s "Test SendGrid" <YOUR_EMAIL> 
  ```


* **Debugging Shipping:**.

* If the email does not arrive, the logs from '/var/log/mail.log' are crucial to diagnose the problem!



--



## 7.7 Errors and solutions

* Any change to the DNS zone file with DNSSEC enabled = re-signing the zone
* If the TXT (e.g. DKIM) or CNAME record is not visible:
	* you forget to sign the zone,
	* BIND9 reload does not load changes without a signature,
	* no period at the end of the value = error!
	* Typos or wrong place to paste the record.

* Troubleshooting ALWAYS start with logs:

	* '/var/log/mail.log' - Postfix, DKIM, relay
	* 'mailq' - whether there are emails in the queue
	* 'dig TXT' or 'dig CNAME' - whether records are externally visible


--

## 7.8 Mail client settings (Apple Mail/Thunderbird, etc.).

* **IMAP (mail pickup):**.

  * Host: 'mail.<YOUR_DOMAIN>'
  * User: your full email address (e.g. [<YOUR_DOMAIN>](mailto:<YOUR_EMAIL)))
  * Password: Your VPS user password


* **SMTP (shipping):**.

  * Host Name: 'mail.<YOUR_DOMAIN>'.
  * User Name: Your full email address
  * Password: Your VPS user password
  * Port: 587
  * SSL/TLS: YES (STARTTLS)
  * Authentication: Password (Plain)


* DO NOT enter "sendgrid" as the Host Name!

Apple Mail connects to your server, and it forwards to SendGrid (relayhost).

* With this:
  * The emails look like they were sent from your domain.
  * DKIM, SPF, DMARC will be correct (if DNS is OK).
  * Mail-tester.com tests pass.


--


## 8. Project status and next steps


### 8.1 What works



* ✅ VPS configured and secured
* ✅ BIND9 installed and working
* ✅ DNSSEC configured, DS imported into registry
* ✅ Postfix installed and receiving mail
* ✅ Dovecot works, IMAP tested
* ✅ TLS certificate from Let's Encrypt, Apple Mail connects correctly
* ✅ SPF, DKIM, DMARC records implemented
* ✅ External tests pass correctly 

### 8.2 What's next

* Installation of webmail (Roundcube)
* Automatic backup of configurations and emails
* Hardening: fail2ban, firewall, cutting off unused ports
* Deliverability and security tests
* Keeping documentation up to date
* Tests: MXToolbox, intodns.com, SSL Labs

## 9. Website with WordPress

### 9.1 Installing Apache and configuring VirtualHost

Apache2 server was installed along with the required UFW profiles:

```bash
sudo apt update
sudo apt install apache2 -y 
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```


Checked the operation of Apache via the server IP and configured a new site via VirtualHost. Creating a directory structure:


```bash
sudo mkdir -p /var/www/<YOUR_DOMAIN>
sudo echo '<h1>[YOUR_DOMAIN] works</h1>'> /var/www/[YOUR_DOMAIN]/index.html sudo chown -R
www-data:www-data /var/www/<YOUR_DOMAIN>
```


Site configuration file:


Then a directory '/var/www/<YOUR_DOMAIN>' and a test file 'index.html' were created. A configuration file '/etc/apache2/sitesavailable/<YOUR_DOMAIN>.conf' was created:


```apache
<VirtualHost *:80>
    ServerName <YOUR_DOMAIN>
    ServerAlias www.<YOUR_DOMAIN>
    DocumentRoot /var/www/<YOUR_DOMAIN>
    ErrorLog ${APACHE_LOG_DIR}/<YOUR_DOMAIN>_error.log
    CustomLog ${APACHE_LOG_DIR}/<YOUR_DOMAIN>_access.log combined
</VirtualHost>
```


Enabled the configuration with the command 'sudo a2ensite <YOUR_DOMAIN>.conf' and reloaded the server with 'sudo systemctl reload
apache2'.



### 9.2 Installing Let's Encrypt SSL certificate Install certbot and plugin for Apache: 

```bash
sudo apt install certbot python3-certbot-apache -y 
```

Certificate configuration has been started:

```bash
sudo certbot --apache -d <YOUR_DOMAIN> -d www.<YOUR_DOMAIN>
```

Certbot automatically set up HTTPS, enabled redirection from HTTP and saved the certificates in '/etc/letsencrypt/live/<YOUR_DOMAIN>/'.


### 9.3 WordPress installation and database 

The required packages have been installed:

```bash
sudo apt install php php-mysql php-curl php-gd php-mbstring php-xml php-zip php-cli libapache2-mod-php
unzip mariadb-server -y
```


Made sure MariaDB was working:


```bash
sudo systemctl enable mariadb sudo
systemctl start mariadb sudo
mysql_secure_installation```


MariaDB and PHP with the necessary extensions were installed. A database and user were created: 

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'haslo123'; GRANT ALL
PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost'; FLUSH PRIVILEGES;
```


WordPress was downloaded and unpacked:

```bash cd
/tmp
curl -O https://wordpress.org/latest.zip unzip
latest.zip
sudo mv wordpress/* /var/www/<YOUR_DOMAIN>/
```


Permissions have been set and 'wp-config.php' has been copied, with database and flip data. Example of settings:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'password123' );
define( 'DB_HOST', 'localhost' );
```
### 9.4 Themes and plugins - portfolio and resume

The **Astra** theme and plugins were installed:
* Elementor - page builder.
* WPForms - contact forms.
* UpdraftPlus - backups.
* Limit Login Attempts - login protection.

Also installed **Starter Templates** and imported a ready-made resume/portfolio template. 

### 9.5 WordPress security features

Added an entry to 'wp-config.php':

```php define('DISALLOW_FILE_EDIT'', true);
```

File and directory permissions have been changed:

```bash
sudo find /var/www/<YOUR_DOMAIN> -type d -exec chmod 755 {} \;
sudo find /var/www/<YOUR_DOMAIN> -type f -exec chmod 644 {} \;
```

A security plugin has been installed, and 'fail2ban' has been enabled at the server level and access to 'wp-login.php' has been
restricted by security plugins. Added automatic backups and tests of certificate performance by 'certbot renew --dry-run'.

Example of manual backups:
```bash
tar czf /root/backup/wordpress_$(date +%F).tar.gz /var/www/<YOUR_DOMAIN>
```


>>>>>>> e00a919 (My home lab project with configuration of DNS, Email and my own website)
