# Linux Server
Linux Server deploying catalog app
IP: 165.227.15.7

Server is running Ubuntu with Apache2 with WSGI

## Steps to configure server:

### Step 1 (Connect to server):
Connect to new ubuntu server using `ssh root@165.227.15.7`

### Step 2 (Create new sudo user)
On **server** add new user using `adduser grader`

Add **user** to the *sudo* or *admin* group to give the **user** sudo priveleges  
`usermod grader -aG (admin/sudo)`

Change to the *grader* user  
`su grader`

Create *authorized_keys* file for grader  
`mkdir .ssh/`  
`touch .ssh/authorized_keys`

### Step 3 (Create key):
On **client machine** open another command prompt/terminal and change into the current users directory using `cd ~/`

If user already has a ssh directory '.ssh' enter it using `cd .ssh/`  
Otherwise make a ssh directory using `mkdir .ssh` then change into it with `cd .ssh/`

Once in the .ssh directory create a new **keypair**  
`ssh-keygen`  
Enter where to create it  
`/c/Users/USER/.ssh/id_rsa`  
Enter **passphrase** if desired  
`SSHPASSPHRASE`

Copy the ssh **public key** to the server  
`cat ~/.ssh/id_rsa.pub | ssh grader@165.227.15.7 "cat >> ~/.ssh/authorized_keys"`

On the **server** change the *file permissions* using  
`sudo chmod 600 ~/.ssh/authorized_keys`  
`sudo chmod 700 ~/.ssh/`

**From now on you must connect to server using your public key** `ssh grader@165.227.15.7 -i ~/.ssh/id_rsa`

### Step 4 (Disable root user & change ssh port)
Enter the *ssh config* file  
`sudo vi /etc/ssh/sshd_config`

Find the line that says `Port 22` and change it to `Port 2200`
Find the line that says `PermitRootLogin yes` and change the *value* to 'no'  
**Insert** line `DenyUsers root`  
**Insert** line `AllowUsers grader` so you don't lock yourself out of the server  
**Uncomment** the line `AuthorizedKeysFile %h/.ssh/authorized_keys`
Save and close the file

Reload ssh  
`sudo service ssh restart`

**From now on you must connect to server using your public key** `ssh grader@165.227.15.7 -i ~/.ssh/id_rsa -p 2200`

### Step 5 [Setup Uncomplicated Firewall (UFW)]
Set up *uncomplicated firewall* to only allow **SSH** (Port 2200), **HTTP** (Port 80), and **NTP** (Port 123)

    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    sudo ufw enable
    
### Step 6 (Configure timezone)
Configure the **timezone** to *server location's* **timezone** using  
`sudo dpkg-reconfigure tzdata`

### Step 7 (Upgrade apps)
Upgrade all apps using

    sudo apt-get update
    sudo apt-get upgrade
    
### Step 8 (Install and setup database)
Install **PostgreSQL** `sudo apt-get install postgresql`  
Check if remote connections are allowed `sudo vim /etc/postgresql/9.3/main/pg_hba.conf`  

Login as **user** 'postgres' `sudo su postgres`  
Get into **postgreSQL *shell*** psql

Create a new **database** named *catalog* and create a new **user** named *catalog* in **postgreSQL *shell***

    postgres=# CREATE DATABASE catalog;
    postgres=# CREATE USER catalog;
    
Set a *password* for **user** *catalog*  
`postgres=# ALTER ROLE catalog WITH PASSWORD 'password';`

Give **user** *catalog* permission to *catalog* *application database*  
`postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`

Quit **postgreSQL** `postgres=# \q`  
Exit from **user** *postgres* `exit`

### Step 9 (Install and configure apache for python)
**Install** *Apache* `sudo apt-get install apache2`  
**Install** *mod_wsgi* `sudo apt-get install python-setuptools libapache2-mod-wsgi` 

**Restart** *Apache* `sudo service apache2 restart`

### Step 10 (Clone git repository of project to deploy)
**Install** *Git* using `sudo apt-get install git`

Move to the `/var/www directory` using `cd /var/www/`  
**Create** the *application directory* `sudo mkdir FlaskApp`  
Move change into new directory `cd FlaskApp`

**Clone** the *Catalog App* to the virtual machine `sudo git clone https://github.com/wsscottca/catalog_app.git`  
Make inner FlaskApp directory `sudo mkdir FlaskApp`  
**Extract** project to FlaskApp `sudo mv ./catalog_app/catalog/* ./FlaskApp`

Move to inner FlaskApp directory `cd FlaskApp`  
**Edit** database_setup.py, \_\_init\_\_.py and starter_content.py and change `engine = create_engine('sqlite:///catalog.db')` to `engine = create_engine('postgresql://catalog:password@localhost/catalog')`

**Update** all file references in all python files using

    with app.open_resource('client_secrets.json') as f:    
        CLIENT_ID = json.load(f)['web']['client_id']

**Install** *pip* `sudo apt-get install python-pip`  
Use *pip* to **install dependencies** 

    sudo pip install flask
    sudo pip install sqlalchemy
    sudo pip install oauth
    sudo pip install oauth2client
    sudo pip install requests
    
    
**Install** *psycopg2* `sudo apt-get -qqy install postgresql python-psycopg2`  
**Create** database *schema* `sudo python database_setup.py`
**Populate** database `sudo puthon starter_content.py`

### Step 11 (Configure and Enable a New Virtual Host)
**Create** FlaskApp.conf to edit:  
`sudo touch /etc/apache2/sites-available/FlaskApp.conf`  
`sudo vi /etc/apache2/sites-available/FlaskApp.conf`

**Add** the following lines of code to the file to *configure* the virtual host.

    <VirtualHost *:80>
	    ServerName 165.227.15.7
	    ServerAdmin wscott@managednetwork.com
	    WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
	    <Directory /var/www/FlaskApp/FlaskApp/>
		    Order allow,deny
		    Allow from all
	    </Directory>
	    Alias /static /var/www/FlaskApp/FlaskApp/static
	    <Directory /var/www/FlaskApp/FlaskApp/static/>
		    Order allow,deny
		    Allow from all
	    </Directory>
	    ErrorLog ${APACHE_LOG_DIR}/error.log
	    LogLevel warn
	    CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    
**Enable** the virtual host with the following command: `sudo a2ensite FlaskApp`

### Step 12 (Create the .wsgi File)
**Create** the *.wsgi* File under '/var/www/FlaskApp':

    cd /var/www/FlaskApp
    sudo vi flaskapp.wsgi
    
**Add** the following lines of code to the *flaskapp.wsgi* file:

    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/FlaskApp/")

    from FlaskApp import app as application
        application.secret_key = 'Add your secret key'
    
### Step 13 (Restart Apache)
**Restart** *apache* `sudo service apache2 restart`
