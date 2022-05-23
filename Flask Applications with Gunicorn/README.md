## Step 1 — Installing the Components from the Ubuntu Repositories

The first step is to install all of the necessary packages from the default Ubuntu repositories. This includes pip, the Python package manager, which will manage your Python components. You’ll also get the Python development files necessary to build some of the Gunicorn components.

First, update the local package:
```
sudo apt update
```
Then install the packages that will allow you to build your Python environment. These include python3-pip, along with a few more packages and development tools necessary for a robust programming environment:

```
sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
```

## Step 2 — Creating a Python Virtual Environment

Next, set up a virtual environment to isolate your Flask application from the other Python files on the system.

Start by installing the python3-venv package, which will install the venv module:
```
sudo apt install python3-venv
```
Next, make a parent directory for your Flask project:

```
mkdir ~/myproject
```
Then change into the directory after you create it:

```
cd ~/myproject
```
Create a virtual environment to store your Flask project’s Python requirements by entering the following:

```
python3.6 -m venv myprojectenv
```

This will install a local copy of Python and pip into a directory called myprojectenv within your project directory.

Before installing applications within the virtual environment, you need to activate it by running the following:
```
source myprojectenv/bin/activate
```
Your prompt will change to indicate that you are now operating within the virtual environment. It will read like the following:

```
(myprojectenv)\ssammy@host:~/myproject$
```

## Step 3 — Setting Up a Flask Application

Now that you are in your virtual environment, you can install Flask and Gunicorn and get started on designing your application.

First, install wheel with the local instance of pip to ensure that your packages will install even if they are missing wheel archives:

```
pip install wheel
```

Note: Regardless of which version of Python you are using, when the virtual environment is activated, you should use the pip command (not pip3).

Next, install Flask and Gunicorn:

```
pip install gunicorn flask
```

Now that you have Flask available, you can create a basic application in the next step.


### Creating a Sample Application

```
vi ~/myproject/myproject.py
```

```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "<h1 style='color:blue'>Hello There!</h1>"

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

If you followed the initial server setup guide in the prerequisites, you should have a UFW firewall enabled. To test the application, first you need to allow access to port 5000:

```
sudo ufw allow 5000
```
Then you can test your Flask application by running the following:

```
python myproject.py
```

You will receive output like the following, including a helpful warning reminding you not to use this server setup in production:

```
Output
 * Serving Flask app 'myproject' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://your_server_ip:5000/ (Press CTRL+C to quit)

```
Visit your server’s IP address followed by :5000 in your web browser:

```
http://your_server_ip:5000
```

### Creating the WSGI Entry Point

Next, create a file that will serve as the entry point for your application. This will tell your Gunicorn server how to interact with the application.

Create a new file using your preferred text editor and name it. Here, we’ll call the file wsgi.py:
```
vi ~/myproject/wsgi.py
```
In this file, import the Flask instance from your application and then run it:

```
~/myproject/wsgi.py
```


```
from myproject import app

if __name__ == "__main__":
    app.run()

```



## Step 4 — Configuring Gunicorn

Your application is now written with an entry point established and you can proceed to configuring Gunicorn.

But first, change into to the appropriate directory:
```
cd ~/myproject
```

Next, you can check that Gunicorn can serve the application correctly by passing it the name of your entry point. This is constructed as the name of the module (minus the .py extension), plus the name of the callable within the application. In our case, this is written as wsgi:app.

You’ll also specify the interface and port to bind to so that the application will be started on a publicly available interface:

```
gunicorn --bind 0.0.0.0:5000 wsgi:app
```

You will receive output like the following:

```
Output
[2021-11-19 23:07:57 +0000] [8760] [INFO] Starting gunicorn 20.1.0
[2021-11-19 23:07:57 +0000] [8760] [INFO] Listening at: http://0.0.0.0:5000 (8760)
[2021-11-19 23:07:57 +0000] [8760] [INFO] Using worker: sync
[2021-11-19 23:07:57 +0000] [8763] [INFO] Booting worker with pid: 8763
[2021-11-19 23:08:11 +0000] [8760] [INFO] Handling signal: int
[2021-11-19 23:08:11 +0000] [8760] [INFO] Shutting down: Master
```
Visit your server’s IP address with :5000 appended to the end in your web browser again:
```
http://your_server_ip:5000
```

When you have confirmed that it’s functioning properly, press CTRL + C in your terminal window.

Since now you’re done with your virtual environment, deactivate it:

```
deactivate
```

Next, create the systemd service unit file

/etc/systemd/system/myproject.service4

```
[Unit]
Description=Gunicorn instance to serve myproject
After=network.target

