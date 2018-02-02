# Deploy to Remote Server
+ ip: 18.216.245.67
+ port: 2200
+ [webapp](http://salahahmed.me)

### Software Installed
1. update and upgrade
```
sudo apt-get update
sudo apt-get upgrade
```
2. Install packages
```
sudo apt-get install python3-pip
sudo apt-get install libapache2-mod-wsgi-py3
sudo apt-get install tmux
sudo apt-get install git
sudo apt-get install postgresql postgresql-contrib
sudo apt-get install apache2
sudo apt-get install jq
sudo pip install virtualenv
```


### Setup Postgresql
1. Add a database user (catalog)
```
sudo -i -u postgres
createuser --interactive
```
2. Create a database (catalog)
```
createdb catalog
```
3. Create a user (catalog)
```
sudo adduser catalog
```
```
sudo passwd catalog
```
4. Set the Password for user "catalog" to use database "catalog"
```
sudo -u catalog psql -d catalog
#ALTER USER 'catalog' WITH PASSWORD 'new_password';
```


### Firewall
1. Reconfigure SSH Port, do not allow root login or password authentication
```
sudo sed -i 's/Port 22/Port 2200/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

2. Setup firewall
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
sudo ufw status
```

3. Reload SSHD
```
sudo systemctl reload sshd
```

4. set up ssh config
```
# In your localhost setup your config file like so
Host remote_server
    HostName 18.216.245.67
    Port 2200
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    ForwardAgent yes
```


### Add User
1. Add user
```
sudo adduser grader
```

2. Place user is sudoers group
```
sudo usermod -a -G admin grader
```

### Setup SSH
1. make appropriate folders
```
sudo su - grader
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
```

2. On local host generate key pairs
```
ssh-keygen -t rsa -b 4096
```

3. Send Public Key to Remote server and Copy to authorized\_keys
```
rsync key.pub remote_server:~/
ssh remote_server
sudo mv key.pub /home/grader/.ssh/
sudo su - grader
sudo chown -R grader:grader ~/.ssh/
echo ~/.ssh/key.pub >> ~/.ssh/authorized_keys
rm ~/.ssh/key.pub
```


### Clone repo
If you ssh with "-A" flag then you github keys will be saved

```
cd /var/www
git clone git@github.com:salah93/catalog_app
cd catalog_app/
virtualenv -p `which python3` env
source env/bin/activate
pip install -r requirements.txt
```



### Setup apache2
1. add config files
```
sudo cat <<EOF >  /etc/apache2/sites-available/catalog.conf
<VirtualHost *:80>
    ServerName 18.216.245.67

    WSGIScriptAlias / /var/www/catalog_app/wsgi.py

    <Directory /var/www/catalog_app>
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>
EOF

cat <<EOF >  wsgi.py
import logging
import site
import sys
from os.path import join, dirname, expanduser

# Add virtualenv site packages
site.addsitedir(join(dirname(__file__), 'env/lib/python3.5/site-packages'))

# Path of execution
sys.path.insert(0, '/var/www/catalog_app')

# Fire up virtualenv before including application
activate_env = expanduser(join(dirname(__file__), 'env/bin/activate_this.py'))
exec(open(activate_env).read(), {'__file__': activate_env})


from run import app as application, log
from utility import random_string


application.secret_key = random_string(30)
logging.basicConfig(filename=log,level=logging.DEBUG)
EOF
```


2. enable wsgi mod, then site
```
sudo a2enmod wsgi
sudo a2ensite catalog  # enable site
```

3. allow apache permission to write to log
```
APP_LOG=`cat config.json | jq '.log'`
touch $APP_LOG
chown ubuntu:www-data $APP_LOG
```

2. Reload
```
sudo service apache2 reload
```

### Set up Domain Name
1. Choose a DNS service
2. Buy a domain
3. Create an "A" record, pointing to your ip address

### Viewing Logs
```
sudo journalctl -u apache2
sudo less /var/log/apache2/error.log
tail -f error.log
```

### Resources
+ [install and use postgres](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
+ [wsgi and apache configuration](http://peatiscoding.me/geek-stuff/mod_wsgi-apache-virtualenv/)
+ [Linking domain name to ip](https://www.godaddy.com/help/add-an-a-record-19238)
