# Linux Server Configuration
Project 6 of the **Udacity Full Stack Web Developer Nanodegree**

## About the project
Take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

- IP address: 18.216.200.133
- Accessible SSH port: 2200
- AWS-Server: http://ec2-18-216-200-133.us-east-2.compute.amazonaws.com
- Loginin as grader: SSH key can be found in the "Notes to Reviewer" field, command to access server: 
`ssh grader@34.214.154.162 -p 2200 -i grader`

## Step by Step Walkthrough

### Secure the server
1. Update all currently installed packages
- `sudo apt-get update`
- `sudo apt-get upgrade`
- `sudo apt autoremove`

2. Change the SSH port from 22 to 2200
- `sudo nano /etc/ssh/sshd_config`
- Change the port from 22 to 2200

3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
- `sudo ufw allow 2200/tcp`
- `sudo ufw allow www`
- `sudo ufw allow 123/tcp`
- `sudo ufw enable`

### Create new user named grader and give it the permission to sudo
- `sudo adduser grader`
- set password as "grader"
- `sudo nano /etc/sudoers.d/grader`
- Add following content to the file:
 `grader ALL=(ALL) NOPASSWD:ALL`
- Create a key for `grader` in local machine with `ssh-keygen`
- Run the following commands to allow `grader` login with the key:
```
sudo mkdir /home/grader/.ssh
sudo chown grader:grader /home/grader/.ssh # changing ownership of .ssh to grader
sudo chmod 700 /home/grader/.ssh           # change folder permission
sudo cp /home/ubuntu/.ssh/authorized_keys /home/grader/.ssh/
sudo nano /home/grader/.ssh/authorized_keys
sudo chmod 644 /home/grader/.ssh/authorized_keys
```
- Now the grader can login using the following command: `ssh grader@18.216.200.133 -p 2200 -i ~/.ssh/grader`

### Configure the local timezone to UTC.
Source: https://help.ubuntu.com/community/UbuntuTime
- `sudo dpkg-reconfigure tzdata`
- In the window that appears, use the arrow keys to Scroll to the bottom of the Continents list and select Etc or None of the Above and then in the second list, select UTC 

### Install and configure Apache to serve a Python mod_wsgi application.
- `sudo apt-get install apache2`
- `sudo apt-get install libapache2-mod-wsgi python-dev`
- `sudo service apache2 start`

### Install and configure PostgreSQL
Source: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps#do-not-allow-remote-connections
- `sudo apt-get install libpq-dev python-dev`
- `sudo apt-get install postgresql postgresql-contrib`
- `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
- Make sure the following appears in `pg_hba.conf`:
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
- `sudo -u postgres psql`
- Create a new database user named `catalog` within psql:
```
CREATE USER catalog WITH PASSWORD 'catalogpassword';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
```

### Install git and clone `item_catalog` file created earlier in this Nanodegree program
- `sudo apt-get install git`
- `cd /var/www`
- `sudo mkdir catalog`
- `cd catalog`
- `git clone https://github.com/welcomeying/item_catalog.git catalog`
- `sudo nano catalog.wsgi`
- Add the following content to the `catalog.wsgi` file:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")
from catalog import app as application
application.secret_key = 'supersecretkey'
```
- Rename `main/py` to `__init_.py`: `mv catalog/main.py catalog/__init__.py`

### Install virtual environment
- `sudo apt install python-pip`
- `sudo pip install virtualenv`
- `sudo virtualenv venv`
- `source venv/bin/activate`
- `sudo chmod -R 777 venv`

### Install needed modules & packages
- `pip install flask`
- `pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests`

### Update files in cloned `catalog` directory
- `nano __init__.py`
- Change `client_secrets.json` path to `/var/www/catalog/catalog/client_secrets.json`
- Change `create engine` line in `__init__.py` and `database_setup.py` and `lotsofitems.py` to: `engine = create_engine('postgresql://catalog:catalogpassword@localhost/catalog')`
- `python database_setup.py`
- `python lotsofitems.py`

### Get Google login working
- Go to: https://console.developers.google.com
- Find catalog app credentials and add URLs as follows:
Authorized JavaScript origins: http://18.216.200.133, http://ec2-18-216-200-133.us-east-2.compute.amazonaws.com
Authorized redirect URIs: http://ec2-18-216-200-133.us-east-2.compute.amazonaws.com/login, http://ec2-18-216-200-133.us-east-2.compute.amazonaws.com/gconnect

### Configure and enable a new virtual host
Source: http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
- `sudo nano /etc/apache2/sites-available/catalog.conf`
- Add the following content to `catalog.conf`
```
<VirtualHost *:80>
    ServerName 18.216.200.133
    ServerAlias ec2-18-216-200-133.us-east-2.compute.amazonaws.com
    ServerAdmin admin@18.216.200.133
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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
- Remove default.conf and from being enabled:
`sudo a2dissite 000-default.conf`
- Enable the virtual host:
`sudo service apache2 reload`
`sudo a2ensite catalog`
- Visit site at http://18.216.200.133
