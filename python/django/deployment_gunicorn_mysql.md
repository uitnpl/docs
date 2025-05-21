# Deploying Django with Gunicorn, Nginx and MySQl on Ubuntu

## Setting up software

1. Update ubuntu software repository
2. Install 
   1. nginx - Serve our website 
   2. mysql-server and libmysqlclient-dev - For database
   3. Python 3
   4. ufw - Firewall for our system
   5. Git - Version control system

```bash
$ apt-get update
$ apt-get install nginx mysql-server python3-pip python3-dev libmysqlclient-dev ufw python3-venv pkg-config default-libmysqlclient-dev build-essential
```

## Installing python libraries

1. Create a directory to hold all django apps (Eg. django)
2. Create a virtual environment
3. Clone/extract repository into django directory
3. Install the requirements via txt file.

```bash
$ mkdir epas
$ cd epas
$ git clone https://github.com/uitnpl/gt-epms
$ cd gt-epms
$ mkdir logs
$ python3 -m venv appenv
$ source appenv/bin/activate
$ pip install -r requirements.txt
```

Also install any other dependencies needed

## Setting up the firewall

We'll enable default firewall ports & allow 8000 for now. Later on we'll remove this give access to all ports that nginx needs

```bash
$ ufw enable
$ ufw allow 8000
```

## Setting up database

1. Install mysql and give root password
2. Create database for the application
3. Create a user for the application and give all access to the database
   
``` bash
$ mysql_secure_installation
```

Give password for root user

``` bash
$ sudo mysql -u root -p
$ CREATE DATABASE tnpl_epas CHARACTER SET 'utf8';
$ CREATE USER tnpl_app_user
$ GRANT ALL ON tnpl_epms.* TO 'tnpl_app_user' IDENTIFIED BY '<app user password>';
$ quit
```
## Setting up the django project

1. Modify the `settings.py` file to use the newly created database.

``` python
DEBUG = False

...

DATABASES = {
    'default': {
         'ENGINE': 'django.db.backends.mysql',
         'OPTIONS': {
             'sql_mode': 'traditional',
         },
         'NAME': 'tnpl_epms',
         'USER': 'tnpl_app_user',
         'PASSWORD': '<app user password>',
         'HOST': 'localhost',
         'PORT': '3306', 
     }
}

...

```
Add appropriate ip address or domain name to allowed hosts.

2. Make database migrations

``` bash
(appenv) $ python manage.py makemigrations
(appenv) $ python manage.py migrate
(appenv) $ python manage.py collectstatic
(appenv) $ python manage.py seeddata
```

These commands create all required tables in the new database created and also collects all static files to a static folder under GoalsTracker directory.

## Permissions

Please run the following command to provide necessary permission to appuser.

```bash
chown -R epasusr:www-data /epas
chmod -R 755 /epas
```

## Setting up Gunicorn

1. Lets daemonise the gunicorn

    1. Open a new gunicorn.service file
        ```bash
        vi /etc/systemd/system/gunicorn.service
        ```
    2. Copy the following lines with appropriate modifications
        ```
         [Unit]
         Description=gunicorn service
         After=network.target
         
         [Service]
         User=epasusr
         Group=www-data
         WorkingDirectory=/epas/gt-epms/
         ExecStart=/epas/gt-epms/appenv/bin/gunicorn  --access-logfile /epas/logs/gunicorn-access.log --error-logfile /epas/logs/gunicorn-error.log  --workers 4 --bind unix:/epas/gt-epms/tnpl_epms.sock GoalsTracker.wsgi:application
         
         [Install]
         WantedBy=multi-user.target
        ```

    3. Enable the daemon

        ```bash
        $ sudo systemctl enable gunicorn.service
        $ sudo systemctl start gunicorn.service
        $ sudo systemctl status gunicorn.service
        ```

    4. If something went wrong, do

        ```
        $ journalctl -u gunicorn
        ```

        If journalctl isnt found, ```sudo apt install journalctl```

    5. If you do any changes to your django application, reload the gunicorn server by
        ```
        $ sudo systemctl daemon-reload
        $ sudo systemctl restart gunicorn
        ```

Thats all for gunicorn

## Setting up Nginx

1. Create a new configuration file for nginx

```
$ sudo vi /etc/nginx/sites-available/tnpl_epms
```

2. Copy following lines with appropriate modifications

```
server {
       listen 80;    
       server_name <ip>;
       location = /favicon.ico {access_log off;log_not_found off;} 
    
        location /static/ {
            alias /home/<user>/django/gt-epms/static/;    
        }

        location /media/ {
            alias /home/<user>/django/gt-epms/media/;
        }
    
        location / {
            include proxy_params;
            proxy_pass http://unix:/epas/gt-epms/tnpl_epms.sock;
        }
     }
```

`listen 80` tell nginx which port to listen to. 
`server_name 127.0.0.1` tells that we are using 127.0.0.1 as the address to access our website. This can be set to the ip address of the machine.

3. Link the configuration so that nginx can recognise
```
$ sudo ln -s /etc/nginx/sites-available/tnpl_epms /etc/nginx/sites-enabled
```

4. Check whether everything is fine
```
$ sudo nginx -t
```
If it returns without any errors, then everything is fine. If not go through the configuration once more and debug.

5. Once everything is done, restart nginx
```
$ sudo systemctl restart nginx
```

6. Now that all setup is done, lets delete the testing firewall rule we had put for 8800 and allow all ports necessary for nginx

```
$ sudo ufw delete allow 8800
$ sudo ufw allow 'Nginx Full'
```


And thats it! If everything worked fine, you should be able to access your django application on 127.0.0.1 or any other ip address you gave on nginx configuration
