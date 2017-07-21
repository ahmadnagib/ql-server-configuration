# QuantumLeap Server Configuration

QuantumLeap (QL) is a Linux Ubuntu server hosting [Quantum Leap Catalog](https://github.com/ahmadnagib/QuantumLeapCatalog) web application. A collection of configuration changes has been made on the server to be able to serve the application properly and securely. An overview of the carried out changes is included in this README file.

This project is part of Udacity's Full Stack Web Developer Nanodegree Program. References to webpages that were relevant to the configuration process are included in this documentation.

- [Accessing the Amazon Lightsail Instance](#accessing-the-amazon-lightsail-instance)
- [URL of QuantumLeap Catalog Application](#url-of-quantumleap-catalog-application)
- [Server Configurations](#Server-Configurations)
    + [Updating Installed Packages](#Updating-Installed-Packages)
    + [Changing the SSH port](#Changing-the-SSH-port)
    + [Configuring the Uncomplicated Firewall](#Configuring-the-Uncomplicated Firewall)
    + [Creating grader User](Creating-grader-User)
    + [Giving grader permission to sudo](#Giving-grader-permission-to-sudo)
    + [Key Based Authentication](#Key-Based-Authentication)
    + [Configuring the local timezone to UTC](#Configuring-the-local-timezone-to-UTC)
- [Software Installed](#Software-Installed)
    + [Apache Server](#Apache-Server)
    + [PostgreSQL](#PostgreSQL)
    + [Git](#Git)
- [Depolying QuantumLeap Catalog](#Depolying-QuantumLeap-Catalog)
    + [Install Needed Python Packages](#Install-Needed-Python-Packages)
    + [Switch from SQLite to PostgreSQL](#Switch-from-SQLite-to-PostgreSQL)
    + [Configure flask Flow](#Configure-flask-Flow)
    + [Configure Domain Name](#Configure-Domain-Name)
    + [Configure OAuth 2.0 client ID](Configure-OAuth-2.0-client-ID)
- [References](#References)


## Accessing the Amazon Lightsail Instance

This project requires a Linux server instance to serve the QuantumLeap Catalog application as a WSGI application. [Amazon Lightsail](lightsail.aws.amazon.com) was used to fulfill this requirement. The detailed steps for creating an Ubuntu server on lightsail can be found [here](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html).

![amazon-lightsail-instance-homepage](https://user-images.githubusercontent.com/13169976/28465356-b6bff0c8-6e28-11e7-9b13-65dae55b2a67.png)

As seen in the Amazon Lightsail instance dashboard, the public IP address of the created instance for this project is 35.176.115.62. The server can be accessed using Git Bash via SSH. The command `ssh grader@35.176.115.62 -p 2200 -i /PATH_TO_THE_USER_PRIVATE_KEY` should be used.

"grader" is the created Ubuntu user that a Udacity reviewer can use to review the project. Additionally, "PATH_TO_THE_USER_PRIVATE_KEY" is the path of the private key on the connecting local machine to be able to log into the server as "grader".

It should be noticed that port 2200 is the port used for SSH instead of the default port 20 which was disabled manually on this instance. Logging into the instance can also be done using PuTTY. More details about how to do this can be found at this [link](https://lightsail.aws.amazon.com/ls/docs/how-to/article/lightsail-how-to-set-up-putty-to-connect-using-ssh). 


## URL of QuantumLeap Catalog Application

The [Quantum Leap Catalog](https://github.com/ahmadnagib/QuantumLeapCatalog) web application can be accessed via [www.quantumleap.cf](http://www.quantumleap.cf/).


## Server Configurations 

This sections includes the configuration changes made on the ubuntu server instance.

### Updating Installed Packages

`ubuntu@ip-172-26-15-133:~$ sudo apt-get update`
`ubuntu@ip-172-26-15-133:~$ sudo apt-get upgrade`
`ubuntu@ip-172-26-15-133:~$ sudo apt-get autoremove`

### Changing the SSH port
Open `sshd_config` file with nano editor
`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/ssh/sshd_config`

Change the SSH port from 22 to 2200
#### from
```
# What ports, IPs and protocols we listen for
Port 22
```
#### to
```
# What ports, IPs and protocols we listen for
Port 2200
```

Additionally, remote SSH Login as root user can be disabled by changing `PermitRootLogin` directive.

#### from
```
# Authentication:
LoginGraceTime 120
PermitRootLogin prohibit-password
StrictModes yes
```
#### to
```
# Authentication:
LoginGraceTime 120
PermitRootLogin no
StrictModes yes
```

More details about SSH port configuration and remote root login can be found [here](https://ae.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306) and [here](https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server).

Make sure that custom port 2200 is also allowed in the Amazon Lightsail instance firewall settings from the networking tab as shown below.

![amazon-lightsail-instance-networking-tab](https://user-images.githubusercontent.com/13169976/28465357-b6c2ba2e-6e28-11e7-9bd1-aa7e2d60cba7.png)

Restart ssh service for changes to take effect
`ubuntu@ip-172-26-15-133:~$ service sshd restart`


### Configuring the Uncomplicated Firewall

Allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

```
ubuntu@ip-172-26-15-133:~$ sudo ufw allow 2200/tcp
ubuntu@ip-172-26-15-133:~$ sudo ufw allow www
ubuntu@ip-172-26-15-133:~$ sudo ufw allow ntp
```

Make sure that everything is fine before enabling the Uncomplicated Firewall (UFW)
`ubuntu@ip-172-26-15-133:~$ sudo ufw status`

enable the configured UFW
`ubuntu@ip-172-26-15-133:~$ sudo ufw enable`

The status by that time should be
```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123                        ALLOW       Anywhere
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123 (v6)                   ALLOW       Anywhere (v6)
```

Additionally, the lightsail instance's firewall should only allow these three port numbers. More details about UFW can be found [here](https://help.ubuntu.com/community/UFW)

### Creating grader User
`ubuntu@ip-172-26-15-133:~$ sudo adduser grader`

### Giving grader permission to sudo

The existing configuration for "ubuntu" initial user should be copied to another file named "grader" 
`ubuntu@ip-172-26-15-133:~$ sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`

Then the file should be edited
`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/sudoers.d/grade`

change "ubuntu" user in the file to be "grader" user

#### from
```
# Created by cloud-init v. 0.7.9 on Wed, 19 Jul 2017 09:03:25 +0000

# User rules for ubuntu
ubuntu ALL=(ALL) NOPASSWD:ALL
```
#### to
```
# Created by Ahmad Nagib (ubuntu) on Wed, 19 Jul 2017 09:15:30 +0000

# User rules for grader
grader ALL=(ALL) NOPASSWD:ALL
```


### Key Based Authentication

Create an SSH key pair for "grader" user using the ssh-keygen tool from Git Bash on the local connecting machine with no passphrase.

```
Ahmad Nagib@Ahmad MINGW64 /f/Downloads
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/Ahmad Nagib/.ssh/id_rsa): graderkey
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in graderkey.
Your public key has been saved in graderkey.pub.
```


`ubuntu@ip-172-26-15-133:~$ sudo su grader`
`grader@ip-172-26-15-133:~$ cd`
`grader@ip-172-26-15-133:~$ mkdir .ssh`
`grader@ip-172-26-15-133:~$ sudo nano .ssh/authorized_keys`

paste the public key from `graderkey.pub` file on your local machine to `authorized_keys` file created on the server.

Change the permissions on the folder and file to secure it from any undesired access

`grader@ip-172-26-15-133:~$ chmod 700 .ssh`
`grader@ip-172-26-15-133:~$ chmod 644 .ssh/authorized_keys`



### Configuring the local timezone to UTC

`ubuntu@ip-172-26-15-133:~$ sudo  timedatectl set-timezone Etc/UTC`



## Software Installed

This section includes a summary of software that was installed on the Ubuntu server instance.

### Apache Server
- Install and configure Apache to serve a Python mod_wsgi application.

`ubuntu@ip-172-26-15-133:~$ sudo apt-get install apache2`
`ubuntu@ip-172-26-15-133:~$ sudo apt-get install libapache2-mod-wsgi`

- Make sure that port 80 is used by Apache server. Open `ports.conf` file and check whether `Listen` directive is set to `80`

`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/apache2/ports.conf`

`Listen 80`

- Error logs of Apache server can always be checked by viewing `/var/log/apache2/error.log` file.

`ubuntu@ip-172-26-15-133:~$ sudo cat /var/log/apache2/error.log`



### PostgreSQL

- Install PostgreSQL

`ubuntu@ip-172-26-15-133:~$ sudo apt-get install postgresql`

- Create a new database user `catalog`
`ubuntur@ip-172-26-15-133:~$ sudo -u postgres createuser --interactive`

- Make sure to give this user limited permissions
```
Enter name of role to add: catalog
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) n
Shall the new role be allowed to create more new roles? (y/n) n
```

- Edit the PostgreSQL Client Authentication Configuration File to map `www-data` Ubuntu user to the created `catalog` PostgreSQL user. This would allow the web application to connect to the database.

`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

- At least add this line at after `Database administrative login by Unix domain socket`

`local   catalog         catalog                                 peer map=wwwmap`

- Additionally, make sure that all the hosts in the PostgreSQL Client Authentication Configuration File `pg_hba.conf` do not have remote addresses. This disallows any remote database connections.

- Then create the map in the PostgreSQL User Name Maps file `pg_ident.conf` 

`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/postgresql/9.5/main/pg_ident.conf`

- Add this line at the end of the file
`wwwmap          www-data                catalog`

- Reload the PostgreSQL service 
`ubuntu@ip-172-26-15-133:~$ sudo /etc/init.d/postgresql reload`

- Now log into PostgreSQL and create a new `catalog` database
```
ubuntu@ip-172-26-15-133:~$ sudo -u postgres psql
psql (9.5.7)
Type "help" for help.

postgres=# CREATE DATABASE test;
```

-Additionally, restrict connection to the `catalog` database to `catalog` user only. More details can be found [here](https://dba.stackexchange.com/questions/17790/created-user-can-access-all-databases-in-postgresql-without-any-grants)
```
postgres=# REVOKE connect ON DATABASE catalog FROM PUBLIC;
postgres=# GRANT connect ON DATABASE catalog TO catalog;
```

### Git
- Install git
`ubuntu@ip-172-26-15-133:~$ sudo apt-get install git`

- Clone the Quantum Leap Catalog project files to the /var/www directory
```
ubuntu@ip-172-26-15-133:~$ cd /var/www
ubuntu@ip-172-26-15-133:/var/www$ sudo git clone https://github.com/ahmadnagib/QuantumLeapCatalog
```

- Change the name of the folder and the name of a file in the folder
```
ubuntu@ip-172-26-15-133:~$ sudo mv /var/www/QuantumLeapCatalog /var/www/quantumleap.cf`
ubuntu@ip-172-26-15-133:~$ sudo mv /var/www/quantumleap.cf/qlcatalog.py /var/www/quantumleap.cf/qlcatalog.wsgi`
```

After these few changes the QuantumLeap Catalog web application directory view is as follows:
```
/var/www/quantumleap.cf/
├──  models/
    ├── __init_.py
    ├── base.py
    ├── category.py
    ├── item.py
    ├── user.py
├──  views/
    ├── static/
        ├── main.css
    ├── templates/
        ├── addcategory.html
        ├── additem.html
        ├── allcategories_public.html
        ├── base.html
        ├── category.html
        ├── category_public.html
        ├── categoryitems_public.html
        ├── deletecategory.html
        ├── deleteitem.html
        ├── editcategory.html
        ├── edititem.html
        ├── item.html
        ├── item_public.html
        ├── login.html
    ├── app.py
    ├── categoryitems.py
    ├── databasesession.py
    ├── deletecategory.py
    ├── deleteitem.py
    ├── editcategory.py
    ├── edititem.py
    ├── getcategories.py
    ├── getcategory.py
    ├── getitem.py
    ├── login.py
    ├── logout.py
    ├── newcategory.py
    ├── newitem.py
    ├── utils.py
├── LICENSE
├── qlcatalog.wsgi
├── README.md
├── secrets_of_g_client.json
```


## Depolying QuantumLeap Catalog

### Install Needed Python Packages
Install all the needed packages so that the web application works properly

#### Install pip
```
ubuntu@ip-172-26-15-133:~$ sudo apt-get install python-pip`
ubuntu@ip-172-26-15-133:~$ sudo pip install --upgrade pip
```

#### Install requests package
`ubuntu@ip-172-26-15-133:~$ sudo -H pip install requests`

#### Install flask package
It should be noticed that [QuantumLeap Catalog](https://github.com/ahmadnagib/QuantumLeapCatalog) requires specific versions to work properly.
```
ubuntu@ip-172-26-15-133:~$ sudo -H pip install flask
ubuntu@ip-172-26-15-133:~$ sudo -H pip install werkzeug==0.8.3
ubuntu@ip-172-26-15-133:~$ sudo -H pip install flask==0.9
ubuntu@ip-172-26-15-133:~$ sudo -H pip install Flask-Login==0.1.3
```

#### Install sqlalchemy package
`ubuntu@ip-172-26-15-133:~$ sudo -H pip install sqlalchemy`

#### Install psycopg2 package
`ubuntu@ip-172-26-15-133:~$ sudo -H pip install psycopg2`

#### Install oauth2client package
`ubuntu@ip-172-26-15-133:~$ sudo -H pip install oauth2client`


### Switch from SQLite to PostgreSQL

It should be noticed that [QuantumLeap Catalog](https://github.com/ahmadnagib/QuantumLeapCatalog) uses an SQLite database. This could be simply switched into PostgreSQL by editing the engine url in both `/var/www/quantumleap.cf/views/databasesession.py` and `/var/www/quantumleap.cf/models/__init__.py` files.

The url should look like `engine = create_engine("postgresql://username:password@localhost/catalog")`. More details about `create_engine` can be found [here](http://docs.sqlalchemy.org/en/latest/core/engines.html).

### Configure flask Flow

A slight modification was made in two files to prevent any server errors. The two files are `/var/www/quantumleap.cf/qlcatalog.wsgi` and `/var/www/quantumleap.cf/views/app.py`

- Edit `/var/www/quantumleap.cf/qlcatalog.wsgi` to look as follows:
```
import sys
sys.path.insert(0,"/var/www/quantumleap.cf/")

from views import app as application
```

- Edit `/var/www/quantumleap.cf/views/app.py` to look as follows:
```
from flask import Flask
app = Flask(__name__)

app.secret_key = "IAM_AHMAD_NAGIB"

if __name__ == '__main__':
        app.run(debug=True)
```

- Additionally `/var/www/quantumleap.cf/views/login.py` should be edited to be able to access the client secret file and work properly.

CLIENT_ID in line 35 should be like
```
# get the client_id from the client secret json file
CLIENT_ID = json.loads(
    open(
        '/var/www/quantumleap.cf/secrets_of_g_client.json',
        'r').read())['web']['client_id']
```
gplus_oauth_flow in line 77 should be like
```
# get a credentials object using the authorization code
        gplus_oauth_flow = flow_from_clientsecrets(
            '/var/www/quantumleap.cf/secrets_of_g_client.json', scope='')
```

Without doing this modification any client browser will show an `Internal Server Error` instead of running the application properly. More details about deploying a flask application on an Ubuntu server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps), [here](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/), and [here](http://nikles.it/block-direct-ip-access-to-your-server-in-apache-2-4/).


### Configure Domain Name

The domain name quantumleap.cf was used for the web application. A free domain name can be got from websites like [freenom.com](http://www.freenom.com). This was used to replace the IP address of the ubuntu instance. This was done by configuring Apache server virtual hosts.

- Create a new virtual host configuration file for QuantumLeap Catalog web application
`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/apache2/sites-available/quantumleap.cf.conf`

- Edit the file to look as follows:
```
<VirtualHost *:80>
                ServerAdmin ahmadnagib@cu.edu.eg
                ServerName quantumleap.cf
                ServerAlias www.quantumleap.cf
                WSGIScriptAlias / /var/www/quantumleap.cf/qlcatalog.wsgi
                <Directory /var/www/quantumleap.cf>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/quantumleap.cf/views/static
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Additionally, accessing the server directly using IP address was disabled. This can be carried out by adding a new virtual host.
`ubuntu@ip-172-26-15-133:~$ sudo nano /etc/apache2/sites-available/direct.conf`

- Edit the file to look as follows:
```
<VirtualHost *:80>
    ServerName 35.176.115.62
    Redirect 403 /
    ErrorDocument 403 "Direct IP access is not allowed. Kindly visit our site at www.quantumleap.cf"
    DocumentRoot /var/www/html
</VirtualHost>
```

- Enable the two created virtual hosts
```
ubuntu@ip-172-26-15-133:~$ sudo a2ensite quantumleap.cf
ubuntu@ip-172-26-15-133:~$ sudo a2ensite direct
```

- Reload the Apache service
`ubuntu@ip-172-26-15-133:~$ sudo service apache2 reload`

- After these steps the web application can be successfully accessed at [www.quantumleap.cf](http://www.quantumleap.cf/). If any client browser tried to access the server using its IP address [http://35.176.115.62/](http://35.176.115.62/) it be disallowed .

For more details, kindly check this [link](https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts).

### Configure OAuth 2.0 client ID
- Configure OAuth 2.0 client ID to include the new url. Otherwise an `origin_mismatch` error will happen in case of trying to authenticate with google account to login to the QuantumLeap Catalog.


Authorized JavaScript origins should include `http://quantumleap.cf` and `http://www.quantumleap.cf`. Additionally, Authorized redirect URIs preferably should include `http://quantumleap.cf/login` and `http://quantumleap.cf/google-auth` as shown below.

![google-cloud-platform](https://user-images.githubusercontent.com/13169976/28465358-b6c3f966-6e28-11e7-8b00-b3c180315e8d.png)

## References
