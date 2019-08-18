## UDACITY Linux Server Configuration Project
# Student:  *Dennis Johns*

This is the final project for Udacity's **Full Stack Web Developer Nanodegree**. 

This page explains how to secure and set up a Linux distribution on a virtual machine, install and configure a web and database server to host a web application. 
- The **Ubuntu** distribution of Linux is used
- The virtual private server is Amazon Lighsail
- The web application is the Item Catalog project, which I created a baseball roster application
- The database server is PostgreSQL

You can see the deployment at http://52.88.238.171/ or http://52.88.238.171.xip.io

**Steps Followed**

* Start a new Ubuntu Linux server instance on Amazon Lightsail 

- Create an account at [Lightsail Services]https://lightsail.aws.amazon.com/ls/webapp/home/resources
- After logging in, choose `Create instance`. 
- Choose `Linux/Unix` platform, `OS Only` and  `Ubuntu`.
- Choose a instance plan of your choosing.
- Enter a name for your instance.  I choose *UdacityForever*
- Click the `Create` button to create the instance.
- Wait for the instance to start up.



* From your local vm, SSH into the server

- From the *Account* menu on Amazon Lightsail, click on *SSH keys* tab and download the Default Private Key.
- Move this private key file named LightsailDefaultPrivateKey-*.pem into the local folder *~/.ssh` and rename it `lightsail_key.rsa*.
- In your terminal, type: `chmod 600 ~/.ssh/lightsail_key.rsa`.
- To connect to the instance via the terminal: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@52-88-238-171`.


* Update and upgrade installed packages on the server

```
sudo apt-get update
sudo apt-get upgrade
```


* Change the SSH port from 22 to 2200

- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number from `22` to `2200`.
- Save and exit.
- Restart SSH: `sudo service ssh restart`.

* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

- Configure firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```

- Turn UFW on: `sudo ufw enable`. 

- Check the status of UFW to list current roles: `sudo ufw status`:

  ```
  Status: active
  
  To                         Action      From
  --                         ------      ----
  2200/tcp                   ALLOW       Anywhere                  
  80/tcp                     ALLOW       Anywhere                  
  123/udp                    ALLOW       Anywhere                  
  22                         DENY        Anywhere                  
  2200/tcp (v6)              ALLOW       Anywhere (v6)             
  80/tcp (v6)                ALLOW       Anywhere (v6)             
  123/udp (v6)               ALLOW       Anywhere (v6)             
  22 (v6)                    DENY        Anywhere (v6)
  ```

- Exit the SSH connection by entering exit.

- Click on the `Manage` option of the Amazon Lightsail Instance, 
then the `Networking` tab, and then change the firewall configuration to match the internal firewall settings above.
  
- From your local terminal, run: `ssh -i ~/.ssh/lightsail_key.rsa -p 2200 ubuntu@52.88.238171`


* Use `Fail2Ban` to ban attackers 


`Fail2Ban` is an intrusion prevention software framework that protects computer servers from brute-force attacks.
- `sudo apt-get install fail2ban`.
- `sudo apt-get install sendmail iptables-persistent`.
- Copy the .conf file to a .local file: `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.
- Change the settings in `/etc/fail2ban/jail.local` file:
  ```
  set bantime = 600
  destemail = useremail@domain
  action = %(action_mwl)s 
  ```
- Under `[sshd]` change `port = ssh` by `port = 2200`.
- Restart the service: `sudo service fail2ban restart`.



* Create a new user account named `grader`

- While logged in as `ubuntu`, add user: `sudo adduser grader`. 
- Enter a password and fill out information for this new user.


* Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `grader` has sudo permissions. Run `su - grader`, enter the password, 
run `sudo -l` and enter the password again. 

* Create an SSH key pair for `grader` using the `ssh-keygen` tool

- On the local machine:
  - Run `ssh-keygen`
  - Enter file in which to save the key (I gave the name `grader_key`) in the local directory `~/.ssh`
  - Enter in a passphrase twice. Two files will be generated (  `~/.ssh/grader_key` and `~/.ssh/grader_key.pub`)
  - Run `cat ~/.ssh/grader_key.pub` and copy the contents of the file
  - Log in to the grader's virtual machine
- On the grader's virtual machine:
  - Create a new directory called `~/.ssh` (`mkdir .ssh`)
  - Run `sudo nano ~/.ssh/authorized_keys` and paste the content into this file, save and exit
  - Give the permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Check in `/etc/ssh/sshd_config` file if `PasswordAuthentication` is set to `no`
  - Restart SSH: `sudo service ssh restart`
- On the local machine, run: `ssh -i ~/.ssh/graderkey -p 2200 grader@52.88.238171`.


* Configure the local timezone to UTC

- While logged in as `grader`, configure the time zone: `sudo dpkg-reconfigure tzdata`. You should see something like that:

  ```
  Current default time zone: 'US/Central'
  Local time is now:      Wed Aug 14 16:23:10 CDT 2019.
  Universal Time is now:  Wed Aug 14 21:23:10 UTC 2019.

  ```

* Install and configure Apache to serve a Python mod_wsgi application

- While logged in as `grader`, install Apache: `sudo apt-get install apache2`.
- Install the Python 3 mod_wsgi package:  
 `sudo apt-get install libapache2-mod-wsgi-py3`.
- Enable `mod_wsgi` using: `sudo a2enmod wsgi`.


* Install and configure PostgreSQL

- While logged in as `grader`, install PostgreSQL:
 `sudo apt-get install postgresql`.
- PostgreSQL should not allow remote connections. In the  `/etc/postgresql/9.5/main/pg_hba.conf` file, you should see:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```

- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal with `psql`.
- Create the `roster` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE roster WITH LOGIN PASSWORD 'roster';
  postgres=# ALTER ROLE roster CREATEDB;
  ```

- List the existing roles: `\du`. 
  ```
                                   List of roles
  Role name |                         Attributes                         | Member of
 -----------+------------------------------------------------------------+-----------
  postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
  roster    | Create DB                                                  | {}
 

  ```

- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.
- Create a new Linux user called `roster`: `sudo adduser roster`. Enter password and fill out information.
- Give `roster` the permission to sudo. Run: `sudo visudo`.
- Search for the lines that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `roster` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  roster  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.

- Type exit to return to Ubuntu.  
- Create a postgres database: `createdb roster`.
- Run `psql` and then run `\l` to see that the new database has been created. 
  ```
- Exit psql: `\q`.


* Install git

- Install `git`: `sudo apt-get install git`.


* Clone and setup my Baseball Roster project from the GitHub repository titled Baseball Roster App

- Create `/var/www/roster/` directory.
- Change to that directory and clone the baseball roster app project:<br>
`sudo git clone https://github.com/HoosierCoder/Baseball-Roster-App.git roster`.
- From the `/var/www` directory, change the ownership of the `roster` directory to `grader` using: `sudo chown -R grader:grader roster/`.
- Change to the `/var/www/roster/roster` directory.
- Rename the `application.py` file to `__init__.py` using: `mv application.py __init__.py`.

- In `__init__.py`, replace:
  ```
  # engine = create_engine("sqlite:///teamwithusers.db")
  engine = create_engine('postgresql://roster:password@localhost/roster')
   
  # app.run(host="0.0.0.0", port=8000, debug=True)
  app.run()
  ```

- In `database_setup.py` and 'teamrosters.py', replace:
   ```
   # engine = create_engine("sqlite:///teamwithusers.db")
   engine = create_engine('postgresql://roster:password@localhost/roster')
   ``` 

* Install the virtual environment and dependencies

- While logged in as `grader`, install pip: `sudo apt-get install python3-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Change to the `/var/www/roster/roster/` directory.
- Create the virtual environment: `sudo virtualenv -p python3 venv3`.
- Change the ownership to `grader` with: `sudo chown -R grader:grader venv3/`.
- Activate the new environment: `. venv3/bin/activate`.
- Install the following dependencies:
  ```
  pip install httplib2
  pip install requests
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```


* Set up and enable a virtual host

- Create `/etc/apache2/sites-available/roster.conf` and add the 
following lines to configure the virtual host:

  ```
  <VirtualHost *:80>
                  ServerName 52.88.238.171
                  ServerAlias 52.88.238.171.xip.io
                  WSGIScriptAlias / /var/www/roster/roster.wsgi
                  <Directory /var/www/roster/roster/>
                          Order allow,deny
                          Allow from all
                  </Directory>
                  Alias /static /var/www/roster/roster/static
                  <Directory /var/www/roster/roster/static/>
                          Order allow,deny
                          Allow from all
                  </Directory>
                  ErrorLog ${APACHE_LOG_DIR}/error.log
                  LogLevel warn
                  CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>

  ```

- Enable virtual host: `sudo a2ensite roster`. The following prompt will be returned:
  ```
  Enabling site roster.
  To activate the new configuration, you need to run:
    service apache2 reload
  ```

- Reload Apache: `sudo service apache2 reload`.

* Set up the Flask application

- Create `/var/www/roster/roster.wsgi` file add the following lines:

  ```
  activate_this = '/var/www/roster/roster/venv3/bin/activate_this.py'
  with open(activate_this) as file_:
      exec(file_.read(), dict(__file__=activate_this))

  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0,"/var/www/roster")
  
  from roster import app as application
  application.secret_key = 'super_secret_key'

- Restart Apache: `sudo service apache2 restart`.


* Set up the database schema and populate the database
	- cd /var/www/roster/roster
	- execute sudo python3 database_setup.py
	- execute sudo python3 teamrosters.py

* Disable the default Apache site

- Disable the default Apache site: `sudo a2dissite 000-default.conf`. 
The following prompt will be returned:

  ```
  Site 000-default disabled.
  To activate the new configuration, you need to run:
    service apache2 reload
  ```

- Reload Apache: `sudo service apache2 reload`.

* Launch the Web Application

- Open your browser to http://52.88.238.171 or http://52.88.238.171.xip.io.


*References
'''
	[Flask Deployment Documentation](https://flask.palletsprojects.com/en/1.1.x/deploying/)
	[Flask with Python3](https://linoxide.com/linux-how-to/install-flask-python-ubuntu/)
	[AWS Documentation](https://linoxide.com/linux-how-to/install-flask-python-ubuntu/)
	[Apache Server Documentation](https://httpd.apache.org/docs/)
	[Installing GIT](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
	[SQLAlchemy for Python3 Documentation and Overview](https://docs.sqlalchemy.org/en/13/intro.html)
	[PostgreSQL Installation and tips](https://wiki.postgresql.org/wiki/Detailed_installation_guides)
'''
