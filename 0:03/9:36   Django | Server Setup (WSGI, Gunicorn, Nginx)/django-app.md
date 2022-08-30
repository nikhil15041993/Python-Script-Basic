## Start a Django App
Every project you build with Django can contain multiple Django apps. When you ran the startproject command in the previous section, you created a management app that you’ll need for every default project that you’ll build. Now, you’ll create a Django app that’ll contain the specific functionality of your web application.

You don’t need to use the django-admin command-line utility anymore, and you can execute the startapp command through the manage.py file instead:
```
(env) $ python manage.py startapp <appname>
```

The startapp command generates a default folder structure for a Django app. This tutorial uses example as the name for the app:

```
python manage.py startapp example
```

Once the startapp command has finished execution, you’ll see that Django has added another folder to your folder structure:

```
myproject/
│
├── example/
│   │
│   ├── migrations/
│   │   └── __init__.py
│   │
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
│
├── myproject/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
└── manage.py
```

The new folder has the name you gave it when running the command. In the case of this tutorial, that’s example/. You can see that the folder contains a couple of Python files.


###  Views

Django views are Python functions that takes http requests and returns http response, like HTML documents.

A web page that uses Django is full of views with different tasks and missions.

Views are usually put in a file called views.py located on your app's folder.

There is a views.py in your example folder that looks like this:

example/views.py:
```
from django.shortcuts import render
```
Find it and open it, and replace the content with this:

```
from django.shortcuts import render
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello world!")
```

### URLs
Create a file named urls.py in the same folder as the views.py file, and type this code in it:

example/urls.py:

```
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

The urls.py file you just created is specific for the examle application. We have to do some routing in the root directory myproject as well.

myprojectmyworld/urls.py:

```
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('members/', include('members.urls')),
    path('admin/', admin.site.urls),
]
```

In the browser window, type 127.0.0.1:8000/examples/ in the address bar.
