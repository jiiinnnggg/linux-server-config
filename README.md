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




