# Migrate Nextcloud From Ubuntu to FreeBSD
 
When I wrote these steps I was performing the migration but not every step may be 100% accurate.  I tried to be as clear as I could.  This write up was done a couple of months after the migration was completed.

The servers are on Digital Ocean.

When I first build the server it was an Owncloud server using the existing one click install from Digital Ocean.  Not longer after I converted the Owncloud server to a Nextcloud server following Nextcloud's instructions.  That is why you will see owncloud referenced below.

## Put Ubuntu Nextcloud in Maintenance Mode

```shell
sudo -u www-data php occ maintenance:mode --on
systemctl stop apache2
```

## Backup Ubuntu Nextcloud Data

**NOTE**:  Do not forget to save the database username and password Nextcloud used on the Ubuntu server.  If you use the same account on the FreeBSD Nextcloud you will have an easier transition.

```shell
cd /var/www
rsync -avx nextcloud/ /nextcloud-backup/
tar cvf nextcloud-data.tar /nextcloud-backup/
```

Transfer nextcloud-data.tar to the FreeBSD server.

## Backup Ubuntu Nextcloud Data

```shell
mysqldump -h [server] -u [username] -p[password] --single-transaction  owncloud > /nextcloud-sqlbkp.sql
```

Transfer the nextcloud-sqlbkp.sql to the FreeBSD server.

## Create ZFS Datasets

Attach a second disk to the FreeBSD droplet which will be used for Nextcloud and MySQL.  The mount points are the default locations Nextcloud and MySQL will install into.

```shell
zpool create data /dev/da0

zfs create data/nextcloud
zfs set compression=lz4 data/nextcloud
zfs set mountpoint=/usr/local/www/nextcloud data/nextcloud

zfs create data/mysql
zfs create data/mysql_secure
zfs create data/mysql_tmpdir
zfs set mountpoint=/var/db/mysql data/mysql
zfs set mountpoint=/var/db/mysql_secure data/mysql_secure
zfs set mountpoint=/var/db/mysql_tmpdir data/mysql_tmpdir

### these were not used during my migration
# these may not be good on small memory machine
# https://wiki.freebsd.org/ZFSTuningGuide#MySQL
zfs set primarycache=metadata data/mysql
zfs set atime=off data/mysql
zfs set recordsize=16k data/mysql/innodb
zfs set recordsize=128k data/mysql/logs
zfs set zfs:zfs_nocacheflush = 1
zfs set sync=disabled data/mysql 
```

## Install Nextcloud Software

```shell
pkg install nextcloud
pkg install mysql56-server
pkg install acme-client
pkg install apache24
pkg install mod_php56
pkg install php56-pecl-APCu
pkg install php56-opcache
pkg install php56-pcntl
```

## Configure sysrc

```shell
sysrc apache24_enable="YES"
sysrc mysql_enable="YES"
sysrc accf_http_load="YES"
sysrc accf_data_load="YES"
```

## Configure MySQL

Found that the MySQL configs used on FreeBSD did not have all the necessary settings.  I got the following error when trying to import data.

```shell
General error: 1665 Cannot execute statement: impossible to write to binary log since BINLOG_FORMAT = STATEMENT and at least one table uses a storage engine limited to row-based logging. InnoDB is limited to row-logging when transaction isolation level is READ COMMITTED or READ UNCOMMITTED.
```

I had to change the binlog_format in my.cnf

```shell
binlog_format = ROW
```

## Start MySQL / Configure Secure Install

```shell
service mysql-server start
mysql_secure_installation
```

## Configure Nextcloud Database

Because I had built an Owncloud server originally, that is the name of the database.  That is why you see owncloud here.

```shell
mysql -h [server] -u [username] -p[password] -e "DROP DATABASE nextcloud"
mysql -h [server] -u [username] -p[password] -e "CREATE DATABASE owncloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci"
```

## Create Nextcloud Database User

Because I had built an Owncloud server originally, that is the name of the database.  That is why you see owncloud here.  I am matching the same information that I had used on the Owncloud server.

```shell
CREATE USER 'owncloud'@'localhost' IDENTIFIED BY 'PASSWORD';
GRANT USAGE ON *.* TO 'owncloud'@'localhost';
GRANT ALL PRIVILEGES ON `owncloud`.* TO 'owncloud'@'localhost';
```

## Restore Data

Extract the data you transferred into /usr/local/www/nextcloud

```shell
tar -C /usr/local/www/nextcloud -xvf nextcloud-data.tar
```

Because of how the tar file was created on Ubuntu you may have to mv all the files so they end up in the /usr/local/www/nextcloud and not in a subdirectory in the /usr/local/www/nextcloud directory.  You should see something like the following.

```shell
$ cd /usr/local/www/nextcloud/
$ ls
3rdparty        console.php     index.html      ocs             resources       themes
apps            core            index.php       ocs-provider    robots.txt      updater
AUTHORS         cron.php        lib             public.php      settings        version.php
config          data            occ             remote.php      status.php
```

Set the proper permissions.

```shell
chown -R www:www /usr/local/www/nextcloud/
```

## Restore Database

```shell
mysql -h [server] -u [username] -p[password] [db_name] < nextcloud-sqlbkp.bak
```

## Turn Off Maintenance Mode

```shell
sudo -u www php occ maintenance:mode --off
```
