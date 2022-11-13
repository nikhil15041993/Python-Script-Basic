## Dockerizing Django with Postgres, Gunicorn, and Nginx


Dependencies:

Django v3.2.6
Docker v20.10.8
Python v3.9.6



### Project Setup
Create a new project directory along with a new Django project:

```
$ mkdir django-on-docker && cd django-on-docker
$ mkdir app && cd app
$ python3.9 -m venv env
$ source env/bin/activate
(env)$

(env)$ pip install django==3.2.6
(env)$ django-admin.py startproject hello_django .
(env)$ python manage.py migrate
(env)$ python manage.py runserver
```
Navigate to http://localhost:8000/ to view the Django welcome screen. Kill the server once done. Then, exit from and remove the virtual environment. We now have a simple Django project to work with.


Create a ```requirements.txt``` file in the "app" directory and add Django as a dependency:
```
Django==3.2.6
```

Since we'll be moving to Postgres, go ahead and remove the ```db.sqlite3``` file from the "app" directory.

Your project directory should look like:
```
└── app
    ├── hello_django
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── requirements.txt

```

## Docker
Install Docker, if you don't already have it, then add a Dockerfile to the "app" directory:


```
# pull official base image
FROM python:3.9.6-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .
```



Next, add a docker-compose.yml file to the project root:

```
version: '3.8'

services:
  web:
    build: ./app
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
  ```
  
  
Update the SECRET_KEY, DEBUG, and ALLOWED_HOSTS variables in settings.py:

```
SECRET_KEY = os.environ.get("SECRET_KEY")

DEBUG = int(os.environ.get("DEBUG", default=0))

# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")
```
 Make sure to add the import to the top:

```
import os
```

Then, create a .env.dev file in the project root to store environment variables for development:

```
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
```

Build the image:

```
$ docker-compose build
```
Once the image is built, run the container:

```
$ docker-compose up -d
```

Check for errors in the logs if this doesn't work via docker-compose logs -f.



## Postgres

To configure Postgres, we'll need to add a new service to the  ```docker-compose.yml``` file, update the Django settings, and install Psycopg2.

```
version: '3.8'

services:
  web:
    build: ./app
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=hello_django
      - POSTGRES_PASSWORD=hello_django
      - POSTGRES_DB=hello_django_dev

volumes:
  postgres_data:
``` 

To persist the data beyond the life of the container we configured a volume. This config will bind ```postgres_data``` to the ```"/var/lib/postgresql/data/"``` directory in the container.

We also added an environment key to define a name for the default database and set a username and password.


We'll need some new environment variables for the web service as well, so update ```.env.dev``` like so:

```
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
```


Update the DATABASES dict in settings.py:

```
DATABASES = {
    "default": {
        "ENGINE": os.environ.get("SQL_ENGINE", "django.db.backends.sqlite3"),
        "NAME": os.environ.get("SQL_DATABASE", BASE_DIR / "db.sqlite3"),
        "USER": os.environ.get("SQL_USER", "user"),
        "PASSWORD": os.environ.get("SQL_PASSWORD", "password"),
        "HOST": os.environ.get("SQL_HOST", "localhost"),
        "PORT": os.environ.get("SQL_PORT", "5432"),
    }
}
```

Here, the database is configured based on the environment variables that we just defined. Take note of the default values.

Update the Dockerfile to install the appropriate packages required for Psycopg2:


```
# pull official base image
FROM python:3.9.6-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .

```

Add Psycopg2 to requirements.txt:

```
Django==3.2.6
psycopg2-binary==2.9.1
```

Build the new image and spin up the two containers:

```
$ docker-compose up -d --build
```

Run the migrations:

```
docker-compose exec web python manage.py migrate --noinput
```


Ensure the default Django tables were created:

