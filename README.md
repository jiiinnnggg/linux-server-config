# linux-server-config

The goal of this project is to ultimately deploy the Item Catalog application developed earlier in the FSND course to a linux server. We'll confiugre a ubuntu OS-only instance on AWS (Amazon Lightsail), install and update required packages, set up SSH key login for a new user, configure an Uncomplicated Firewall (UFW), configure a PostgresQL database server, configure Apache/WSGI to serve the Flask application, and modify the application's OAuth for user logins.

The deployed application is available (until about early Sep 2018) at: http://35.174.159.105.xip.io/

## User Setup

After logging into ubuntu (via Terminal, using SSH - more instructions at: https://lightsail.aws.amazon.com/ls/webapp/account/keys), first run the base ubuntu updates & upgrades:
```sh 
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt update
$ sudo apt upgrade
```
Then, create a user ```grader```:
```sh 
$ sudo adduser grader
```
And for the UNIX password prompt, we'll use ```grader``` as the password as well.
To add the ```grader``` user to sudo priviledges, run:
```sh 
$ sudo visudo
```
which will open nano, and in the text file, find the entry 
```sh
# User privilege specification
root    ALL=(ALL:ALL) ALL
```
and copy/insert below that row: ```grader    ALL=(ALL:ALL) ALL```.
Try the command:
```sh
$ sudo login grader
```
with password: ```grader``` to confirm.
After you're logged in as ```grader```, enter the commands:
```sh
$ sudo mkdir /home/grader/.ssh
$ cd .ssh
$ sudo nano authorized_keys
```
Leave this terminal window open and now let's set up SSH-keygen for the user ```grader``` to log via SSH from another terminal window.
Open a new window and run:
```sh
$ ssh-keygen
```
Create a file name for the key, and two files will be generated: ```[yourfilename]``` and ```[yourfilename].pub```.  From that same terminal, let's copy and paste the content of the public key into the ```authorized keys``` file still open on the ubuntu terminal. You can find the contents of the public key by running the command:
```sh
$ cat /Users/[yourlocalusername]/.ssh/[yourfilename].pub
```
Save the ```authorized_keys``` file and from a separate local terminal, try logging in via SSH by running:
```sh
$ ssh grader@[your.ip.address.here] -i ~/.ssh/[yourfilename] -p 22
```
Now, logged in as ```grader``` via SSH, run:
```sh
$ sudo nano /etc/ssh/sshd_config
```
and under ```# What ports, IPs and protocols we listen for```, change ```Port 22``` to ```Port 2200```. Check that under ```# Authentication:```, it reads ```PermitRootLogin no```. Also check that under ```# Change to no to disable tunnelled clear text passwords```, it reads ```PasswordAuthentication no```.

For the UFW firewall, run the follow commands to allow port 2200(tcp), 80(tcp), 123(udp) and deny 22(tcp):
```sh
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw deny 22/tcp
```
To enable the firewall run:
```sh
$ sudo ufw enable
```
To check the status, run:
```sh
$ sudo ufw status
```

Check if the date is configured to UTC by running:
```sh
$ date
```
If the timezone is not UTC, configure timezone to UTC, run the command:
```sh
$ sudo dpkg-reconfigure tzdata
```
Select ```'None of the above'```, then select ```'UTC'```.

## Install and Configure Apache and PostgresQL
First, install Apache2, run:
```sh
sudo apt-get install apache2
```
Install the libapache2-mod-wsgi package and python, and enable wsgi:
```sh
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```
Restart Apache:
```sh
sudo service apache2 restart
```

## Git, dependencies, and clone repository:
Run:
```sh
$ sudo apt-get install git
```
Install packages + dependencies:
```sh
$ sudo apt-get install python-flask python-sqlalchemy
$ sudo apt-get install python-psycopg2 python-pip
$ sudo pip install --upgrade pip
$ sudo pip install oauth2client requests httplib2
```
Follow the instructions here: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps to set up a virtualenv and deploy the Flask Item Catalog application. You'll clone your repository into ```/var/www/[FlaskApp]/```. The Flask app will need to be configured with a .wsgi file in the ```/var/www/[FlaskApp]/``` folder, as well as a .conf file in ```/etc/apache2/sites-available/```. You'll need to rename your python file (e.g. ```catalog.py```) to ```__init__.py``` (i.e. the path will be ```/var/www/[FlaskApp]/[FlaskApp]/__init__.py```) in order for the Flask app to run.

Disable the default Apache .conf file as well:
```sh
$ sudo a2dissite 000-default
```

For modifications/changes to the repository cloned to the ubuntu server, it's convenient to create a new branch (e.g. ```ubuntu```) to store these changes while testing the ```master``` branch locally on the vagrant machine for debugging purposes.

In your ```___init__.py``` file, modify the section containing ```app.run()``` to:
```sh
if __name__ == '__main__':
    app.run()
```
since you're no longer running on localhost 5000.

In addition, you'll need to recreate the ```[your_client_secrets].json``` file(s) in the folder since these are not stored on the git repository. You'll also need to modify any ```.py``` files that refer to these ```.json``` files to reflect the updated file path on the ubuntu server.

## Configure PostgresQL

Install by running:
```sh
$ sudo apt-get install postgresql
$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf
```
Check that the only allowed connections are from the localhost ```127.0.0.1``` for IPv4 and ```::1``` for IPv6.

Create a password as the ```postgres``` user:
```sh
$ sudo passwd postgres
```
(pw locally is ```postgres```)
Log in as the user ```postgres```:
```sh
$ su postgres
$ psql
```
Execute the following commands after opening ```psql``` to create a database ```catalog``` with password ```catalog```:
```sh
# CREATE USER catalog;
# ALTER USER catalog WITH PASSWORD 'catalog';
# CREATE DATABASE catalog WITH OWNER catalog;
# \c catalog
# REVOKE ALL ON SCHEMA public FROM public;
# GRANT ALL ON SCHEMA public TO catalog;
# \q
```
Logout of the ```postgres``` user with:
```sh
$ exit
```
Restart postgresql:
```sh
$ sudo service postgresql restart
```
In the python files where the sqlite reference is located for the localhost version, replace with `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`, ie. in the `database_setup.py` and `__init__.py` files. (Note: since there is a table already named `user` containing the user `postgres`, I refactored my database `.py` file to rename that table as `users` and changed any associated code throughout the application.)

In the repo folder, run:
```sh
$ sudo python database_setup.py
$ sudo load_db_samples.py
```
to set up the database. To check if the sample data has been probably loaded into the right tables, you can go into `psql` and run the `SELECT * from [tablename]` queries in database `catalog`.

## Google OAuth

Assuming the `.json` file(s) containing the client secrets have been properly added and file paths correctly referenced in the application files, we then need to add the URI for our application onto the Google Developers Console -> API Manager -> Credentials page under 'Authorized JavaScript origins' and 'Authorized redirect URIs'. For this, we'll use the `xip.io` appendix to our IP address, ie. http://35.174.159.105.xip.io/.

Reload Apache2 by restarting:
```sh
$ sudo service apache2 restart
```

Note: if your browser settings do not allow third party cookies to set, the log in might hang.

Go to http://35.174.159.105.xip.io/ to test out the application served on your ubuntu instance.


### References:

https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config and https://github.com/AbigailMathews/FSND-P5 were helpful for postgres setup.






