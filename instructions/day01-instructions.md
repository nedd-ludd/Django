# DJANGO DAY 1 - FIRST BASIC APP

## SET UP AND INSTALLATION

1. First of all, we need to install something called pipenv, Pipenv is a tool that provides all necessary means to create a virtual environment for your Python project. It automatically manages project packages through the Pipfile file as you install or uninstall packages. Pipenv also generates the Pipfile. It's like NPM for python projects. We only need to run this once as it will install globally.

```bash
pip install pipenv
```

2. Now we need to create a project folder and navigate into it.

```bash
mkdir django_day_one_basic_app && cd django_day_one_basic_app
```

3. Now we need to install django. Let's get pipenv to create a virtual environment and install django into that env.

```bash
pipenv install django
```

4. Let's see the result - ls inside the folder and make sure there is a `Pipfile` and a `Pipfile.lock` file. These are the equivalent of `package.json` and `package-lock.json` in node.

```bash
ls
```

5. open in vs code and open the integrated terminal

6. enter the shell

```bash
pipenv shell # note the changes in terminal and explain the shell
```

<em>Pipenv is a packaging tool for Python that solves some common problems associated with the typical workflow using pip , virtualenv , and the good old requirements. txt . In addition to addressing some common issues, it consolidates and simplifies the development process to a single command line tool.</em>

If you don't do it in the shell, it's not happening, when it comes to the manage.py!

7. start new project

```bash
django-admin startproject project .
```

8. Let's see what we got from that:

- get a new folder and lots of files, lots that we wont touch
- `manage.py`: this is our control center. It holds scripts to start the app, create database migrations, create superusers, seed the DB and much more. We don't edit this file.

- `init.py`: file that lets us write things that can be exported from a top level, so stuff we might want to use everywhere

- `asgi.py`: config for web servers, never use it
- `settings.py`: generates key for us, holds the overall project settings.
- `urls.py`: basically the router - we import individual app urls and bring them together here to make the project urls (router)

- click python in vscode at the bottom -> change to shell
- if not prompted -> `pipenv install pylint` -> check in pipfile
- dont want all the linter rules -> make file to change some settings
  - `touch .pylintrc`
  - add config

```py
[MESSAGES CONTROL]
disable=arguments-differ,missing-function-docstring,missing-class-docstring,no-self-use,raise-missing-from,no-member,missing-module-docstring,invalid-name,too-few-public-methods

```

## STARTING THE APP

<strong>Make sure you're in the temminal window that has the shell running.</strong></br>

1. We use the manage.py file to run functions within the file to execute commands on our application. To start the command, we run `python manage.py runserver` in the shell.

```bash
python manage.py runserver # it throws an error the first time
```

## The First Error

- Django comes with lots out of the box including
- user management, have and auth users already
- wants to run but needs a db
- default is SQLite (there a file called db.sqlite3 - that's our default db!)
- migrations are changes to the sql db, here django is saying ive got this whole db and I need to build it to run the app, can I put this info somewhere please? we need to make changes to the db or "migrations" so it can create tables etc.

- let's create or "migrate" the changes to the database that we need:

```bash
python manage.py migrate
```

- Now our changes have been made, we can restart the app and hopefully we won't have any errors!

```bash
python manage.py runserver # no errors!

# Django version 4.1.5, using settings 'project.settings'
# Starting development server at http://127.0.0.1:8000/
# Quit the server with CONTROL-C.
```

The feedback tells us we have started the development server at http://127.0.0.1:8000/ so let's go there in the browser and see what awaits. Should be a success message with something like <em>The install worked successfully! Congratulations!</em>

- as we write our own code this will go and we can start interacting with it
- From here we can access the Django Admin site (created for us because we love free stuff)

2. `/admin` to see login

## Creating an Admin User (superuser)

1. Make sure the project is running. If it's not then start the shell and run the runserver command.
2. If you get errors about migrations, see above re: `python manage.py migrate` and run the server again.
3. go to `localhost:8000/admin` and see the option to login
4. We need to create a superuser so we can login and look around

```bash
python manage.py createsuperuser
# make the credentials easy to remember/type. You'll need them A LOT.
# username: admin
# email: admin@admin.com
# password: admin

# It WILL say your password is too similar to the username but gives you the option to ignore and continue. JUST IGNORE AND CONTINUE.
# we don't need fancy passwords in development, it just slows us down. admin is arguably already too long.
```

5. run then server again and login to user portal from the login page

<em>note a new file has appeared -> sqllite -> txt db that works like our sql db for now</em>

## IT'S PARTY TIME - LET'S BUILD AN APP

1. Create a new "app" in the project

```bash
django-admin startapp albums
```

2. tell django about new app when created. Go to the project folder and open the settings.py. We need to add our albums app to the installed apps.

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'albums' # add this line
]
```

3. create a model in `albums/models.py` -> note import is different

```py
from django.db import models
# django has its own model class we can use, models.Model -> inherit properties from this class

