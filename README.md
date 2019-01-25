# Linux Server Configuration

This Project sets up a Linux Server for the deployment of ToDo Web Application. It includes installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

## QuickStart

Obtain `id_rsa` from "Notes to Reviewer" field.
<ul>
<li> Public IP Address: 13.233.250.67
<li> Username: grader
<li> Port: 2200
<li> URL: http://ec2-13-233-250-67.ap-south-1.compute.amazonaws.com/
<li> To Login into the server use the following command: <code>ssh grader@13.233.250.67 -p 2200 -i /path_to_file/id_rsa
</code>
</ul>

## Creating your LightSail Instance for ubuntu

1. Login into [AWS Light Sail](https://lightsail.aws.amazon.com) using your provided email and password to amazon

2. Once you are login into the site, there is a button called create Instance. Click that button.
![Create Instance](/images/create_instance.png)

3. You'll be given two options: App+OS, OS only.
![Create Instance Image](/images/create_instance_image.JPG)

4. Choose OS only and ubuntu

![OS Only](/images/os_only.png)

5. Choose the instance plan and then click the button

![Choose Instance Plan](/images/choose_instance_plan.png)

You can name your instance or just leave the default name provided by AWS .  They give an option to change the instance name.

Once we have created our ubuntu image, now we need to start configuring the server to deploy our flask application.

## Accessing the instance

Before setting up the server, we need to download the lightsail pem file from the LightSail website.  In order to get the file, you need to go the accounts page.
Once you are in the accounts page, there is an option to download the default private key.

![Light Sail Private Key](/images/lightsail_private_key.png)

To see if you can login into instance. Two things need to be done:

1. You need to change permission of the downloaded lightsail private key.
`chmod 400 LightsailDefaultPrivateKey.pem` or `chmod 600 LightsailDefaultPrivateKey.pem`

2. To login:
`ssh user@PublicIP -i LightsailDefaultPrivateKey`

**Please make sure your private key is the same directory. The default port for SSH is 22 but we'll change it later on in the documentation**

To check your PublicIP, LightSail has manage option on the instance. Click the icon that has 3 dots

![Three Dot Icon](/images/dot.png)

Click the manage option.  You'll go to a screen looking this:

Management Console

![Management Console](/images/console.png)

Once you are able to access the instance, lets update our system and then create our new user.

## Updating the server
Since you are running ubuntu, its only two commands:

```
sudo apt-get update
sudo apt-get upgrade
```

## Creating the New user with sudo permissions
You are login into the instance. Lets create a new user called grader:
```
sudo adduser grader
```
It will prompt for a password. Add any password to that field.
After the user gets created, run this command to give the grader sudo priviledges:
```
sudo visudo
```
Once your in the visudo file, go to the section user privilege specification
```
root    ALL=(ALL:ALL) ALL
```
You've noticed the root user in the file. You need add the grader user to same section.  It should look like this:
```
grader  ALL=(ALL:ALL) ALL
```
If you want to login into the account, check to see if works: `sudo login grader`.
We've created our new user, lets configure our ssh to non-default port

## Configuring SSH to a non-default port

SSH default port is 22. We want to configure it to non default port.  Since we are using LightSail, we need to open the non default port through the web interface. If you don't do this step, you'll be locked out of the instance. Go to the networking tab on the management console. Then, the firewall option

![LightSail firewall](/images/lightsailfirewall.png)

Go into your sshd config file: `sudo nano /etc/ssh/sshd_config`
Change the following options. # means commented out

```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200

# Authentication:
LoginGraceTime 120
#PermitRootLogin prohibit-password
PermitRootLogin no
StrictModes yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication No
```
Once these changes are made, restart ssh: `sudo service restart ssh`
Any user can login into the machine using a specify port: `sudo user@PublicIP -p 2200`
### Creating the ssh keys
```
LOCAL MACHINE:~$ ssh-keygen
```
Two things will happen:
1. You can name the file whatever you want.  Keep in mind when you name the file it will give you a private key and public key
2. It will prompt you for a pass phrase. You can enter one or leave it blank

Once the keys are generated, you'll need to login to the user account aka grader here:
You'll make a directory for ssh and store the public key in an authorized_keys files

```
mkdir .ssh
cd .ssh
```
When you are in the directory, create an authorized_keys file. This where you paste the public key that you generated on your local machine.
```
sudo nano authorized_keys
```
Please double check your path by pwd. You should be in your `/home/grader/.ssh/` when creating the file. You should be login with grader account using the private key from local machine into the server: `ssh user@PublicIP -i ~/.ssh/whateverfile_id_rsa`

## Configuring Firewall rules using UFW
We need to configure firewall rules using UFW. Check to see if ufw is active: `sudo ufw status`. If not active, lets add some rules
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp or sudo ufw allow www (either one of these commands will work)
sudo ufw allow 123/udp
```
Now, you enable the rules: `sudo ufw enable` and re check the status to see what rules are activity

## Configure timezone for server
```
sudo dpkg-reconfigure tzdata
```
Choose none of the above and choose UTC.  The server by default is on UTC.

## Install Apache, Git, and flask

### Apache
We will be installing apache on our server. To do that:

```
sudo apt-get install apache2
```
If apache was setup correctly, a welcome page will come up when you use the PublicIP. We are going to install mod wsgi, python setup tools, and python-dev
```
sudo apt-get install libapache2-mod-wsgi python-dev
```
We need to enable mod wsgi if it isn't enabled: `sudo a2enmod wsgi`

Let's setup wsgi file and sites-available conf file for our application.
Create the WSGI file in `path/to/the/application directory`
```
WSGI file

import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/catalog')

from application import app as application
application.secret_key='super_secret_key'
```
Please note application.py is my python file. Wherever you housed your application logic.

To setup a virtual host file: `cd /etc/apache2/sites-available/itemsCatalog.conf`:
```
Virtual Host file
<VirtualHost *:80>
     ServerName  PublicIP
     ServerAdmin email address
     #Location of the items-catalog WSGI file
     WSGIScriptAlias / /var/www/itemsCatalog.wsgi
     #Allow Apache to serve the WSGI app from our catalog directory
     <Directory /var/www/catalog>
          Order allow,deny
          Allow from all
     </Directory>
     #Allow Apache to deploy static content
     <Directory /var/www/catalog/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```
### Git
`sudo apt-get install git`

Clone repository into the apache directory. In the `cd /var/www`

```
mkdir itemsCatalog
cd /var/www/catalog

sudo git clone https://github.com/avi1245/FSND_item_catalog.git catalog
```

### Flask
Do these commands:

```
sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
sudo pip install oauth2client requests httplib2
```

## Install PostGreSql
Install PostGreSql `sudo apt-get install postgresql`
To make sure remote are not allowed, please check the following configuration file: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

### Create database
`sudo su - postgres`
Type in `psql` as postgres user

Do following the commands:
```
postgres=# CREATE USER catalog WITH PASSWORD 'catalog';
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
```
Connect to the catalog database: `\c catalog`
```
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
```
Exit postgres and postgres user
```
postgres=# \q
postgres@PublicIP~$ exit
```

We need to update our database setup and application python files to illustrate the new engine connection
`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

Run `sudo python database_setup.py`

## Oauth Client Login

To setup it up. Update the path in the application.py program. Update the client id and oauth_flow

```
CLIENT_ID = json.loads(
    open('/var/www/catalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/catalog/client_secrets.json', scope='')
```
