# Linux-Project
Setting up an AWS lightsail server to run a flask app

* Provide details for the reviewer:

    * public Ip: `http://34.253.50.16/`
    * SSH PORT: `2200`
    * Full project URL: [Catalog Site](http://ec2-34-253-50-16.eu-west-1.compute.amazonaws.com)

Configuration:

* Start a new Ubuntu server instance on Amazon Lightsail:

    * Get started: https://amazonlightsail.com/
    * Set it to OS Only, Ubuntu.

* Once finished, you will have an instance.

    * You will get your respective public IP address.
    * On your local machine (Your pc) make a directory .ssh
    * Download the default key-pair for the instance from the website and copy to the /.ssh folder.
    * Open your terminal and type in chmod 600 ~/.ssh/key.pem (You can rename it to anything)
    * Now Use the command `ssh -i ~/.ssh/key.pem ubuntu@34.201.114.178` to log in from the local machine terminal

* Creating a new user named grader

    * Now that you are logged in as ubuntu, do the following.
    * `sudo adduser grader`
    *  You can optionally install finger to check a user has been added `apt-get install finger`
    * `finger grader`

* Give the grader sudo permission

    * `sudo visudo`
    * inside the file add `grader   ALL=(ALL:ALL) ALL` below the root user under "#User privilege specification", save it.
    * Add grader to `/etc/sudoers.d/` and type in `grader   ALL=(ALL:ALL) ALL`by command `sudo nano /etc/sudoers.d/grader`
    * Add root to `/etc/sudoers.d/` and type in `root   ALL=(ALL:ALL) ALL`by command `sudo nano /etc/sudoers.d/root`

* Update installed packages

    * Find updates:`sudo apt-get update`
    * Install updates:`sudo sudo apt-get upgrade` 

* Change the SSH port from 22 to 2200 

    * `nano /etc/ssh/sshd_config` add `port 2200` below `port 22`
    * In the file also change `PermitRootLogin prohibit-password` to `PermitRootLogin no` to disable root login
    * Change `PasswordAuthentication` from `no` to `yes`. We will change back after finishing SHH login setup
    * save your settings.
    * restart the ssh service using `sudo service ssh reload`

* Create SSH keys and use with your instance:

    * On your local machine generate with: `ssh-keygen`
    * save your keygen file in the ssh directory: `/home/ubuntu/.ssh/` example full file path that could be used: `/home/ubuntu/.ssh/item-catalog`
    * No need to add a passphrase for the generated file.
    * In Amazon lightsail, find the networking settings and add a custom rule with port 2200.
    * login into grader account using password set during user creation `ssh -v grader@*Public-IP* -p 2200`
    * On succesful login, make .ssh directory `mkdir .ssh`
    * make file to store your authorized keys `touch .ssh/authorized_keys`
    * On your local machine read contents of the key from earlier, it should end with .pub `cat .ssh/item-catalog.pub`
    * Within the instance, in grader, paste the key into authorized_keys file `nano .ssh/authorized_keys`
    * save the file.
    * Set permissions as follow: `chmod 700 .ssh` `chmod 644 .ssh/authorized_keys`
    * Change `PasswordAuthentication` from `yes` back to `no`.  `nano /etc/ssh/sshd_config`
    * save the file.
    * From your local machine you can now log in via key pair: `ssh grader@*Public-IP* -p 2200 -i ~/.ssh/item-catalog`

* Configure the Uncomplicated Firewall (UFW) to only allow  incoming connections for SSH (port 2200), HTTP (port 80),  and NTP (port 123)

    * Check the firewall UFW status to make sure its inactive `sudo ufw status`
    * Deny all incoming by default `sudo ufw default deny incoming`
    * Allow outgoing by default` sudo ufw default allow outgoing`
    * Allow SSH `sudo ufw allow ssh`
    * Allow SSH on port 2200 `sudo ufw allow 2200/tcp`
    * Allow HTTP on port 80 `sudo ufw allow 80/tcp`
    * Allow NTP on port 123 `sudo ufw allow 123/udp`
    * Turn on firewall` sudo ufw enable`

* Configure the local timezone to UTC (Universal Time)

    * run `sudo dpkg-reconfigure tzdata` from prompt: select none of the above. Then select UTC.

* Install and configure Apache to serve a Python mod_wsgi application

    * `sudo apt-get install apache2` 
    * Go to your public IP address on a browser, it should show an Apache page with "It works!"
    * Install mod_wsgi `sudo apt-get install libapache2-mod-wsgi`
    * Configure Apache to handle requests using the WSGI module `sudo nano /etc/apache2/sites-enabled/000-default.conf`
    * Add `WSGIScriptAlias / /var/www/html/myapp.wsgi` before `</VirtualHost>` closing line
    * save the file and restart Apache `sudo apache2ctl restart`

* Install git, we will clone your catalog project at a later step

    * Install git `sudo apt-get install git`

* install the python-dev package and verify WSGI is enabled

    * Install python-dev package`sudo apt-get install python-dev`
    * Verify wsgi is enabled `sudo a2enmod wsgi`
    
* Create a flask app

    * `cd /var/www`
    * `sudo mkdir catalog`
    * `cd catalog`
    * `sudo mkdir catalog`
    * `cd catalog`
    * `sudo mkdir static templates`
    * `sudo nano __init__.py `
    * Copy the contents below into __init__.py and save the file, for now.

    ```
     from flask import Flask
    app = Flask(__name__)
    @app.route("/")
    def hello():
        return "Hello, world (Testing!)"
    if __name__ == "__main__":
    app.run()
    ```

* install flask

    * `sudo apt-get install python-pip`
    * `sudo pip install virtualenv `
    * `sudo virtualenv venv`
    * `sudo chmod -R 777 venv`
    * `source venv/bin/activate`
    * `pip install Flask`
    * `python __init__.py`
    * `deactivate`

* Configure And Enable New Virtual Host

    * Create host config file `sudo nano /etc/apache2/sites-available/catalog.conf`
    * paste the following:

    ```
    <VirtualHost *:80>
      ServerName 34.201.114.178
      ServerAdmin admin@34.201.114.178
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
    * Save the file.
    * Enable `sudo a2ensite catalog`

* Create the wsgi file

    * `cd /var/www/catalog`
    * `sudo nano catalog.wsgi`
    * Copy the contents below into the catalog.wsgi file, for now.

    ```
     #!/usr/bin/python
     import sys
     import logging
     logging.basicConfig(stream=sys.stderr)
     sys.path.insert(0,"/var/www/catalog/")

     from catalog import app as application
     application.secret_key = 'Add your secret key'
    ```

    * Save the file
    * `sudo service apache2 restart`

* Clone Github Repo

    * Go to your github and copy the url address of the catalog project
    * `sudo git clone yoururladdresshere`
    * Hide your files `shopt -s dotglob`. 
    * Move files from the clone directory to catalog `mv /var/www/catalog/item-catalog/* /var/www/catalog/catalog/`
    * remove clone directory `sudo rm -r item-catalog`

* make .git inaccessible

    * from `cd /var/www/catalog/` create .htaccess file `sudo nano .htaccess`
    * paste in `RedirectMatch 404 /\.git`
    * Save the file

* install project dependencies:

    * `source venv/bin/activate`
    * `pip install httplib2`
    * `pip install requests`
    * `sudo pip install --upgrade oauth2client`
    * `sudo pip install sqlalchemy`
    * `pip install Flask-SQLAlchemy`
    * `sudo pip install python-psycopg2`
    * `pip install requests`
    * If you used any other packages, install them too. You can usually tell from your imports.

* Install and configure PostgreSQL:

    * Install postgres `sudo apt-get install postgresql`
    * install additional models `sudo apt-get install postgresql-contrib`
    * config database_setup.py `sudo nano database_setup.py`
    * Replace your SQLite line with `engine = create_engine('postgresql://catalog:db-password@localhost/catalog')`
    * repeat for project.py and any files that connect to your database i.e. data.py (if you insert data).
    * copy your main project.py content into the __init__.py file, this will now be your main file.
    * Add catalog user `sudo adduser catalog`
    * login as postgres super user `sudo su - postgres`
    * enter postgres `psql`
    * Create user catalog `CREATE USER catalog WITH PASSWORD 'db-password';`
    * Change role of user catalog to creatDB ` ALTER USER catalog CREATEDB;`
    * List all users and roles to verify `\du`
    * Create new DB "catalog" with own of catalog `CREATE DATABASE catalog WITH OWNER catalog;`
    * Connect to database `\c catalog`
    * Revoke all rights `REVOKE ALL ON SCHEMA public FROM public;`
    * Give access to only catalog role `GRANT ALL ON SCHEMA public TO catalog;`
    * Quit postgres `\q`
    * logout from postgres super user `exit`
    * Setup your database schema `python database_setup.py`

* Setting up Facebook OAuth to work with the server:

    * If you used or also used google OAuth, the best step is to browse the relevant forums.
    * Go to https://developers.facebook.com/
    * At the top myapps, select the catalog app you made for your project.
    * On the sidebar, go to facebook login.
    * If not enabled, enable Client OAUTH Login, Web OAUTH Login, Embedded Browser OAUTH Login.
    * Insert into OAuth valid redirect Url's: http://yourpublicip/_oauth/facebook, http://yourpublicip/login,     http://yourpublicip/

* Setting up the server to work with Facebook OAuth:

    * Your __init__.py file, will need a full path to the fb_client_secrets.json like so `/var/www/catalog/catalog/fb_client_secrets.json`
    * Do your final restart to see changes take effect `sudo service apache2 restart`

* References

https://discussions.udacity.com/t/connection-error-linux/241400/17
https://discussions.udacity.com/t/cant-access-flask-application-in-the-browser/39931/9