# this is a child class of the django Model class
class Album(models.Model): # inside the brackets is where its inheriting from
    title = models.CharField(max_length=50)
    author = models.CharField(max_length=50)
    cover_image = models.CharField(max_length=300)
    # represents the class objects as a string, add in after
    def __str__(self):
        return f"{self.title} - {self.author}"

# makemigrations is responsible for packaging up your model changes into individual migration files
# migrate is responsible for applying those to your database.
```

4. make migrations

```bash
python manage.py makemigrations
```

5. migrate

```bash
python manage.py migrate
```

7. We need to register our model so the admin site is aware of it and I can see it in the browser. Go to `albums/admin.py` and import the model

```py
from django.contrib import admin
from .models import Album

# Register your models here.
admin.site.register(Album)
```

8. run the server and login to admin site, add an album and
   note the POST request showing in terminal

# ADDING IN DRF -> DJANGO REST FRAMEWORK

1. install the django rest framework

```bash
pipenv install djangorestframework
```

2. register this in our `project/settings.py` -> `INSTALLED_APPS`

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework', # add rest_framework above our own apps
    'albums',  # we add the name of the app here so the project knows it exists
]
```

## Serializers

A serializer takes the data we get from the database and transform it to python data types - postgres stores in tables so needs middleware to convert that data to something usable

The steps a request goes through once it hits our django API are:

make call to db for data -> run through serializer to transform -> sent back to the user

1. inside the `albums` folder create a new file `serializers.py`
2. add the imports to the top of the file and build out the serializer. This is where we define the model that the JSON will be using and specify which fields to look at

```py
from rest_framework import serializers
from .models import Album


class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        model = Album  # the model it should use
        fields = '__all__'  # which fields to serialize
```

3. Move into `views.py` and replace the default imports with:

```py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .models import Album
from .serializers import AlbumSerializer


class AlbumListView(APIView):

    def get(self, _request):
        albums = Album.objects.all()  # get everything from the shows table in the db
        # run everything through the serializer
        serialized_products = AlbumSerializer(albums, many=True)
        # return the response and a status
        return Response(serialized_products.data, status=status.HTTP_200_OK)
```

4. make a new file called `urls.py` inside `albums`.

```py
from django.urls import path  # import path from django
from .views import AlbumListView  # import class from .views

urlpatterns = [
    path('', AlbumListView.as_view()),
]
```

5. `project/urls.py` -> add in the new urlpattern for the list view

```py
"""project URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/4.1/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('albums/', include('albums.urls')),
]
```

6. if needed, make migrations -> migrate -> run server
7. fix intentially thrown error
8. navigate to `localhost:8000/albums` to see the data
9. show in postman too, response being returned as JSON

## KEY THINGS

### VIEWS

Views are essentially our controllers.
The request is directed from the router to the view, which then queries the model and receives a response. We then need to convert that to usable data as the response comes back as a queryset.
Serializers are then used to convert that queryset into a format we specify.

### SERIALIZERS

As above, serializers exist to convert data into a useable data type
They're also used to DEserialize, when we add data back in
