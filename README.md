# FSND Linux Configuration


## Introduction
A baseline installation of Ubuntu Linux on a virtual machine to host a Flask web application. This includes the installation of updates, securing the system from a number of attack vectors and installing/configuring web and database servers.

## Server Details
* IP address: 13.127.211.49
* SSH port: 2200
* URL: http://ec2-13-127-211-49.ap-south-1.compute.amazonaws.com/

## Summary of software installed and configuration changes made

### Create Development Environment: Launch Virtual Machine and SSH into the server
* Download Amazon Lightsail private key

* Move the private key file into the ssh folder:
```
$ mv ~/Downloads/udacity.pem ~/.ssh/
```
* Set file rights (only owner can write and read):
```
$ chmod 600 ~/.ssh/udacity.pem
```
* Login to your remote VM:
```
$ ssh -i ~/.ssh/udacity.pem ubuntu@13.127.211.49
```
### Create a new user named grader with a secure password and give grader the permission to 'sudo'

* Add new user grader:
```
 $ sudo adduser grader
 ```
 * Give grader sudo access:
 ```
 # copy sudoers file as grader
 $ sudo cp /etc/sudoers.d/ubuntu /etc/sudoers.d/grader
 $ sudo nano /etc/sudoers.d/grader
 # Edit "grader ALL=(ALL:ALL) ALL", save it
 ```

### Update all currently installed packages
* Update packages:
```
$ sudo apt update
$ sudo apt upgrade
```
* Install finger, a utility software to check user's status: 
```
$ apt-get install finger
```

### Change the SSH port from 22 to 2200 and configure SSH access
* Generate an encryption key on your local machine with:
```
 $ ssh-keygen -f ~/.ssh/udacity_key.rsa
 ```
* Create authorized_keys file and copy the content of udacity_key.pub
```
$ sudo mkdir /home/grader/.ssh
$ sudo touch /home/grader/.ssh/authorized_keys 
$ sudo nano /home/grader/.ssh/authorized_keys
# Paste udacity_ key.pub
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys
$ sudo chown -R grader:grader /home/grader/.ssh
```
* Login using:
```
$ ssh -i ~/.ssh/udacity_key.rsa grader@13.127.211.49
```

### Enforce key-based authentication
```
$ sudo nano /etc/ssh/sshd_config
# Find the PasswordAuthentication line and edit it to no
$ sudo service ssh restart
```

### Disable ssh login for root user
```
$ sudo nano /etc/ssh/sshd_config
# Find the PermitRootLogin line and edit it to no
$ sudo service ssh restart
```

### Change the SSH port from 22 to 2200
```
$ sudo nano /etc/ssh/sshd_config
#Find the Port line and edit it to Port 2200 from Port 22
$ sudo service ssh restart.
# Now you are able to login to port 2200 using the following command: 
$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@13.127.211.49
```

### Configure the Uncomplicated Firewall (UFW)
```
$ sudo ufw status
$ sudo ufw default deny incoming 
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable
```

### Configure the local timezone to UTC
```$ sudo dpkg-reconfigure tzdata```
* Use the arrow keys to choose the bottom option None of the Above
* Press the u key until the UTC option is selected and press Return

### Include cron scripts to automatically manage package updates
* Install the unattended-upgrades package:
```$ sudo apt-get install unattended-upgrades```
* Enable the unattended-upgrades package:
```$ sudo dpkg-reconfigure -plow unattended-upgrades```

### Install Apache, mod_wsgi, Git
* Install apache:
```
$ sudo apt-get install apache2
```
* Install mod_wsgi:
```
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo apt install libapache2-mod-wsgi-py3
```
* Enable mod_wsgi:
```
$ sudo a2enmod wsgi
```
* Install git:
```
$ sudo apt-get install git
```
Configure GitHub username:
```
$ git config --global user.name rhutujashevde
```
Configure GitHub email: 
```
$ git config --global user.email rshevde.555@gmail.com
```
* Start apache
```
$ sudo service apache2 start
```

