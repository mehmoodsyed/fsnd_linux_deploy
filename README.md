LINUX SERVER DEPLOYMENT (FSND):
===============================
INTRODUCTION:

In this project we are required to deploy the application that we built previously (https://github.com/mehmoodsyed/fsnd_ab_catalog.git) on a Linux Server.

DESCRIPTION:

The suggested platform for this project was Amazon Lightsail and that's where I started. I ditched Amazon Lightsail for Microsoft Azure
after few days of frustration. Setting up the additional user and configuring a non-standard SSH port took more time than I anticipated
on Amazon Lightsail. However deploying on Azure proved to be much better experience. In all it took me roughly 6-8 hours after selecting my Azure instance to get my app operational. For size of my VM I selected Standard B1ls (1 vcpus, 0.5 GiB memory). For region I selected East US and for image I selected Ubuntu Server 18.04 LTS. The cost quoted at the time of creation was less than $4. The IP address of my instance is 40.87.93.145. I selected user name ubuntu as my admin user. For DNS name I selected msyedhobbyserver. So my application can be accessed at the following URL:
http://msyedhobbyserver.eastus.cloudapp.azure.com/


SETUP, MODIFICATIONS and EXECUTION:

After creating the instance, the next order of business was to create key pair to connect with my instance via SSH client. I used PuTTY Key Generator to generate the public and the private keys. I needed to paste public key in my Azure admin portal so that my admin user ubuntu was ready when I connected with SSH client. For my SSH client I selected PuTTY. Connecting with my instance via SSH client using the private key generated in the previous step was successful.
Issuing command


    sudo apt update


resulted in following output:


    All packages are up to date.


Issuing the date command on the terminal displayed that the server was configured for the UTC time zone. Nevertheles I issued following command to go through the exercise of setting up the time zone:


    sudo dpkg-reconfigure tzdata


The next step was to setup the user grader as required. I installed finger using the following command:


    sudo apt install finger


The user grader was created using following command:


    sudo adduser grader


To grant the sudo rights to user grader I issued following command: 


    usermod -aG sudo grader


I followed it by creating file grader in folder /etc/sudoers.d with the following contents:


    grader ALL=(ALL) NOPASSWD:ALL


In order to login with user grader via my SSH client I needed to copy public key from my ubuntu account to the .ssh directory of the user grader. I issued following commands to accomplish this task:


    sudo mkdir /home/grader/.ssh
    sudo chmod 700 /home/grader/.ssh
    sudo chgrp grader /home/grader/.ssh
    sudo cp ./.ssh/authorized_keys /home/grader/.ssh
    sudo chown grader /home/grader/.ssh/authorized_keys
    sudo chgrp grader /home/grader/.ssh/authorized_keys


Now the user grader is setup properly and I could connect using my ssh client and I was able to access higher privilige resources using sudo command.

Next step was configuring the firewall according to the following rules:
Only allow connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

Issuing following command:


    sudo ufw status


indicated that firewall is inactive.

I made changes to /etc/ssh/sshd_config by replacing port 2200 for port 22.

Following commands were issued to configure the firewall for the above rules and enabling it:


    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    sudo ufw enable


Firewall is configured for the requisite rules but I still had to disable port 22 from the Azure Portal and configuring port 2200 for SSH. At this point I also enabled port 80 and port 123 on my Azure Portal. Changing my SSH client to use port 2200 confirmed that the change was successful. 

Looking at the file /etc/ssh/sshd_config confirmed that the root login was disabled by default for my instance. However I had to add following line to enforce the requirement that only SSH authentication is allowed.


    PasswordAuthentication no


For changes made to /etc/ssh/sshd_config to take effect I restarted my VM from the Azure Portal.


It is now time to start deploying the application. First I needed to install Apache on my server. Following were the steps I took to install Apache:


    sudo apt install apache2
    sudo ufw allow 'Apache'
    sudo apt-get install libapache2-mod-wsgi-py3


After above steps, pointing my browser to the URL of my server displayed "Apache2 Ubuntu Default Page".

After this I added Apache configuration file audiobook_catalog.conf in the folder /etc/apache2/sites-available with the following contents to allow my app to be served by Apache:


    <VirtualHost *:80>
        ServerName msyedhobbyserver.eastus.cloudapp.azure.com
        ServerAdmin mehmoodsyed@gmail.com
        WSGIDaemonProcess application user=ubuntu threads=3
        WSGIScriptAlias / /var/www/audiobook_catalog/wsgi.py
        <Directory /var/www/audiobook_catalog/>
            WSGIProcessGroup application
            WSGIApplicationGroup %{GLOBAL}
            Require all granted
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>

After the above steps, I started to install components that are needed by my application by issuing following commands:


    sudo apt install python3-pip
    sudo pip3 install flask
    sudo pip3 install oauth2client
    sudo apt install libpq-dev
    sudo apt install postgresql-common
    sudo pip3 install psycopg2
    sudo pip3 install requests
    sudo pip3 install sqlalchemy


 I imported my code from the private repository on the GitHub, and placed it into /var/www/audiobook_catalog directory. Even though we were encouraged to use PostgreSQL for this project in place of SQLite, I decided to stay with SQLite because of the time constraints. I created and filled the database by using following commands:


    python3 database_setup.py
    python3 database_fill.py


Since my application will no longer be hosted on local host, I need to make some modifications to my client_secrets.json file. I logged onto Google API Dashboard @  https://console.developers.google.com and modified my Catalog Web App OAuth 2.0 Client ID by adding http://msyedhobbyserver.eastus.cloudapp.azure.com to "Authorized JavaScript origins". I also added http://msyedhobbyserver.eastus.cloudapp.azure.com/oauth2callback to "Authorized redirect URIs". I downloaded my new client_secrets.json file to my local machine and then updated the JSON file on the server with this one.

I also made following changes to the project_catalog.py file to update the path for the database and the client_secrets.json files. I also commented out the last two lines of the code as shown below since this application will be served by Apache.

    engine = create_engine('sqlite:////var/www/audiobook_catalog/audiobookcatalog.db?check_same_thread=False')
    app.config['GOOGLE_OAUTH2_CLIENT_SECRETS_FILE'] = '/var/www/audiobook_catalog/client_secrets.json'

    #if __name__ == '__main__':
    #    app.run(host='0.0.0.0')


Next I created wsgi.py file in /var/www/audiobook_catalog/ with following contents:


    # Add path to application directory
    import sys
    sys.path.insert(0, "/var/www/audiobook_catalog")

    # Import Flask instance from main application file
    from project_catalog import app as application

    if __name__ == '__main__':
        app.config['SESSION_TYPE'] = 'filesystem'
        app.run(host='0.0.0.0', port=80)


My application is now ready to go live. I execute following two commands:


    sudo a2ensite audiobook_catalog
    sudo service apache2 restart


From this point on the application will be available @ http://msyedhobbyserver.eastus.cloudapp.azure.com


  RESOURCES USED:
  1) Instructors notes
  2) http://www.stackoverflow.com
  3) https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-18-04-quickstart
  4) http://httpd.apache.org/docs/current/configuring.html
  5) https://askubuntu.com
  6) https://docs.sqlalchemy.org/en/13/core/engines.html
  7) https://br3ndonland.github.io/udacity/
  8) http://initd.org/psycopg/docs/install.html
  
  and probably many more

  ISSUES:
  
