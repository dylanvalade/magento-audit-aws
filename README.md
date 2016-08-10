# Deploy Magento

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

  - [Log into AWS Console (payment method required)](#log-into-aws-console-payment-method-required)
    - [Amazon Web Services Terminology](#amazon-web-services-terminology)
    - [Tips](#tips)
  - [Launch an Elastic Compute Cloud (EC2) instance](#launch-an-elastic-compute-cloud-ec2-instance)
    - [SSH to EC2](#ssh-to-ec2)
  - [Amazon CodeDeploy](#amazon-codedeploy)
  - [Upload Files to EC2 with `scp` (secure copy)](#upload-files-to-ec2-with-scp-secure-copy)
  - [Database - Create an RDS (relational database)](#database---create-an-rds-relational-database)
    - [Notes](#notes)
    - [Launch a new database server](#launch-a-new-database-server)
    - [Enable access to the RDS database server](#enable-access-to-the-rds-database-server)
    - [Connecting to your DB Instance](#connecting-to-your-db-instance)
    - [Steps to import Magento database](#steps-to-import-magento-database)
    - [MySQLWorkbench - GUI Database Administration](#mysqlworkbench---gui-database-administration)
    - [Change the Magento Domain Name](#change-the-magento-domain-name)
    - [Disable caching](#disable-caching)
  - [Prepare Web Server](#prepare-web-server)
    - [Enable HTTP](#enable-http)
    - [Install LAMP Stack](#install-lamp-stack)
    - [Confirm Apache is running](#confirm-apache-is-running)
    - [View Apache test page](#view-apache-test-page)
    - [Set File Permissions](#set-file-permissions)
    - [Test PHP](#test-php)
    - [Setup MySQL (if not using RDS)](#setup-mysql-if-not-using-rds)
  - [Enable SSL](#enable-ssl)
  - [Install PHPMyAdmin on EC2 if you're not using RDS](#install-phpmyadmin-on-ec2-if-youre-not-using-rds)
- [Magento](#magento)
  - [Update Magento permissions](#update-magento-permissions)
  - [Edit Magento Database Configuration](#edit-magento-database-configuration)
  - [Turn on Debugging](#turn-on-debugging)
  - [Magento Admin Panel](#magento-admin-panel)
    - [Adding a user programmatically](#adding-a-user-programmatically)
    - [Finding the Admin Login Path](#finding-the-admin-login-path)
  - [Rebuild Indexes](#rebuild-indexes)
    - [Problem: You can view the Homepage but no other pages](#problem-you-can-view-the-homepage-but-no-other-pages)
      - [Relax the permissions on your root Magento folder](#relax-the-permissions-on-your-root-magento-folder)
    - [Password protect store with .htaccess](#password-protect-store-with-htaccess)
    - [Magento page templates](#magento-page-templates)
    - [Find out which .phtml template is being used](#find-out-which-phtml-template-is-being-used)
    - [Edit page/template/layout structure](#edit-pagetemplatelayout-structure)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Log into AWS Console (payment method required)

### Amazon Web Services Terminology

- *Amazon Machine Images (AMI)*: disk image, AMI you specified at launch is used to boot the instance
- *EC2*: virtual server
- *EBS Volume*: disk storage, hard drive
- *RDS*: relational database server
- *Snapshot*: backup of the disk storage (volume)
- *Public DNS*: subdomain on amazonaws.com (ec2-52-207-215-180.compute-1.amazonaws.com) which includes self-signed SSL cert
- *Reboot*: An instance reboot is equivalent to an operating system reboot. In most cases, it takes only a few minutes to reboot your instance. When you reboot an instance, it remains on the same physical host, so your instance keeps its public DNS name, private IP address, and any data on its instance store volumes. Rebooting an instance doesn't start a new instance billing hour, unlike stopping and restarting your instance.
- *Terminate*: When you've decided that you no longer need an instance, you can terminate it. As soon as the status of an instance changes to shutting-down or terminated, you stop incurring charges for that instance.
- *

### Tips
- Stopping an EC2 instance is not like restarting a virtual or dedicated server
- Every time you restart EC2 a new public DNS subdomain is provided which can be an issue for a test URL, use a custom domain name to avoid changing URLs, however the SSH keys remain the same
- Shell history is available after restarting EC2
- When you stop an EC2 instance, if you don't have EBS storage then do not expect the files to be available again after restarting. Two types of storage: [ephemeral-one](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/InstanceStorage.html) (or "instance-stored"), those drives are not persistent and will be re-created after instance stop/start. [EBS volumes](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/AmazonEBS.html) are separate from EC2 instances, keeping data after instance stop/start.
- Make a snapshot before stopping EC2
- There are multiple ways to increase the disk size but it isn't as easy "Click to add 5GB" in the console. [Here is Amazon's documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-expand-volume.html#migrate-data-larger-volume)

## Launch an Elastic Compute Cloud (EC2) instance

- Amazon Linux (free tier)
- Needed to [expand the disk size](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-expand-volume.html#migrate-data-larger-volume) which is known as an Elastic Block Store (EBS) Volume
- [EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html?icmpid=docs_ec2_console)
- It takes a few minutes to deploy

### SSH to EC2

- You will be able to create an SSH key by clicking the CONNECT button after the instance is created
- `ssh -i ~/path/to/downloaded/ssh/key/file/aws-au.pem ec2-user@ec2-54-174-122-3.compute-1.amazonaws.com`

*SSH Error `Permission denied (publickey).`*
- Need to whitelist your IP address or make SSH available from all IP's
- Documenation: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html
- [Add the rule for a group to allow inbound IP's to connect via SSH](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups:sort=groupId)

Successful connection:

```sh
Dylan$ ssh -i /Users/Dylan/.ssh/aws-au.pem ec2-user@ec2-54-174-122-3.compute-1.amazonaws.com

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2016.03-release-notes/
8 package(s) needed for security, out of 17 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-59-74 ~]$
```

- See how much disk space is available `df -h`

## Amazon CodeDeploy

- [with Github](http://docs.aws.amazon.com/codedeploy/latest/userguide/github-integ-tutorial.html)

## Upload Files to EC2 with `scp` (secure copy)

- Use a shell that is not logged into the server
- `scp -i /path/my-key-pair.pem /path/SampleFile.txt ec2-user@ec2-198-51-100-1.compute-1.amazonaws.com:~`
- Upload compressed (tar) `http` directory to home (~)
- Upload compressed database to home (~)
- Uncompress files `tar –xvzf ~/filename.tar`
- Overwrite existing http contents `mv ~/http /var/www`

## Database - Create an RDS (relational database)

### Notes
- Your EC2 instance can be either VPC or Classic (legacy). If it's VCP then your EC2 security groups for VCP control the access to the separate RDS database services too.

### Launch a new database server

- Log into AWS Console
- To see existing databases click RDS > Instances
- To add a new database: RDS > Launch DB Instance
- Select MariaDB instead of MySQL for the engine
- Choose Dev/Test (free tier)
- Use default settings, most basic service options
- Enable monitoring
- Launch DB
- It takes several minutes to create the new DB server
- Once the DB is up, you can get the endpoint by clicking on the black right arrow in the second column of the instance row (aws1au.ch9n1thno1ii.us-east-1.rds.amazonaws.com:3306)

### Enable access to the RDS database server

- AWS Console > EC2 > Security Groups
- Choose the group named something like `rds-launch-wizard-1` with a description about the RDS Management Console
- Edit the Inbound tab should have one rule `Type: MYSQL/Aurora` `Protocol: TCP` `Port: 3306` and your IP address when you set up RDS
- If you need or want to, change the Source to `Anywhere` so you can access it from multiple or dyanmic IP's.


### Connecting to your DB Instance

- You need to have `mysql` installed on the machine that is connecting to the RDS. If it's not installed on the computer you're using, you can connect to it from the EC2 shell assuming the EC2 instance is allows to connect with the security group rule. Acheive this by alling Anyone (e.g. any IP) to connect in the inbound rule.
- You can only connect to your database instance if EC2 has the necessary security group rule giving your IP access
- If you are using RDS MySQL provide the database endpoint in the `host` field with the username and password for the RDS database server instance
- Sample connection which will prompt for the database password `mysql -h aws-db1.ch9n1thno1ii.us-east-1.rds.amazonaws.com -P 3306 -u databaseUsername -p -v`

Successful connection to Maria DB on RDS:

```sh
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 304
Server version: 5.5.5-10.1.14-MariaDB MariaDB Server

Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Reading history-file /home/ec2-user/.mysql_history
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

### Steps to import Magento database

[Amazon AWS Import MySQL/MariaDB Documentation](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.NonRDSRepl.html)

You will be piping the existing database .sql export/dump file into a mysql command. The file must exist on a computer that can run `mysql` commands. If your computer can't connect, then upload the .sql file to EC2 and connect to RDS from the EC2 shell.

The following commands are run from the `mysql>` prompt

- Check to see if the database you want to use for Magento is already created `show databases;`
- If the database you want is there run `use databaseName;` else add the database `create database databaseName;` `use databaseName;`
- Assuming the Magento backup.sql is your `source backup.sql;`
- You'll watch for several minutes as the SQL commands are displayed and each query result is printed
- If it was successful you should see in the output that rows were changed

### MySQLWorkbench - GUI Database Administration

- [Download the MySQLWorkbench app](https://www.mysql.com/products/workbench/)
- Open the program and add a connection
- Type: TCP
- Host: RDS endpoint (aws-db1.ch9n1thno1ii.us-east-1.rds.amazonaws.com)
- Port: 3306
- User: RDS databaseUsername
- Password: RDS databasePassword
- Click `Test Connection` and if it's good you have the right security group rules in AWS EC2 and the correct credentials
- Expand your databaseName from the left sidebar and you can see and edit the database, tables, users and row values

### Change the Magento Domain Name

- Default URL paths for Magento

- [Change Base URLs](http://magento.stackexchange.com/questions/39752/how-do-i-fix-my-base-urls-so-i-can-access-my-magento-site)
- Update table `core_config_data` `web/secure/base_url` to `https://yourEC2PublicDomain/` and `web/unsecure/base_url` to `http://yourEC2PublicDomain/` with the trailing slashes
- Update `web/cookie/cookie_domain` to your EC2 public domain without http:// `ec2-52-207-215-180.compute-1.amazonaws.com`, which is required to avoid CSRF attack prevention on the form submission. You will not be able to log into the admin panel. Instead you'll see the error "invalid form key. please refresh the page."
- Clear Magento cache `sudo rm -r /var/www/html/var/cache/*`
- Clear Magento sessions `sudo rm -r /var/www/html/var/sessions/*`
- Clear the secret Magento cache if the Magento cache directory is not writable `sudo rm -r /tmp/magento/var/cache/*`
- If you have data in the secret cache then you need to change the permissions.
- Restart apache `sudo service httpd reload`

### Disable caching

Set every value in the `core_cache_option` table from `1` to `0`

```sql
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='block_html';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='collections';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='config';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='config_api';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='config_api2';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='eav';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='layout';
UPDATE `dbname`.`core_cache_option` SET `value`='0' WHERE `code`='translate';
```

Remove extra config files

- `sudo rm /var/www/http/var/etc/local.xml.backup`
- `sudo rm /var/www/http/var/etc/local.xml.template`
- `sudo rm /var/www/http/var/etc/local.xml.additional`


## Prepare Web Server

### Enable HTTP
Configure your security group for EC2 to allow connections:

- SSH (port 22)
- HTTP (port 80), and HTTPS (port 443)

### Install LAMP Stack

- [Documentation for Amazon Linux EC2 LAMP install](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-LAMP.html)
- SSH to server
- Update packages `sudo yum update -y`
- Install Apache, MySQL and PHP `sudo yum install -y httpd24 php56 mysql55-server php56-mysqlnd`
- Start Apache `sudo service httpd start`
- Configure the Apache web server to start at each system boot `sudo chkconfig httpd on`

### Confirm Apache is running

`chkconfig --list httpd`

Correct output:

`httpd           0:off   1:off   2:on    3:on    4:on    5:on    6:off`

### View Apache test page

- AWS > EC2 Dashboard > Instances > Click instance name > Public DNS (use CMD+F or CTRL+F to find "Public DNS")
- Public DNS may be hidden, click Show/Hide
- Example: http://ec2-54-174-122-3.compute-1.amazonaws.com/
- Amazon Linux Apache document root for the `httpd` service is `/var/www/html`

### Set File Permissions

- Add the www group to your instance. `sudo groupadd www`
- Add your user (ec2-user) to the `www` group. `sudo usermod -a -G www ec2-user`
- Log back in to pick up the new group. `exit` `ssh ...`
- Verify your membership in www group `groups`
- correct ouput: `ec2-user wheel www`
- Change the group ownership of /var/www and its contents to the www group.
- `sudo chown -R root:www /var/www`
- Change the directory permissions of /var/www and its subdirectories to add group write permissions and to set the group ID on future subdirectories.
- `sudo chmod 2775 /var/www`
- `find /var/www -type d -exec sudo chmod 2775 {} \;`
- Recursively change the file permissions of /var/www and its subdirectories to add group write permissions.
- `find /var/www -type f -exec sudo chmod 0664 {} \;`

Now ec2-user can add, delete, and edit files in the Apache document root. You are ready to add content.

### Test PHP

- `echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php`
- Check your test URL (e.g. http://ec2-54-174-122-3.compute-1.amazonaws.com/phpinfo.php)
- `rm /var/www/html/phpinfo.php`

### Setup MySQL (if not using RDS)

- Start MySQL server `sudo service mysqld start`
- Run setup `sudo mysql_secure_installation`
- No password exists yet so click `enter` at the prompt
- Type `Y` to create a root password
- Type `Y` to remove the anonymous user accounts.
- Type `Y` to disable remote root login.
- Type `Y` to remove the test database.
- Type `Y` to reload the privilege tables and save your changes.
- Run MySQL server every time instance boots `sudo chkconfig mysqld on`

## Enable SSL

- Install mod_ssl `sudo yum install -y mod24_ssl`
- Restart Apache `sudo service httpd restart`
- SSL on your AWS Public DNS domain will work but throw warnings for a self-signed cert. (e.g. https://ec2-54-174-122-3.compute-1.amazonaws.com/)
- Test SSL effectiveness with [SSL Test](https://www.ssllabs.com/ssltest/analyze.html) A is great, C is encrypted but could have better user experience and security

## Install PHPMyAdmin on EC2 if you're not using RDS
Do not use PHPMyAdmin without SSL because the database login will be comprimised

- Enable the Extra Packages for Enterprise Linux (EPEL) repository from the Fedora project on your instance.
- `sudo yum-config-manager --enable epel`
- Install PHPMyAdmin `sudo yum install -y phpMyAdmin`
- Get your IP Address (e.g. 66.227.133.167)
- `sudo sed -i -e 's/127.0.0.1/66.227.133.167/g' /etc/httpd/conf.d/phpMyAdmin.conf`
- Restart Apache `sudo service httpd restart`
- Restart MySQL `sudo service httpd restart`
- Login with root user and password setup during mysql install at your AWS domain/phpmyadmin (e.g. https://ec2-54-174-122-3.compute-1.amazonaws.com/phpmyadmin)


# Magento

## Update Magento permissions

- [StackExchange - Magento folder and file permissions](http://magento.stackexchange.com/questions/68627/file-permissions-changed-after-applying-patch-403-forbidden-error)
- [403 Forbidden Error](http://magento.stackexchange.com/questions/68627/file-permissions-changed-after-applying-patch-403-forbidden-error)

[Recommended Privileges Before Install](http://devdocs.magento.com/guides/m1x/install/installer-privileges_before.html)

- `cd /var/www/html/`
- `find . -type d -exec chmod 700 {} \;`
- `find . -type f -exec chmod 600 {} \;`

[Recommended Privileges After Install](http://devdocs.magento.com/guides/m1x/install/installer-privileges_after.html)

- `cd /var/www/html/`
- `find . -type f -exec chmod 400 {} \;`
- `find . -type d -exec chmod 500 {} \;`
- `find var/ -type f -exec chmod 600 {} \;`
- `find media/ -type f -exec chmod 600 {} \;`
- `find var/ -type d -exec chmod 700 {} \;`
- `find media/ -type d -exec chmod 700 {} \;`

[Reset File Permissions](https://support.terranetwork.net/web/knowledgebase/119/Resetting-File-Permissions-for-Magento.html)

- `cd /var/www/html/`
- `find . -type f -exec chmod 644 {} \;`
- `find . -type d -exec chmod 755 {} \;`
- `chmod 550 pear`
- `chmod 550 mage`

## Edit Magento Database Configuration

- Look at `/app/etc/config.xml` or `/app/etc/local.xml` to update the `connection`
- Change the `host`, `username`, `password` and `dbname`
- If you are using RDS, the RDS Endpoint is database host (aws-db1.ch9n1thno1ii.us-east-1.rds.amazonaws.com)

```xml
<connection>
    <host><![CDATA[yourDatabseHostname]]></host>
    <username><![CDATA[yourDatabaseUsername]]></username>
    <password><![CDATA[yourDatabasePassword]]></password>
    <dbname><![CDATA[yourDatabaseName]]></dbname>
    <initStatements><![CDATA[SET NAMES utf8]]></initStatements>
    <model><![CDATA[mysql4]]></model>
    <type><![CDATA[pdo_mysql]]></type>
    <pdoType><![CDATA[]]></pdoType>
    <active>1</active>
</connection>
```

## Turn on Debugging

- Turn on PHP errors, in `index.php` uncomment or add `ini_set('display_errors', 1);`

## Magento Admin Panel

### Adding a user programmatically

- Create a file called `eureka.php` with the following code and put the file the root Magento folder (/var/www/html/eureka.php).
- Add the user `php eureka.php`
- Remove the file `rm eureka.php`
- [Source](https://santoshyadavcse.wordpress.com/2013/02/04/magento-create-admin-user-programmatically/)

```php
<?php
// Create New admin User programmatically.
require_once('./app/Mage.php');
umask(0);
Mage::app();

try {
    $user = Mage::getModel('admin/user')
        ->setData(array(
            'username'  => 'admin1',
            'firstname' => 'Admin',
            'lastname'    => 'Admin',
            'email'     => 'dylan@sngm.me',
            'password'  =>'',
            'is_active' => 1
        ))->save();

} catch (Exception $e) {
    echo $e->getMessage();
    exit;
}

// Assign Role Id
// Administrator role id is 1
try {
    $user->setRoleIds(array(1))
    ->setRoleUserId($user->getUserId())
    ->saveRelations();

} catch (Exception $e) {
    echo $e->getMessage();
    exit;
}

echo "User created successfully";

?>
```

### Finding the Admin Login Path

- Check `/app/etc/local.xml`
- Look for `frontname` and in this case the path to the admin panel would be `/admin/` from this XML `<frontName><![CDATA[admin]]></frontName>`
- Browse to https://baseurl.com/index.php/admin and you will view the Magento login

## Rebuild Indexes

If you can log into admin panel go to System > Index Management

The catalog reindexing can take a long time and will time out in PHP. Use this command from Magento root:

`php -d max_execution_time=0 ./shell/indexer.php --reindex catalog_url` or pass `reindexall` instead

Successful reindex: `Catalog URL Rewrites index was rebuilt successfully in 00:16:23`

### Problem: You can view the Homepage but no other pages

It is likely that the rewrite rules are not working, which is related to mod_rewrite and .htaccess

- Confirm with phpinfo() that PHP has mod_rewrite installed and if it's not there install it
- .htaccess rules cascade down so you can customize them per directory or vhost. The top level .htaccess can have strict persmissions set that do not allow the sub .htaccess files to override the settings.

#### Relax the permissions on your root Magento folder

- Edit `/etc/httpd/conf/httpd.conf`
- Find the block starting with `<Directory "/var/www/html">`
- Edit it so that the remaining directives are as follows (note, the comments don't matter):

```sh
<Directory "/var/www/html">
    Options Indexes FollowSymLinks Includes
    AllowOverride All
    Allow from All
</Directory>
```

- Restart Apache `sudo service httpd reload`
- Try viewing website again, it may be working now.

### Password protect store with .htaccess

- Generate the contents of a login file called `.htpasswd` with this http://www.htaccesstools.com/htpasswd-generator/
- Example output: `admin@test.com:$apr1$HRwJVYdU$o74.vQbjemjIYpGcbsCM5.`
- Paste into `.htpasswd` into Magento root
- Add the following to the beginning of `/var/www/html/.htaccess`

```sh
AuthName "Login"
AuthUserFile "/var/www/html/.htpasswd"
AuthType Basic
require valid-user
ErrorDocument 401 "Login Required"
```

### Magento page templates

- Markup, .phtml template files, layout files `app/design/frontend/default/THEME_NAME/template/`
- Assets (CSS, JS, images) `skin/frontend/default/THEME_NAME/`

### Find out which .phtml template is being used

- Add this to beginning of .phtml files and then view your page source to see if they are being used `echo '<!-- ' . $this->getLayout()->getBlock('root')->getTemplate() . ' -->';`
- Example output if the .phtml you added it to is in use: `<!-- page/homepage.phtml -->`

### Edit page/template/layout structure

- Layouts `/var/www/html/app/design/frontend/default/THEME_NAME/layout/local.xml`
- .phtml templates `/var/www/html/app/design/frontend/default/THEME_NAME/template`
- Page templates (homepage.phtml) `/var/www/html/app/design/frontend/default/THEME_NAME/template/page`
