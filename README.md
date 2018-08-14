# Project: Linux Server Configuration  - Savneet Singh

README for running the Item Catalog web application live on a Linux Ubuntu server instance on Amazon Lightsail. 

Public IP: 13.232.212.122

SSH Port: 2200

URL: http://13.232.212.122.xip.io/
  
## Update and upgrade the Server
```
sudo apt-get update
sudo apt-get upgrade
```
## Create a grader user and give sudo access to grader
```
•	sudo adduser grader
•	Give pasword 
   • Give full name as Grader and press enter the skip other fields
•	sudo visudo
	• add grader ALL=(ALL) ALL
```

## Create an ssh key-pair for grader
In local machine:
```
•	ssh-keygen
•	Enter the file (gives /home/ubuntu/.ssh/id_rsa), I typed linuxProject /path-to-ssh-directory/.ssh/linuxServerProjectKey
•	Give passphrase 
•	cat .ssh/linuxServerProjectKey.pub
•	Copy contents
```

In lightsail terminal:
```
•	sudo su - grader
•	mkdir .ssh
•	chmod 700 .ssh
•	touch .ssh/authorised_keys
•	chmod 644 .ssh/authorised_keys
•	nano .ssh/authorised_keys
•	Paste contents of linuxProject.pub and save file
•	exit
```

## Configure the ports and firewalls

In lightsail console add port 2200 and 123 under the networking tab.
#### Ports
Then in light sail terminal
```
•	sudo nano /etc/ssh/sshd_config (add port 2200, keep port 22)
•	sudo service sshd restart
```
#### Firewalls
In light sail terminal
```
•	sudo ufw default deny incoming
•	sudo ufw default allow outgoing
•	sudo ufw allow 2200/tcp
•	sudo ufw allow ssh
•	sudo ufw allow www
•	sudo ufw allow 80
•	sudo ufw allow 123
•	sudo ufw enable
```
(You can ssh from local terminal using port 22 and 2200 successfully)
#### Configure the local timezone to UTC
```
• sudo dpkg-reconfigure tzdata
    • select "None of the above"
    • select "UTC"
```

#### Remove Port 22
In light sail terminal
```
•	nano /etc/ssh/sshd_config (remove port 22, only port 2200 remains)
•	sudo ufw deny 22
```

In lightsail console remove port 22 under the networking tab. You will not be able to use the Connect using SSH button in the lightsail console from now on.

Now I can login remotely as grader by running 
```
ssh grader@publicIP -p 2200 -i ~/.ssh/linuxServerProjectKey
```

## Install and configure Apache to serve a Python mod_wsgi application
```
• sudo apt-get install apache2
• sudo apt-get install libapache2-mod-wsgi python-dev
• sudo a2enmod wsgi (to enable mod wsgi if it isn't enabled)
• sudo service apache2 restart
(now if you go to your public ip address in your browser you should see an apache2 message)
```

## Install git
```
• sudo apt-get install git
• cd /var/www
• sudo mkdir ItemCatalog
• cd ItemCatalog
• sudo git clone https://github.com/savisingh/Udacity-Item-Catalog-Project.git ItemCatalog 
```

## Create wsgi and conf files for Item Catalog application
```
• cd /var/www/ItemCatalog/ItemCatalog/
• sudo nano itemCatalogApp.wsgi
```
Inside itemCatalogApp.wsgi paste this:
```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/ItemCatalog/ItemCatalog/')

from application import app as application
application.secret_key='super_secret_key'
```

Now to create the conf file:
```
• cd /etc/apache2/sites-available
• sudo nano itemCatalog.conf
```
Inside itemCatalog.conf paste:
```
<VirtualHost *:80>
     ServerName  PublicIP
     ServerAdmin email address
     #Location of the items-catalog WSGI file
     WSGIScriptAlias / /var/www/ItemCatalog/ItemCatalog/itemCatalogApp.wsgi
     #Allow Apache to serve the WSGI app from our catalog directory
     <Directory /var/www/ItemCatalog/ItemCatalog/>
          Order allow,deny
          Allow from all
     </Directory>
     #Allow Apache to deploy static content
     <Directory /var/www/ItemCatalog/ItemCatalog/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
 
 Then run
 ```
• sudo a2dissite 000-default.conf
• sudo s2ensite itemCatalog.conf
sudo service apache2 restart
```
## Install and configure PostgreSQL and Flask
```
• cd /var/www/ItemCatalog/ItemCatalog
• sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
• sudo pip install oauth2client requests httplib2

• sudo apt-get install postgresql
```
Now we create the database.
```
• sudo su - postgres
• psql
```

We are now connected as postgres. We then run:
```
CREATE USER catalog WITH PASSWORD 'grader';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog; 
```
To connect as catalog we run `\c catalog`
```
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
```

## Deploy application
We first need to update our database setup and application python files
In database_setup.py, category.py and application.py (use `sudo nano` to enter and edit these files) we change the line 
```
create_engine('sqlite:///sportcategorieswithusers.db') 
```
to 

```
engine = create_engine('postgresql://catalog:grader@localhost/catalog')
```

## Oauth login
```
CLIENT_ID = json.loads(
    open('/var/www/ItemCatalog/ItemCatalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/ItemCatalog/ItemCatalog/client_secrets.json', scope='')
```

Then we run:
```
• sudo python database_setup.py (to initialize the database)
• sudo python category.py (to populate the databse)
• sudo python application.py (to run the application)
```

Now going to the ip address in browser should give the working application.


## Login page
For the login page to work I used xip.io as Google doesn't recognise public ip addresses yet.
I added the urls under the restrictions within credentials in the google developers console.
Then I copied and pasted the new `client_secrets.json` (deleted the old content).
Now to successfully login you have to go to the url http://13.232.212.122.xip.io/login

## Notes on errors
You may get errors in the `application.py` file. 

Change line 147 from 
`user = session.query(User).filter_by(id=user_id).one()`
to 
`user = session.query(User).filter_by(id=user_id).first()`.

Also remove the trailing backslash on line 173 and bring the content on line 174 to be on line 173.

## Acknowledgements
I was stuck for 2 weeks after the firewall and port configuration, this was only solved by changing from my work to personal laptop as my work laptop had been blocking something with it's aws configurations.

I also gained a lot of help from fellow udacity peer Subhadeep Dey.

