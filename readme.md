# roger-skyline-1

# INTRODUCTION

Roger-skyline-1 let you install a Virtual Machine, discover the basics about system and network administration as well as a lots of services used on a server machine.

# TABLE OF CONTENTS

- [Installation](#installation)
  * [Adding non-root user](#adding-non-root-user)
  * [Network configuration](#network-configuration)
  * [SSH connection](#ssh-connection)
- [Network Security](#network-security)
  * [Firewall](#firewall)
  * [Fail2ban](#fail2ban)
    + [Unbanning myself](#unbanning-myself)
  * [Portsentry](#portsentry)
- [Monitoring & Updates](#monitoring---updates)
  * [Mail](#mail)
  * [Scripts](#scripts)
- [Resource Efficiency](#resource-efficiency)
  * [Disabled nonmandatory services](#disabled-nonmandatory-services)
- [SSL](#ssl)
- [DEPLOYMENT AUTOMATION](#deployment-automation)
  * [Git & GitHub access token](#git---github-access-token)
  * [SSH agent forwarding](#ssh-agent-forwarding)
  * [Deployment with GitHooks](#deployment-with-githooks)
    + [How it works](#how-it-works)

# INSTALLATION

Installed Debian 11, disk size 8 GB and 4.2 GB home partition on VirtualBox VM.
Started the VM, logged in as root.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image08.png?raw=true)
If we add these together we get 8033 MB ~ 8 GB. However the designated 4.2 GB partiton seems to be too small at 3.8 GB.

I installed parted, a tool to analyze and resize partition tables.
![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image09.png?raw=true)
Here we see that the size of the partition is indeed 4.2 GB, However the size of the entire disk is now listed as 8.590 MB which is slightly more than previously listed. When looked outside of VM the disk size is listed at 8.59 GB. This weird because I spcefied 8.0 GB for the disk size at the installation of the VM.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image10.png?raw=true)
Then if we go to the Virtual box  and check the settings for the disk, it is listed as 8,00 GB so apparently the 590 MB is needed for configuration  of the dislk file.

```
apt-get update
apt-get upgrade
```

## Adding non-root user

```
adduser kpolojar
apt-get install sudo
usermod -aG sudo kpolojar
```
You can assume that everything after this point will be done logged in as kpolojar.

## Network configuration

Changed VM network setting NAT -> Bridged

### Setting up static ip address

```
sudo nano /etc/network/interfaces
```

```
#Primary network interface
auto enp0s3
```
```
sudo nano /etc/network/interfaces.d/enp0s3
```
```
iface enp0s3 inet static
	address 10.13.199.214
	netmask 255.255.255.252 (/30)
	gateway 10.13.254.254
```

## SSH connection

Changed the ssh default port to 5555

```
sudo nano /etc/ssh/sshd_config
```

Created key pair with

```
sudo ssh-keygen
```

Changed SSH settings to allow for password connection.
```
sudo nano /etc/ssh/sshd_config
```
uncomment PasswordAuthentication

Connected from iMac to virtual machine with SSH using password (no need for username because same on both systems)
```
ssh 10.13.199.214 -p 5555
```
Copied key to VM
```
ssh-copy-id
```

Changed settings back to refuse password connection
```
sudo nano /etc/ssh/sshd_config
```
comment out PasswordAuthentication
uncommment PubkeyAuthentication

# NETWORK SECURITY

## Firewall

Installed ufw

Added rules to allow tcp at ports 5555 and 80 for SSH and HTTP respectively, 443 for https (udp & tcp)

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 5555/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443
sudo ufw enable
```

## Fail2ban

https://www.garron.me/en/go2linux/fail2ban-protect-web-server-http-dos-attack.html

https://www.xmodulo.com/configure-fail2ban-apache-http-server.html

```
sudo nano /etc/fail2ban/jail.conf
```

```
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/varnish/access.log
maxretry = 300
findtime = 300
#ban for 5 minutes
bantime = 600
action = iptables[name=HTTP, port=http, protocol=tcp]

# detect password authentication failures
[apache]
enabled  = true
port     = http,https
filter   = apache-auth
logpath  = /var/log/apache*/*error.log
maxretry = 6

# detect potential search for exploits and php vulnerabilities
[apache-noscript]
enabled  = true
port     = http,https
filter   = apache-noscript
logpath  = /var/log/apache*/*error.log
maxretry = 6

# detect Apache overflow attempts
[apache-overflows]
enabled  = true
port     = http,https
filter   = apache-overflows
logpath  = /var/log/apache*/*error.log
maxretry = 2

# detect failures to find a home directory on a server
[apache-nohome]
enabled  = true
port     = http,https
filter   = apache-nohome
logpath  = /var/log/apache*/*error.log
maxretry = 2
```

Create the filter
```
sudo nano  /etc/fail2ban/filter.d/http-get-dos.conf
```

```
[Definition]

failregex = ^<HOST> -.*"(GET|POST|PUT|DELETE).*
ignoreregex =
```

## Portsentry

https://en-wiki.ikoula.com/en/To_protect_against_the_scan_of_ports_with_portsentry

Added the iMac ip address to the ignore list.
```
sudo nano /etc/portsentry/portsentry.ignore.static
```
```
sudo nano /etc/default/portsentry
```
Asvanced mode for TCP and UDP -protocols.
```
TCP_MODE="atcp"
UDP_MODE="audp"
```
We opt for a blocking of malicious persons through iptables.
```
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```

## Testing network security

### slowloris

Tested fail2ban with slowloris (Launched from other VM)
```
slowloris 10.13.199.214
```
monitoring the log file shows thefailed DOS -attempts. 
```
sudo tail -f /var/log/fail2ban.log
```

### Trying to SSH directly to root (forbidden)

```
ssh root@10.13.199.214
```

### Unbanning myself

So I had to go and manually unban myself. 

Bans are listed in file:
```
/etc/fail2ban/jail.local
```
Clearing the file will end all current bans without affecting the filters.

### List open ports

https://www.cyberciti.biz/faq/how-to-check-open-ports-in-linux-using-the-cli/
```
sudo netstat -tulpn | grep LISTEN
```

### Port scan with nmap (other computer in same network)

```
brew install nmap
nmap -PN -sS 10.13.199.214
```

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image05.png?raw=true)

Notice that the ufw needs to be stopped because otherwise it would block the portscans instead.
```
sudo cat /var/log/syslog
```
![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image06.png?raw=true)

The first scan reveals the open ports for HTTP, HTTPS and SSH. Howerever the attacker gets banned and on the second try. All of the ports are hidden.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image07.png?raw=true)

# MONITORING & UPDATES

## Mail

All in all configuring the mail service was the biggest hurdle in the whole project. This is mostle due to unclear instructions from the subject combined with the design choice of not allowing root to receive email in Debian. I'm  not happy with my solution to the problem, but I fail to see any better alternative.

```
sudo apt-get install mailutils
sudo apt-get install postfix
```

Delivering mail to the root has been disabled on any Debian distribution.
```
4.3. No deliveries to root!
No Exim 4 version released with any Debian OS can run deliveries as root. If you don't redirect mail for root via /etc/aliases to a nonprivileged account, the mail will be delivered to /var/mail/mail with permissions 0600 and owner mail:mail.
This redirection is done by the mail4root router which is last in the list and will thus catch mail for root that has not been taken care of earlier.
```
https://www.linuxquestions.org/questions/debian-26/root-not-getting-mail-4175423619/

This has been done because receiving mail as root is security vulnerability and bad practise.

https://www.howtogeek.com/124950/htg-explains-why-you-shouldnt-log-into-your-linux-system-as-root/

This is why we have the less privileged sudo account for system administrator in the first place. Thus mails are sent to root and then redirected to sudo user kpolojar. To be able to read the mail on the root account I replaced the /var/mail/root with a soft link to kpolojar mail box. This is done purely to comply with the subject and I wouldn't recommend it as a solution otherwise. I'm not sure if tis breaks some hidden functionality that relies on the /var/mail/root -file. As an additional headache, all mail directed to account kpolojar also becomes available to root.
```
sudo rm /var/mail/root
sudo ln -s /var/mail/kpolojar /var/mail/root
```
https://serverfault.com/questions/890429/debian-exim4-eading-local-mail-with-the-mail-command-as-root

## Scripts

Created a script that updates all packages.
```
sudo nano /scripts/update.sh
```

With following contents

```
!/bin/bash
sudo apt-get update -y >> /var/log/update_script.log
sudo apt-get upgrade -y >> /var/log/update_script.log
```

Created the backup file

```
sudo nano /etc/crontab.backup
sudo chmod 755 /etc/crontab.backup
```

Created a  script that compares /etc/crontab and /etc/crontab.backup, if there is diff, creates new backup and send mail to root (see [Mail](#mail)).
```
sudo nano scripts/cron_monitor.sh
```
Contents
```
#!/bin/bash
DIFF=$(diff /etc/crontab.backup /etc/crontab)
cat /etc/crontab > /etc/crontab.backup
if [ "$DIFF" != "" ]; then
        echo "crontab changed, sending mail to root" | mail -s "crontab updated" root
fi
```

Created a cronjob to run the scripts at correct times. 
(Note that this is different file from /etc/crontab so it wont trigger the script itself)
sudo crontab -e

```
# Update packages every sunday at 4 AM and at reboot.
0 4  * *  0 sudo sh /scripts/update.sh
@reboot sudo sh /scripts/update.sh

# Monitor crontab and send mail in case of change at midnight
0 0 * * *  sudo sh /scripts/cron_monitor.sh
```

# RESOURCE EFFICIENCY

## Disabled nonmandatory services

List enabled services

```
systemctl list-unit-files --state=enabled
```

Disable unnescessary services

```
kpolojar@debian:~$ sudo systemctl disable console-setup.service
Removed /etc/systemd/system/multi-user.target.wants/console-setup.service.
kpolojar@debian:~$ sudo systemctl disable cryptdisks-early.service
Unit /lib/systemd/system/cryptdisks-early.service is masked, ignoring.
kpolojar@debian:~$ sudo systemctl disable cryptdisks.service
Unit /lib/systemd/system/cryptdisks.service is masked, ignoring.
kpolojar@debian:~$ sudo systemctl disable e2scrub_reap.service
Removed /etc/systemd/system/default.target.wants/e2scrub_reap.service.
kpolojar@debian:~$ sudo systemctl disable ifupdown-wait-online.service
kpolojar@debian:~$ sudo systemctl disable keyboard-setup.service
Removed /etc/systemd/system/sysinit.target.wants/keyboard-setup.service.
kpolojar@debian:~$ sudo systemctl disable nftables.service
kpolojar@debian:~$ sudo systemctl disable apt-daily-upgrade.timer
Removed /etc/systemd/system/timers.target.wants/apt-daily-upgrade.timer.
kpolojar@debian:~$ sudo systemctl disable apt-daily.timer
Removed /etc/systemd/system/timers.target.wants/apt-daily.timer.
kpolojar@debian:~$
```

This is what we are left with.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image01.png?raw=true)

# SSL

https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-debian-9

Creating certificate

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

```
Country Name (2 letter code) [AU]:FI
State or Province Name (full name) [Some-State]:Uusimaa
Locality Name (eg, city) []:Helsinki
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Hive
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Kimmo Polojarvi
Email Address []:kpolojar@debian.debbie
```

```
sudo nano /etc/apache2/conf-available/ssl-params.conf
```

```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
# Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```

sudo nano /etc/apache2/sites-available/default-ssl.conf

```
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin kpolojar@debian.debbie
                ServerName 10.13.199.214

                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
      
                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>
        </VirtualHost>
</IfModule>

```
sudo nano /etc/apache2/sites-available/000-default.conf
```
<VirtualHost *:80>
        . . .

        Redirect permanent "/" "https://10.13.199.214/"

        . . .
</VirtualHost>
```

Navigating to 10.13.199.214 with browser warns about unknown certificate.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image03.png?raw=true)

Clicking the "view certificate" shows you the newly created SSL -certificate.

![Image](https://github.com/N1GH7C4P/roger-skyine-1/blob/documented/image04.png?raw=true)

"Accept the risk and continue" Takes you to the website hosted on the server, by default the Apache landing page.

# DEPLOYMENT AUTOMATION

## Git & GitHub access token 

sudo apt-get install git

We need to setup GitHub access token to be able to fetch changes from my personal github repo.

Created GitHub access token
https://www.edgoad.com/2021/02/using-personal-access-tokens-with-git-and-github.html

To cache your GitHub credentials for HTTPS access follow this tutorial.

https://docs.github.com/en/get-started/getting-started-with-git/caching-your-github-credentials-in-git

## SSH agent forwarding

This is needed to use GitHub commands through SSH -connection without providing credentials each time.

https://docs.github.com/en/developers/overview/using-ssh-agent-forwarding

Make sure that you have the same SSH keys on your VM and local machine.
Otherwise the VM doesn't have permission to fetch from GitHub.

Then add GitHub to your known hossts file.
https://serverfault.com/questions/856194/securely-add-a-host-e-g-github-to-the-ssh-known-hosts-file

ssh-keyscan -t rsa github.com 

```
# github.com:22 SSH-2.0-babeld-f3847d63
2048 SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8 github.com (RSA)
```

Append here:
```
~/.ssh/known_hosts
```

## Deployment with GitHooks

### How it works
You are developing in a working-copy on your local machine, lets say on the master branch. Most of the time, people would push code to a remote server like github.com or gitlab.com and pull or export it to a production server. Or you use a service like deepl.io to act upon a Web-Hook that's triggered that service.

But here, we add a "bare" git repository that we create on the production server and pusblish our branch (f.e. master) directly to that server. This repository acts upon the push event using a 'git-hook' to move the files into a deployment directory on your server. No need for a midle man.

This creates a scenario where there is no middle man, high security with encrypted communication (using ssh keys, only authorized people get access to the server) and high flexibility tue to the use of .sh scripts for the deployment.

https://gist.github.com/noelboss/3fe13927025b89757f8fb12e9066f2fa

Follow the tutorial.

https://towardsdatascience.com/how-to-create-a-git-hook-to-push-to-your-server-and-github-repo-fe51f59122dd

If you get permission problems do this.
https://stackoverflow.com/questions/14127255/remove-git-index-lock-permission-denied

Now when you do
```
git push server master
```
The changes are pushed straight to the VM server amd you can see the change happen in browser.

https://10.13.199.214/

# CHECKSUM

shasum ~/goinfre/debbie.vdi
d480b8ec3299920e6bc93bebf8de5faab1f9adfc