### Clone the Catalog app from Github
* Create a directory to clone the project:
```
$ cd /var/www
$ sudo mkdir catalog
```
* Change owner for the catalog folder:
```
$ sudo chown -R grader:grader catalog
```
* cd inside catalog directory:
```
$ cd /catalog
```
* Clone the item catalog repository from Github: 
```
$ git clone https://github.com/rhutujashevde/anime_catalog catalog
```
* Create a catalog.wsgi file
```
$ sudo touch /var/www/catalog/catalog.wsgi
```
* Write the following code in the file:
```
# !/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'thisisasecret'
application.config['DATABASE_URL'] = 'postgresql://catalog:udacity@localhost/catalog'
```

#### Install virtual environment, Flask and other packages
* Install pip
```
$ sudo apt-get install python-pip
```
* Install and activate virtual env
```
$ sudo pip install virtualenv
$ cd /var/www/catalog
$ sudo virtualenv venv
$ source venv/bin/activate
$ sudo chmod -R 777 venv
```
* Install Flask and other packages
```
$ pip install Flask httplib2 requests oauth2client flask-sqlalchemy psycopg2
```

#### Configure and enable a new virtual host
* Create a virtual host conifg file:
```
$ sudo nano /etc/apache2/sites-available/catalog.conf
```
* Paste in the following lines of code:
```
<VirtualHost *:80>
                ServerName 13.127.211.49
                ServerAlias ec2-13-127-211-49.ap-south-1.compute.amazonaws.com
                ServerAdmin admin@13.127.211.49
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
* Enable the new virtual host:
```
$ sudo a2ensite catalog
```
* Restart Apache
```$ sudo service apache2 restart``` 

#### Install and configure PostgreSQL
* Install Python packages for working with PostgreSQL:
```
$ sudo apt-get install libpq-dev python-dev
```
* Install PostgreSQL:
```
$ sudo apt-get install postgresql postgresql-contrib
```
* Postgres is automatically creating a new user during its installation called 'postgres'
* Change the user with:
```
$ sudo su - postgres
```
* Connect to the database system with
```$ psql```
```
1. Create a new user called 'catalog':
# CREATE USER catalog WITH PASSWORD 'udacity';
2. Give catalog user the CREATEDB capability:
# ALTER USER catalog CREATEDB;
3. Create the 'catalog' database owned by catalog user:
# CREATE DATABASE catalog WITH OWNER catalog;
4. Connect to the database:
# \c catalog
5. Revoke all rights:
# REVOKE ALL ON SCHEMA public FROM public;
6. Lock down the permissions to only let catalog role create tables:
# GRANT ALL ON SCHEMA public TO catalog;
7. Log out from PostgreSQL:
# \q
```
* Return to the grader user:
```$ exit```
* Inside the Flask application, the database connection is now performed with:
```app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://catalog:udacity@localhost/catalog'```
* To prevent potential attacks from the outer world we double check that no remote connections to the database are allowed.
* Open the following file:
```
$ sudo cat /etc/postgresql/9.5/main/pg_hba.conf:
```
* It should look something like this:
```
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
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

###  Update OAuth authorized JavaScript origins
* Fill in the client_id and client_secret fields in the file client_secrets.json.
* Change the javascript_origins field to the IP address and AWS assigned URL of the host.
* These addresses also need to be entered into the Google Developers Console -> API Manager -> Credentials, in the web client under "Authorized JavaScript origins".

## List of third-party resources used to complete this project:
http://flask.pocoo.org/docs/0.12/deploying/#deployment
https://lightsail.aws.amazon.com/ls/docs/en/articles/getting-started-with-amazon-lightsail
https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-set-up-ssh
https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server
https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29
https://blog.udacity.com/2015/03/step-by-step-guide-install-lamp-linux-apache-mysql-python-ubuntu.html
https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu
http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
https://stackoverflow.com/questions/12201928/python-open-gives-ioerror-errno-2-no-such-file-or-directory

## NOTE
* This app uses flask-sqlalchemy insted of sqlalchemy, you can check the unchanged code [here](https://github.com/rhutujashevde/anime_catalog).
* This app is also deployed on Heroku, check it out [here](https://anime-catalog.herokuapp.com/).
* Heroku app uses .sqlite database created in the initial stages of the project so don't be confused.
