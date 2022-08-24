In this tutorial Django project runs in development and production environments.
I also show how to run a Django project using a WSGI server (Gunicorn) and a web server (Nginx).

## Step 1. Installing the Packages from the Ubuntu Repositories

```
sudo apt install libpq-dev postgresql postgresql-contrib nginx
```

## Step 2. Creating the PostgreSQL Database and User

We’re going to jump right in and create a database and database user for our Django application.

Log into an interactive Postgres session
```
sudo -u postgres psql
```

First, create a database for your project:

```
postgres=# CREATE DATABASE myproject;
```

Next, create a database user for our project.

```
postgres=# CREATE USER nikhil WITH PASSWORD 'password';
```

```
postgres=# ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
postgres=# ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE myprojectuser SET timezone TO 'UTC';
```
Now, we can give our new user access to administer our new database:

```
GRANT ALL PRIVILEGES ON DATABASE myproject TO nikhil;
```
exit out of the PostgreSQL
```
postgres=# \q
```

## Step 3.  Create virtual environment for your Project using Anaconda 

### Download And Install Anaconda
```
curl -O https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh

sha256sum Anaconda3-2021.11-Linux-x86_64.sh

bash Anaconda3-2021.11-Linux-x86_64.sh

```

Once finished, activate the installation by entering:
```
source ~/.bashrc

conda --version or  /home/ubuntu/anaconda3/condabin/conda --version
```
### List Packages

```
conda list 
```
it list the pakages under defalut environment
```
conda < env name> list 
```
it will list the packages from the specific environment

### Create virtual environment

```
conda create --name <name of env> <python=version>
```
Active newly created virtual environent
```
source activate <name of virtual environment>
```

To Deactive the environment
```
source deactivate
```



## Step 4. Install Django and Gunicorn

With your virtual environment active, install Django, Gunicorn, and the psycopg2 PostgreSQL adaptor with the local instance of pip:
```
pip install django
pip install gunicorn
pip psycopg2-binary
```

## Step 5. Creating and Configuring a New Django Project

```
django-admin startproject myproject
```
### Adjusting the Project Settings
The first thing we should do with our newly created project files is adjust the settings. Open the settings file in your text editor:

```
ALLOWED_HOSTS = ['<server ip>']

------------------------------------------------------

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'myproject',
        'USER': 'nikhil',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}

-------------------------------------------------------

STATIC_URL = '/static/'
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')

```


### Completing Initial Project Setup

Now, we can migrate the initial database schema to our PostgreSQL database using the management script

```
(myenv) $ myproject/manage.py makemigrations
(myenv) $ myproject/manage.py migrate
```
Create an administrative user for the project by typing
```
(myenv) $ myproject/manage.py createsuperuser
```
collect all of the static content into the directory location we configured by typing
```
(myenv) $ myproject/manage.py collectstatic
```
The static files will then be placed in a directory called static within your project directory.

### Finally, you can test our your project by starting up the Django development server

```
(myenv) $ myproject/manage.py runserver 0.0.0.0:8000
```

visit your server’s domain name or IP address followed by :8000
```
http://server_domain_or_IP:8000
```


### Testing Gunicorn’s Ability to Serve the Project

entering our project directory and using gunicorn to load the project’s WSGI module:

```
(myenv) $ gunicorn --bind 0.0.0.0:8000 myproject.wsgi
```
This will start Gunicorn on the same interface that the Django development server was running on.

When you are finished testing, hit CTRL-C in the terminal window to stop Gunicorn.


## Step 6. Creating systemd Socket and Service Files for Gunicorn

The Gunicorn socket will be created at boot and will listen for connections. When a connection occurs, systemd will automatically start the Gunicorn process to handle the connection

creating a systemd socket file for Gunicorn

```
sudo vi /etc/systemd/system/gunicorn.socket
```

```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

[Unit] section to describe the socket
[Socket] section to define the socket location
[Install] section to make sure the socket is created at the right time


Save and close the file and create systemd service file for Gunicorn 

```
sudo vi /etc/systemd/system/gunicorn.service
```

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=sammy
Group=www-data
WorkingDirectory=/home/ubuntu/anaconda3/envs/myproject
ExecStart=/home/ubuntu/anaconda3/envs/myenv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

[Unit] section, which is used to specify metadata and dependencies.
[Service] section specify the user and group that we want to process to run under.
[Install] section tell systemd what to link this service to if we enable it to start at boot


Save and close it now.

We can now start and enable the Gunicorn socket
This will create the socket file at /run/gunicorn.sock now and at boot. When a connection is made to that socket, systemd will automatically start the gunicorn.service to handle it

```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

Check the status of the process to find out whether it was able to start:
```
sudo systemctl status gunicorn.socket
```

### Testing Socket Activation

Currently, if you’ve only started the gunicorn.socket unit, the gunicorn.service will not be active yet
```
sudo systemctl status gunicorn
```

To test the socket activation mechanism, we can send a connection to the socket through curl by typing:
```
curl --unix-socket /run/gunicorn.sock localhost
```

You can verify that the Gunicorn service is running

```
sudo systemctl status gunicorn

sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```


## Step 7. Configure Nginx to Proxy Pass to Gunicorn

we need to configure Nginx to pass traffic to the process.
Start by creating and opening a new server block in Nginx’s sites-available directory

```
sudo nano /etc/nginx/sites-available/myproject
```

```
server {
    listen 80;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/ubuntu/anaconda3/envs/myproject;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}

```

Save and close the file when you are finished. Now, we can enable the file by linking it to the sites-enabled

```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

Test your Nginx configuration for syntax errors by typing:
```
sudo nginx -t

sudo systemctl restart nginx
```










