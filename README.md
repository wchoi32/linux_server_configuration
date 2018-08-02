# Linux Server Configuration Projct
Deployed and host my web application called Item Catalog. 
Involves hosting using AWS Lightsail, installing Ubuntu and updates, securing from number of attack vectors, and configuring web and database servers.

## IP Address and SSH Port

IP Address:
```
54.71.80.71
```

SSH Port:
```
2200
```

## Complete URL to hosted web application

```
http://ec2-54-71-80-71.us-west-2.compute.amazonaws.com/
```

## Summary of software installed and configuration changes made

1. Login to Amazon Lightsail from Amazon Web Services Account
2. Create Instance and install OS Only with Ubuntu 16.04LTS
3. Goto Account and download default private key 
4. locate the key and from terminal type: chmod 600 ~/locationOfPrivateKey.pem
5. from terminal type ssh -i ~/key.pem ubuntu@54.71.80.71 #IP_Address
6. Once connected, update and upgrade packages with commands:
```
sudo apt-get update
sudo apt-get upgrade
```
7. Change SSH to 2200 by command:
```
sudo nano /etc/ssh/sshd_config
```
then locate portnumber, and change to 2200. 
Save and exit then restart ssh by:
```
sudo service ssh restart
```
8. Configure UFW
a. Configure firewall to only allow SSH Port:2200, HTTP Port:80, and NTP Port:123 by command:
```
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

If 22 enabled, you can disable by:
```
sudo ufw deny 22
```
or remove rule from ufw by:
```
sudo ufw status numbered
sudo ufw delete [number]
```
b. Once done, check whether ufw is enabled by command: sudo ufw status 
c. make sure to goto Amazon Lighsail instance on your browser and configure firewall ports as well.
9. Create user grader and give access
a. while loggedin as ubuntu user type:
```
sudo adduser grader
```
b. type in password.
c. create file /etc/sudoers.d/grader, and type in grader ALL=(ALL:ALL) ALL
d. generate private key in local machine by: ssh-keygen and copy contents
e. goto grader by su - grader
f. create new director by mkdir ~/.ssh
g. inside create file authorized_keys and paste in content.
h. give permission by chmod 700 .ssh and chmod 644 .ssh/authorized_keys
i. goto /etc/ssh/ssh_config and set PasswordAuthentication to no and PermitRootLogin to no
j. sudo service ssh restart
k. from terminal login to grader by: 
```
ssh grader@54.71.80.71 -p 2200 -i ~/.ssh/grader
```
10. Configure timezone to UTC by: 
```
sudo dpkg-reconfigure tzdata
```
Should be set to UTC by default
11. Install apache2 and wsgi package and git
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi-py3
sudo apt-get install git
```
Once done restart Apache:
```
sudo service apache2 restart
```
12. Install and configure PostgreSQL
a. install
```
sudo apt-get install postgresql
```
b. then check if no remote connections are allowed:
```
sudo nano /etc/postgresql/9.5/main/pg_hpa.conf
```
c. login to postgresql by sudo su - postgres and type psql 
d. create db name catalog and new user catalog, give password and grant permission by:
```
CREATE DATABASE catalog;
CREATE USER catalog;
ALTER ROLE catalog WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
\q
exit
```

13. Download ItemCatalog App and Configure
a. goto /var/www/ and git clone from item catalog repository. use mv to change name of folder to catalog
b. rename application.py to __init__.py using mv function again
c. for database_setup and __init__ and seed_data change create_engine to 'postgresql://catalog:password@localhost/catalog'
d. Install virtual environment and python3:
```
sudo apt-get install python3-pip
sudo apt-get install python-virtualenv
``` 
in /var/www/catalog/catalog directory create virtual env by:
```
sudo virtualenv -p python3 ven3
sudo chown -R grader:grader venv3
```
activate environment by . venv3/bin/activate
e. Install all dependencies by 
```
sudo pip install sqlalchemy flask-sqlalchemy psycopg2 requests flask oauthclient httplib2
```
f. create database schema by running 
```
python database_setup.py
```
g. run python3 __init__.py to make sure everything is running and all dependcies are installed then deactivate venv
h. goto /etc/apache2/mods-enabled/wsgi.conf makmake sure WSGIPythonPath uses venv3 python3.5
i. create by nano /etc/apache2/sites-available/catalog.conf by add:
```
<VirtualHost *:80>
        ServerName 54.71.80.71
        ServerAdmin johnwschoi@gmail.com
        WSGIDaemonProcess catalog user=www-data group=www-data threads=5
        WSGIProcessGroup catalog
        WSGIApplicationGroup %{GLOBAL}
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
then enable site by and disable default apache site by: 
```
sudo a2dissite 000-default.conf
sudo a2ensite catalog
sudo service apache2 restart
```
j. create by nano /var/www/catalog/catalog.wsgi add:
```
activate_this = '/var/www/catalog/catalog/venv3/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")
sys.path.insert(1,"/var/www/")

from catalog import app as application
application.secret_key = "secret_key"
```
and again
```
sudo service apache2 restart
```
k. change google oauth:
goto google oauth 2.0 and add in the new ip and address. 
make sure to add .xip.io after ip address
Once done, save and retrive client_secrets.json from site and overwrite it

14.
to see error log:
```
sudo tail /var/log/apache2/error.log
```
and remember so sudo service apache2 restart everytime you make changes
also: 
```
# For relative imports to work in Python 3.6
import os, sys; sys.path.append(os.path.dirname(os.path.realpath(__file__)))
```
also:
```
basedir = os.path.abspath(os.path.dirname(__file__))
client_secrets = basedir + '/client_secrets.json'

CLIENT_ID = json.loads(
    open(client_secrets, 'r').read())['web']['client_id']
```

## List of any third-party resources used
*Udacity Linux Config Learning Page & Project Details
*https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
*http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/
*https://blog.codeasite.com/how-do-i-find-apache-http-server-log-files/
*https://serverfault.com/questions/262751/update-ubuntu-10-04/262773#262773
*https://www.vultr.com/docs/how-to-configure-ufw-firewall-on-ubuntu-14-04

