# hlstats-centos-7-install-guide
Install guide for HLStats 1.65 on CentOS 7

First, create a new virtual machine with the CentOS 7.X virtual machine.  Use the CentOS 7 DVD that you can download from [https://wiki.centos.org/Download](https://wiki.centos.org/Download)

You don't need a lot of horse power for this VM so 10GB of disk and 1GB of memory is a good place to start.

Base Environment: `Basic Web Server`

Once the OS is installed and boots for the first time, install VMWare tools.  That process goes like this
```bash
mount /dev/cdrom /mnt
cd ~
tar xvfz /mnt/VMWare<whatever date>.gz
cd vmware-tools/distrib
./vmware-install.pl
Just accept all the defaults
```
Next do an initial update with `yum update` and install all the available updates and `reboot` the machine

Next we need to install MariaDB (MySQL)
```bash
yum install mariadb mariadb-server
systemctl start mariadb
systemctl enable mariadb
```

Some other pre-requisites that we need
```bash
yum install gcc
yum install php php-mysqli
```

Make sure Apache starts on boot
```bash
systemctl start httpd
systemctl enable httpd
```

Create a user to run the HLStats daemon
```bash
useradd hlstats
```

Set the root MySQL user password
```bash
mysqladmin -u root password <password>
```

Fetch HLStats and uncompress files to appropriate locations
```bash
wget http://www.hlstats-community.org/release/1.65/HLStats-1.65.tar.gz
tar xvfz HLStats-1.65.tar.gz
cd HLStats-1.65
mkdir /var/www/html/hlstats
cp -R web/* /var/www/html/hlstats/
```

Now we need to create the hlstats database
```bash
mysqladmin create hlstats --user=root -p
```

Now we need to grant the user access to the database to the hlstats user.  Whatever you put for the password will need to be used for all the hlstat configuration files to follow.
```bash
mysql --user=root -p
GRANT ALL ON hlstats.* TO 'hlstats'@'localhost' IDENTIFIED BY 'password';
exit
```

Now let's work on building the database
```bash
cd install
cp hlstats.sql.with.placeholder hlstats.sql
```

Edit the SQL script with `nano -w hlstats.sql` and search and replace `#DB_PREFIX#` with `hlstats` and save.  Then run the script to create the database with:
```bash
mysql --user=root -p hlstats < hlstats.sql
```

Let's move on to the configuration of the daemon
```bash
cd ../daemon
cp hlstats.conf.ini.sample hlstats.conf.ini
nano -w hlstats.conf.ini
```
Set the DBUsername, DBPassword and DBPrefix based upon the configuration done above and save.

Now we need to configure the web interface to talk to the database as well.
```bash
cd /var/www/html/hlstats/hlstatsinc
cp hlstats.conf.example.php hlstats.conf.php
nano -w hlstats.conf.php
```
Set the DB parameters in a similar fashion

Let's open up the port 80 on the firewall
```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```

`ifconfig` to see what your internal IP address is.  From your workstation running windows
http://<ip>/hlstats/index.php?mode=admin

username: admin
password: 123456

Let's install the PERL modules that we're going to need
```bash
yum install perl-CPAN
perl -MCPAN -e shell
```
Choose all the defaults.
Once screen prompt appears run all of these commands.
```bash
install Bundle::DBI
install Config::Tiny
install Digest::MD5
quit
```

Open the port for the HLStats daemon to receive logs
```bash
firewall-cmd --zone=public --add-port=27500/udp --permanent
firewall-cmd --reload
```

Let's run the daemon as a different user so we'll move those files
```bash
cd ~
mv HLStats-1.65 ~hlstats/
chown hlstats.hlstats ~hlstats/HLStats-1.65
```

Let's shift from root to the hlstats user and test some things out
```bash
su - hlstats
cd ~/HLStats-1.65/daemon/
perl ./hlstats.pl
```
Make sure it fires up successfully then CTRL-C it

Create hlstats.sh that looks like this:
```bash
#!/bin/bash
cd ~/HLStats-1.65/daemon/
while true; do nohup perl ./hlstats.pl; sleep 5; done &
```

Give the file permissions
```bash
chmod u+x ./hlstats.sh
```

Return to the root user by typing `exit`

Let's make this start up.  First make rc.local executable
```bash
chmod u+x /etc/rc.d/rc.local
```
Let's edit the startup script
```bash
nano -w /etc/rc.d/rc.local
```

Add this line to the bottom of it
```bash
su -s /bin/bash -c "/home/hlstats/HLStats-1.65/daemon/hlstats.sh" hlstats &
```

Save the file then reboot
Test that when the machine starts that it's listening on the web and daemon ports
```bash
netstat -a -n | egrep "80|27500"
```
