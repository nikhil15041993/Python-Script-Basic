In this tutorial Django project runs in development and production environments.
I also show how to run a Django project using a WSGI server (Gunicorn) and a web server (Nginx).

## Step 1.  Create virtual environment using Anaconda 

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


## Step 2. Install django and gunicorn

```
pip install django
```
```
pip install gunicorn
```

## Step 3. Create a new django project

```
django-admin startproject <name of project>
```
### change the setting.py file of the newly created project

```
ALLOWED_HOSTS = ['<server ip>']
```
## Step 4. Create a configuration file for gunicorn

```
=> mkdir conf
=> sudo vi /conf/gunicorn_config.py
```

Add Following code on gunicorn_config.py file

```
command = '/home/ubuntu/anaconda3/envs/dhango_env/bin/gunicorn'
pythonpath = '/home/ubuntu/anaconda3/envs/myproject'
bind = '127.0.0.1:8000'
workers = 3
```

### Start gunicorn 

```
gunicorn -c conf/gunicorn_config.py myproject.wsgi
```

## Step 5. Start nginx

```
sudo service nginx start
```

### Make a static directory and add it to the setting.py file on project directory

```
mkdir static
```
```
STATIC_URL = '/home/ubuntu/anaconda3/envs/static/'
```
## Step 6. Create a nginx config file for our project

```
sudo vi /etc/nginx/sites-available/myproject
```
Add Following code 

```
server {
    listen 80;
    server_name 13.232.69.8;


location /static/ {
    root /home/ubuntu/anaconda3/envs/static/;
}

location / {
    proxy_pass http://127.0.0.1:8000;
 }
}
```

Enable the nginx newly created sites-avalable

```
sudo ln -s /etc/nginx/sites-available/myproject 
```









