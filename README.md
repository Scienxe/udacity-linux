# Udacity Full Stack: Linux Server Config

## User Setup
1. Create new development environment through provided Udacity page
1. ssh into new dev environment as root
2. Create new user "grader"

		adduser grader

3. in /etc/sudoers.d add a file called "grader" (the name is not important) containing the line:

		grader ALL=(ALL) NOPASSWD:ALL
	
4. The instance is set to disallow password logins, and grader does not yet have a public key on the server, so the grader cannot yet log in. One way to proceed would be to temporarily allow password logins in /etc/ssh/sshd_config by changing "PasswordAuthentication no" to "PasswordAuthentication yes". I chose to create the public key as root and then change the owner and group of the key. First, on the local machine, run

		ssh-keygen

5. Logged in as root on the server, run

		mkdir /home/grader/.ssh
		chown grader /home/grader/.ssh
		chgrp grader /home/grader/.ssh

6. Next create the file /home/grader/.ssh/authorized_keys containing the contents of udacityGrader.pub. Set ownership and permissions on the new file and folder:

		chown grader /home/grader/.ssh/authorized_keys
		chgrp grader /home/grader/.ssh/authorized_keys
		chmod 644 /home/grader/.ssh/authorized_keys
		chmod 700 /home/grader/.ssh

7. Disallow remote root login and set ssh to a non-default port: change two lines in /etc/ssh/sshd_config to read

		Port 2200
		PermitRootLogin no
	
8. Then run

		sudo service ssh restart

9. Log out of root and ssh in as grader, using the new port 2200.

### All following commands are run as grader
The default instance is configured with a hostname which does not match the actual public URL. This will cause the error "sudo: unable to resolve host ip-xx-xx-xx-xxx" to be printed every time you run a command. Configuring the server with the correct hostname will correct this problem. Set the server's hostname by editing the file /etc/hostname and replacing the current name with the server's public URL, in this case "ec2-54-68-165-124.us-west-2.compute.amazonaws.com".

## Update installed packages

	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get autoremove
	
## Set timezone and set up time synchronization

		sudo dpkg-reconfigure tzdata

Select "None of the Above" and then "UTC".
Next, install and configure ntp for network time synchronization. Run

		sudo apt-get install ntp
	
The default ntp servers are sufficient, otherwise we could edit /etc/ntp.conf to add or remove servers. The ntp daemon is started during the installation process. If changes are made to /etc/ntp.conf, reload ntpd by running

		sudo service ntp reload

## Configure ufw firewall
Lock down all ports except those we'll actually use by running

		sudo ufw default deny incoming
		sudo ufw default allow outgoing
		sudo ufw allow 2200/tcp
		sudo ufw allow 80/tcp
		sudo ufw allow 123/udp
		sudo ufw enable

## Install and configure Apache HTTP server

		sudo apt-get install apache2

Test the apache2 install by visiting the server IP in a browser. An Apache default page should appear. With default settings, apache will serve content from /var/www/html for any request it receives. If the server needs to serve different content based on the address by which it is accessed, additional <VirtualHost> directives must be added to the conf files in the /etc/apache2/sites_enabled directory. 
Next, install the tools that enable apache to work with python.

		sudo apt-get install python-setuptools libapache2-mod-wsgi

## Install Flask 
Step-by-step instructions are at [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps). This will also configure Apache to serve the new Flask app. One command is missing from these instructions. Since we want our Flask app to respond to requests at the server root, we need to disable the default behavior.

		sudo a2dissite 000-default

## Install extra packages
Enter the virtual environment in which Flask is installed

		source venv/bin/activate

Run the following commands to install (if not already installed) the necessary packages

		sudo pip install httplib2
		sudo pip install requests
		sudo pip install oauth2client
		sudo pip install sqlalchemy
		sudo apt-get install python-psycopg2
		deactivate

## Install and configure PostgreSQL

		sudo apt-get install postgresql postgresql-contrib
	
Set a password for the newly created postgres linux user to access the postgres database. First, open a postgres prompt as the postgres user by running

		sudo -u postgres psql postgres
	
then type

		\password postgres
	
and enter a new password twice. Then exit the postgres prompt by typing

		\q

Create a new postgres user called catalog and a new database, also called catalog, by running

		sudo -u postgres createuser -P --interactive
		sudo -u postgres createdb -O catalog catalog
	
In the first command, the -P flag will immediately prompt for a password for the new user, and --interactive will ask questions about the rights granted to the new user. Answer 'n' to all of these since the catalog user only needs to access a single database, and it's already been created. In the second command, -O creates the new db with owner catalog. Next, lock down the new db.

		sudo -u postgres psql catalog
		REVOKE ALL ON SCHEMA public FROM public;
		GRANT ALL ON SCHEMA public TO catalog;
		\q

## Install and configure git 
by following the instructions at [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-14-04).

## Clone the Catalog app repo

		git clone https://github.com/Scienxe/udacity-catalog.git

Move the entire contents of the new folder into the /var/www/FlaskApp/FlaskApp directory created earlier, and delete the now-empty folder. Hide the .git repository from public view by creating a new .htaccess file in the same directory, with the contents

		RedirectMatch 404 /\.git

Since the client\_secrets.json file should not be in the public repo, copy its contents into a new file of the same name in /var/www/FlaskApp, then provide absolute links to it in the catalog.py app. This directory is not web-accessible, so client\_secrets.json is protected from public view. Now all files are in place, and some edits are required.

## Modify catalog app to use PostgreSQL
In the catalog app files, replace all instances of

		engine = create_engine('sqlite:///catalog.db')

with 

		engine = create_engine('postgresql://catalog:CATALOG-USER-PW@localhost/catalog')

Rename the main catalog.py app file to \_\_init\_\_.py.

## Start the app

		sudo service apache2 restart
		
## Update OAuth2 settings
Visit the console of any third-party auth providers and update the authorized addresses to include the new hostname.

## Open the new hostname in a web browser
For my catalog app, visit [Udacious Musical Instruments](http://ec2-54-68-165-124.us-west-2.compute.amazonaws.com/catalog/).

## Resources
In addition to the pages linked above, thanks are due to the following authors of extremely helpful resources:

* [Keenan Payne](http://blog.udacity.com/2015/03/step-by-step-guide-install-lamp-linux-apache-mysql-python-ubuntu.html)
* [allan](https://discussions.udacity.com/t/markedly-underwhelming-and-potentially-wrong-resource-list-for-p5/8587)
* [stueken](https://github.com/stueken/FSND-P5_Linux-Server-Configuration)
* [kirkbrunson](https://discussions.udacity.com/t/project-5-resources/28343)