```
$ docker-compose exec db psql --username=hello_django --dbname=hello_django_dev

psql (13.0)
Type "help" for help.

hello_django_dev=# \l
                                          List of databases
       Name       |    Owner     | Encoding |  Collate   |   Ctype    |       Access privileges
------------------+--------------+----------+------------+------------+-------------------------------
 hello_django_dev | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres         | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 template0        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
 template1        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
(4 rows)

hello_django_dev=# \c hello_django_dev
You are now connected to database "hello_django_dev" as user "hello_django".

hello_django_dev=# \dt
                     List of relations
 Schema |            Name            | Type  |    Owner
--------+----------------------------+-------+--------------
 public | auth_group                 | table | hello_django
 public | auth_group_permissions     | table | hello_django
 public | auth_permission            | table | hello_django
 public | auth_user                  | table | hello_django
 public | auth_user_groups           | table | hello_django
 public | auth_user_user_permissions | table | hello_django
 public | django_admin_log           | table | hello_django
 public | django_content_type        | table | hello_django
 public | django_migrations          | table | hello_django
 public | django_session             | table | hello_django
(10 rows)

hello_django_dev=# \q
```



You can check that the volume was created as well by running:

```
docker volume inspect django-on-docker_postgres_data
```
You should see something similar to:

```
[
    {
        "CreatedAt": "2021-08-23T15:49:08Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "django-on-docker",
            "com.docker.compose.version": "1.29.2",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/django-on-docker_postgres_data/_data",
        "Name": "django-on-docker_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]
```

Next, add an ```entrypoint.sh``` file to the "app" directory to verify that Postgres is healthy before applying the migrations and running the Django development server:

```
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

python manage.py flush --no-input
python manage.py migrate

exec "$@"

```

Update the file permissions locally:

```
chmod +x app/entrypoint.sh
```


Then, update the Dockerfile to copy over the entrypoint.sh file and run it as the Docker entrypoint command:



```
# pull official base image
FROM python:3.9.6-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy entrypoint.sh
COPY ./entrypoint.sh .
RUN sed -i 's/\r$//g' /usr/src/app/entrypoint.sh
RUN chmod +x /usr/src/app/entrypoint.sh

# copy project
COPY . .

# run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```


## Gunicorn

Moving along, for production environments, let's add Gunicorn, a production-grade WSGI server, to the requirements file:

```
Django==3.2.6
gunicorn==20.1.0
psycopg2-binary==2.9.1
```


Since we still want to use Django's built-in server in development, create a new compose file called ```docker-compose.prod.yml``` for production:

```
version: '3.8'

services:
  web:
    build: ./app
    command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
    ports:
      - 8000:8000
    env_file:
      - ./.env.prod
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db

volumes:
  postgres_data:
  
```  


Take note of the default command. We're running Gunicorn rather than the Django development server. We also removed the volume from the web service since we don't need it in production. Finally, we're using separate environment variable files to define environment variables for both services that will be passed to the container at runtime.

.env.prod:

```
DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

.env.prod.db:
```
POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod
```

Add the two files to the project root. You'll probably want to keep them out of version control, so add them to a .gitignore file.

Bring down the development containers (and the associated volumes with the -v flag):

```
docker-compose down -v
```
Then, build the production images and spin up the containers:

```
docker-compose -f docker-compose.prod.yml up -d --build
```


## Nginx

Next, let's add Nginx into the mix to act as a reverse proxy for Gunicorn to handle client requests as well as serve up static files.

Add the service to docker-compose.prod.yml:

```
nginx:
  build: ./nginx
  ports:
    - 1337:80
  depends_on:
    - web
```
Then, in the local project root, create the following files and folders:

```
└── nginx
    ├── Dockerfile
    └── nginx.conf
```

Dockerfile:

```
FROM nginx:1.21-alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d
```

nginx.conf:

```
upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}
```
Review Using NGINX and NGINX Plus as an Application Gateway with uWSGI and Django for more info on configuring Nginx to work with Django.

Then, update the web service, in docker-compose.prod.yml, replacing ports with expose:

```
web:
  build:
    context: ./app
    dockerfile: Dockerfile.prod
  command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
  expose:
    - 8000
  env_file:
    - ./.env.prod
  depends_on:
    - db
```
Now, port 8000 is only exposed internally, to other Docker services. The port will no longer be published to the host machine.

For more on ports vs expose, review this Stack Overflow question.

Test it out again.

```
$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
```
Ensure the app is up and running at http://localhost:1337.

Your project structure should now look like:
```

├── .env.dev
├── .env.prod
├── .env.prod.db
├── .gitignore
├── app
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── entrypoint.prod.sh
│   ├── entrypoint.sh
│   ├── hello_django
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── manage.py
│   └── requirements.txt
├── docker-compose.prod.yml
├── docker-compose.yml
└── nginx
    ├── Dockerfile
    └── nginx.conf

```
