# Reviewer Information

## Description

Take a baseline installation of a Linux distribution on a virtual machine and prepare it to host a web application, install updates, secure it from a number of attack vectors, and install/configure web and database servers.

## IP Address and SSH Port

Static IP Address: 52.37.6.18

SSH Port: 2200

URL of hosted application: http://ec2-52-37-6-18.us-west-2.compute.amazonaws.com

## Configuration Steps

### Create Virtual Machine

- Create an Ubuntu 16.04 LTS instance on Amazon Lightsail
- Download public/private keypairs from Amazon
- Attach a static IP

### Connect to VM via SSH
- Establish an SSH connection to the VM using the key from above, ie: ```ssh -i key.pem -p 2200 ubuntu@52.37.6.18```

### Update Packages
- Update and upgrade packages:

```
sudo apt-get update
sudo apt-get upgrade
```

### Set Time Zone to UTC
- Run ```sudo dpkg-reconfigure tzdata```
- Choose None and UTC from menu

### Set Up User _grader_
- Add grader user: ```sudo adduser grader```
- Create grader file in sudoers: ```sudo nano /etc/sudoers.d/grader```
- Add this line to this grader file and save the file: grader ```ALL=(ALL:ALL) ALL```

### Set Up Key-Based Authentication for grader
- Generate keys on local machine using ```ssh-keygen```
- Log into VM as root via ssh and:

```
sudo mkdir /home/grader/.ssh
sudo touch /home/grader/.ssh/authorized_keys
```

- Paste contents of local public key into the newly-created authorized-keys file and save
- Change permissions of the folder and file:

```
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
```

- Change the owner to grader: ```sudo chown -R grader:grader /home/grader/.ssh```

### Change SSH Port to 2200
- Change SSH port: ```sudo nano /etc/ssh/sshd_config```: change Port to 2200
- Restart SSH: ```sudo service ssh restart```
- Log out
- Go to Lightsail console and set:
- In Lightsail console: Allow TCP on port 2200 and delete default SSH port 22

### Switch to grader User Going Forward
- Log in as grader: ```ssh -i key.pem -p 2200 grader@52.37.6.18```

### Enforce Key-Based SSH Authentication
- ```sudo nano /etc/ssh/sshd_config```: change PasswordAuthentication to No and save

### Disable root Login
- Disable root: ```sudo nano /etc/ssh/sshd_config```: set PermitRootLogin to no
- Restart SSH: ```sudo service ssh restart```

### Configure UFW
- Only allow connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):

```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

### Install and Configure Apache to Serve a Python mod_wsgi Application
- Install Apache: ```sudo apt-get install apache2```
- Install mod_wsgi and Python: ```sudo apt-get install libapache2-mod-wsgi python-dev```
- Start the server: ```sudo service apache2 start```

### Git Clone the Application from Project 4
- Install Git: ```sudo apt-get install git```
- Make a catalog directory in the www directory: ```sudo mkdir /var/www/catalog```
- Change owner to grader user: ```sudo chown -R grader:grader /var/www/catalog```
- Go to the new directory: ```cd /var/www/catalog```
- Clone the repository: ```git clone https://github.com/cloverhub/ACME-Catalog.git catalog```

### Make Several Modifications to Code to Allow Apache Webhosting
- Rename catalog.py to __init__.py because an init file is required: ```sudo mv catalog.py __init__.py```
- Add full path to the client_secrets.json file in both references in __init__.py: ```/var/www/catalog/catalog/client_secrets.json```

### Set Up WSGI
- Make a wsgi file: ```sudo nano catalog.wsgi```
- Paste the following into the new file and save:

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey
```

### Set Up Dependencies
- Install pip: ```sudo apt-get install python-pip```
- Install virtualenv: ```sudo pip install virtualenv```
- Go to catalog directory: ```cd /var/www/catalog/catalog```
- Create virtual environment: ```sudo virtualenv venv```
- Activate virtual environment: ```source venv/bin/activate```
- Change permissions: ```sudo chmod -R 777 venv```
- Install Flask: ```pip install Flask```
- Install the remainder of the dependencies: ```pip install sqlalchemy httplib2 requests oauth2client psycopg2_binary```

### Create Virtual Host
- Create catalog.conf file: ```sudo nano /etc/apache2/sites-available/catalog.conf```
- Paste the following into this file and save:

```
<VirtualHost *:80>
    ServerName 52.37.6.18
    ServerAlias ec2-52-37-6-18.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.37.6.18
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Enable the virtual host: ```sudo a2ensite catalog```

### Set Up PostgreSQL and Change SQLite to PostreSQL
- Install PostgreSQL and PostgreSQL library: ```sudo apt-get install postgresql postgresql-contrib libpq-dev```
- Change default postgres user to  catalog user:

```
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'catalogpassword';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
/q
exit
```

- Change the SQLite connection string lines in all three .py files to ```engine = create_engine('postgresql://catalog:catalogpassword@localhost/catalog')```
- Set up the database: ```sudo python database_setup.py```
- Populate the database: ```sudo python database_seed.py```
- Ensure no remote database connections are allowed: ```sudo nano /etc/postgresql/9.5/main/pg_hba.conf``` should end with:

```
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             postgres                                peer
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```

### Reconfigure Google OAuth
- Install oauth2client: ```sudo pip install --upgrade oauth2client```
- Generate a new client ID in the Google API interface and overwrite the client id path in both __init__.py and client_secrets.json (this step may or may not be necessary--I changed the client id as part of troubleshooting an issue with gconnect not working)
- In the Google API interface, change Authorized JavaScript origins to ```http://ec2-52-37-6-18.us-west-2.compute.amazonaws.com```
- In the Google API interface, change Authorized redirect URIs to: ```http://ec2-52-37-6-18.us-west-2.compute.amazonaws.com/gconnect``` and ```http://ec2-52-37-6-18.us-west-2.compute.amazonaws.com/login```

### Restart the Apache Server
```sudo service ssh restart```

## Third-Party Resources Used
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04
- https://www.sqlalchemy.org/features.html
- https://www.daveoncode.com/2017/03/07/how-to-solve-python-modulenotfound-no-module-named-import-error/
- https://unix.stackexchange.com/questions/109245/hostname-and-ip-address-mapping-in-etc-hosts
- https://www.jakowicz.com/flask-apache-wsgi/
- https://www.digitalocean.com/community/questions/flask-deployment-issue-with-imports