[Service]
User=sammy
Group=www-data
WorkingDirectory=/home/sammy/myproject
Environment="PATH=/home/sammy/myproject/myprojectenv/bin"
ExecStart=/home/sammy/myproject/myprojectenv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target

```
Now start the Gunicorn service you created:
```
sudo systemctl start myproject
```
Then enable it so that it starts at boot:
```
sudo systemctl enable myproject
```
Check the status:
```
sudo systemctl status myproject
```

## Step 5 — Configuring Nginx to Proxy Requests

 creating a new server block configuration file in Nginx’s sites-available directory. We’ll call this myproject to stay consistent with the rest of the guide:

```
sudo nano /etc/nginx/sites-available/myproject
```

Open up a server block and tell Nginx to listen on the default port 80. Also, tell it to use this block for requests for your server’s domain name:


Next, add a location block that matches every request. Within this block, include the proxy_params file that specifies some general proxying parameters that need to be set. Then pass the requests to the socket you defined using the proxy_pass directive:

/etc/nginx/sites-available/myproject

```
server {
    listen 80;
    server_name your_domain www.your_domain;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/sammy/myproject/myproject.sock;
    }
}
```

To enable the Nginx server block configuration you’ve created, link the file to the sites-enabled directory. You can do this by running the ln command and the -s flag to create a symbolic or soft link, as opposed to a hard link:

```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```
With the link in that directory, you can test for syntax errors:

```
sudo nginx -t
```
If this returns without indicating any issues, restart the Nginx process to read the new configuration:

```
sudo systemctl restart nginx
```
Finally, adjust the firewall again. Since you no longer need access through port 5000, remove that rule:
```
sudo ufw delete allow 5000
```
Then allow full access to the Nginx server:

```
sudo ufw allow 'Nginx Full'
```
You should now be able to navigate to your server’s domain name in your web browser:

http://your_domain

Your application’s output will appear in your browser:


## Step 6 — Securing the Application

o ensure that traffic to your server remains secure, you should get an SSL certificate for your domain. There are multiple ways to do this, including getting a free certificate from Let’s Encrypt, generating a self-signed certificate, or buying one from another provider and configuring Nginx to use it by following Steps 2 through 6 of How to Create a Self-signed SSL Certificate for Nginx in Ubuntu 18.04. We will go with option one for the sake of expediency.

First, install Certbot using snap:

```
sudo snap install --classic certbot
```

Your output will display the current version of Certbot and indicate a successful installation:

```
Output
certbot 1.21.0 from Certbot Project (certbot-eff✓) installed
```
Next, create a symbolic link to the newly installed /snap/bin/certbot executable from the /usr/bin/ directory. This will ensure that the certbot command can run correctly on your server:

```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
Certbot provides a variety of ways to obtain SSL certificates through plugins. The Nginx plugin will take care of reconfiguring Nginx and reloading the configuration whenever necessary. To use this plugin, type the following:

```
sudo certbot --nginx -d your_domain -d www.your_domain
```

This runs certbot with the --nginx plugin, using -d to specify the names you’d like the certificate to be valid for.

To verify the configuration, navigate once again to your domain, using https://:

https://your_domain
