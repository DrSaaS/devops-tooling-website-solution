   
# Heading level 1 Devops Tooling Website 

Infrastructure: AWS
Webserver Linux: Red Hat Enterprise Linux 8
Database Server: Ubuntu 20.04 + MySQL
Storage Server: Red Hat Enterprise Linux 8 + NFS Server
Programming Language: PHP
Code Repository: GitHub

### I created one instance for the Web Server
### One instance for the NFS server
### One instance for the Database Server

PREPARING THE NFS SERVER
------------------------
Created 3 volumes and attached them to the NFS Server
I created a partition in each volume using the gdisk utility

sudo gdisk /dev/xvdf /dev/xvdg /dev/xvdh

# Then I installed the LVM package

sudo yum install lvm2 -y
# I created a physical volume on the partition

sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
# Volumes successfully created

# To scan for available disks and volumes
sudo lvmdiskscan

#Next, I created a volume group webdata-vg for the physical volumes
sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

# Created logical volumes:
sudo lvcreate -n lv-apps -L 4G webdata-vg
sudo lvcreate -n lv-logs -L 4G webdata-vg
sudo lvcreate -n lv-opt -L 4G webdata-vg

formated the file system to xfs
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt

# make mount points:
sudo mkdir /mnt/apps
sudo mkdir /mnt/logs
sudo mkdir /mnt/opt

# mount :
sudo mount /dev/webdata-vg/lv-apps  /mnt/apps
sudo mount /dev/webdata-vg/lv-log  /mnt/logs
sudo mount /dev/webdata-vg/lv-opt  /mnt/opt

#Install nfs server

sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service

# nfs server started successfully

# We set up permission that will allow our Web servers to read, write and execute files on NFS:

# change ownership to nobody

sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

# Grant read,write execute

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

# restart nfs server

sudo systemctl restart nfs-server.service
sudo systemctl status nfs-server.service

#active, running

Configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.32.0/20 ):

sudo vi /etc/exports
# Allow access from webserver1 (its CIDR)
/mnt/apps 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!
# export to make it visible when webservers try to connect

sudo exportfs -arv

# Opened the following ports in the NFS Security group


SSH   TCP  22  0.0.0.0/0


Custom TCP TCP   111  172.31.32.0/20


custom UDP UDP   111   172.31.32.0/20


Custom UDP UDP   2049    172.31.32.0/20


NFS TCP   2049    172.31.32.0/20




PREPARING THE DB SERVER
------------------------
sudo apt update
sudo apt install mysql-server
# Next I created a database (tooling) ad database user (webaccess)
sudo mysql
#Create user and grant access to webserver1 using webserver1 subnet CIDR 172.31.32.0/20 
create user 'webaccess'@'172.31.32.0/20' IDENTIFIED WITH mysql_native_password BY 'password'  
GRANT ALL PRIVILEGES ON tooling.* to 'webaccess'@'172.31.32.0/20' WITH GRANT OPTION;

flush privileges;


PREPARING THE WEB SERVERS
----------------------------
# we will utilize NFS and mount previously created Logical Volume lv-apps to the folder 
where Apache stores files to be served to the users (/var/www).

This approach will make our Web Servers stateless, which means we will be able to add new ones
 or remove them whenever we need, and the integrity of the data (in the database and on NFS) 
will be preserved.

#Install NFS client on Webserver 1

sudo yum install nfs-utils nfs4-acl-tools -y
# Mount /var/www/ and target the NFS server’s export for apps
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.38.79:/mnt/apps /var/www  - The NFS servers private IP address
/mnt/apps is located on nfs server and /var/www on our web server

Verify that NFS was mounted successfully by running df -h. Make sure that the changes will persist on Web Server after reboot:

Note: Files created in /var/www will be visible in mnt/apps

sudo vi /etc/fstab
# add following line

172.31.38.79:/mnt/apps /var/www nfs defaults 0 0


Install Remi’s repository, Apache and PHP on webserver1
----------------------------------------
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

sudo setsebool -P httpd_execmem 1

# Verify that Apache files and directories are available on the Web Server in /var/www 
and also on the NFS server in /mnt/apps. We had the same files – it means NFS 
is mounted correctly.
ls /mnt/apps and ls /var/www

#They contained same files so mounted properly

# mount and then update changes on vi /etc/fstab
sudo mount -t nfs -o rw,nosuid 172.31.38.79:/mnt/logs /var/log/httpd



# Fork the tooling source code from Darey.io Github Account to my Github account. 

sudo yum install git

git init

git clone https://github.com/darey-io/tooling.git

ls   gives tooling directory
ls tooling     give html file

#copy the html files (contents of html only) of the cloned repository to /var/www/html 

sudo cp -R html/. /var/www/html (copy when cd into tooling folder)

# file succesfully copied

#next we open port 80 on the webserver and allow connection from 0.0.0.0/0

# Try running webserver public IP in browser. If failed, troubleshoot

1. check security groups
2. check status of apache
sudo setenforce 0 ( Do this from root/home)
To make this change permanent – open following config file sudo vi /etc/sysconfig/selinux 
and set SELINUX=disabledthen restrt httpd.

#Apache page now displayed after previous 403 error.
#Success
# Install mysql
sudo yum install mysql -y

Update the website’s configuration to connect to the database (in /var/www/html/functions.php file). 
Apply tooling-db.sql script to your database using this command 

sudo mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql



Create in MySQL a new admin user with username: myuser and password: password:

INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES
-> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);

Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and make sure you can login into the websute with myuser user.

sudo mysql -h 172.31.47.167 -u webaccess -p tooling < tooling-db.sql


INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES
(1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);
































