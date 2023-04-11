# SWITCHING THE DB TO POSTGRES

1. We need to install a few packages to get postgres to work in Django, namely Psycopg. It is the most popular PostgreSQL database adapter for Python.

```bash
pipenv install psycopg2-binary
```

1. We need to update our `project/settings.py` to tell Django to use Postgres instead of the deault sqlite database.

```py
DATABASES = { # added this to use postgres as the databse instead of the default sqlite. do this before running the initial migrations or you will need to do it again
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'django-albums',
        'HOST': 'localhost',
        'PORT': 5432
    }
}
```

2. Go to the terminal and create the db. The name of the database we create needs to match the name of the database we put in the Django database settings.

```bash
createdb nameofdbgoeshere
```

3. Stop the server and migrate. As it's a new database, we need to create the tables again in postgres.

```bash
python manage.py migrate
```

5. Run server again -> check for success in browser -> localhost:8000

```bash
python manage.py runserver
```

6. Create a super user

```bash
python manage.py createsuperuser
```

7. Log into the admin site as your superuser and create a few albums.

## MORE METHODS

We can get all of our albums, but currently we can't create an album without having to use the admin app. Let's address this.

1. We need to add a post method to our `AlbumListView` in view.py. We will also need to import the IntegrityError

```py
# this imports a Django core exception: ValidationError
from django.db import IntegrityError
```

```py
def post(self, request):
        album_to_add = AlbumSerializer(data=request.data)
        try:
            album_to_add.is_valid()
            album_to_add.save()
            return Response(album_to_add.data, status=status.HTTP_201_CREATED)
        # exceptions are like a catch in js, but if we specify an exception like we do below then the exception thrown has to match to fall into it
        # For example the below is the exception thrown when we miss a required field
        # link: (this documentation entry is empty but shows it exists) https://docs.djangoproject.com/en/4.0/ref/exceptions/#django.db.IntegrityError
        except IntegrityError as e:
            res = {
                "detail": str(e)
            }
            # IntegrityError is an exception thrown when fields are missing
            return Response(res, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # AssertionError occurs when the incorrect type is passed as a value for an existing field
        # AssertionError is a native python error
        # link: https://docs.python.org/3/library/exceptions.html#AssertionError
        except AssertionError as e:
            return Response({ "detail": str(e) }, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # If we leave it blank, (except:) then all exceptions will fall into it
        # We will add this as a fallback.
        except:
            return Response({ "detail": "Unprocessable Entity" }, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
```

## MORE ROUTES

We can get all of our albums already, let's add a route to get a single album.
The serializer isn't going to change, so this should be very easy.

1. Go to `albums/views.py` and add a new path

```py
path('<int:pk>/', AlbumDetailView.as_view()),
```

The above `<int:pk>` is known as a captured value - it works the same as a placeholder in react/express: ":id".
It's made up of two parts:

1. On the left is the path converter - in this case we've specified an integer or "int"
2. On the right is the placeholder - in this case pk but could be anything the path converter is optional, but you should use it to ensure it's the type you expect without it, the captured value would be written like: `<pk>`

3. Your `albums/urls.py` should now look like this:

```py
from django.urls import path
from .views import AlbumListView, AlbumDetailView

urlpatterns = [
    path('', AlbumListView.as_view()),
    path('<int:pk>/', AlbumDetailView.as_view())
]
```

4. Now we have an endpoint, we need to create the `AlbumDetailView` in our views.py. Let's start with the get method:

```py
class AlbumDetailView(APIView):
    # show this one first
    def get(self, _request, pk):
        try:
            # querying by primary key will always return a single document,
            # so there's no need to pass many=True into the serializer
            album = Album.objects.get(pk=pk)
            serialized_album = AlbumSerializer(album)
            return Response(serialized_album.data, status=status.HTTP_200_OK)
        except Album.DoesNotExist:
            raise NotFound(detail="Can't find that show!")
```

5. We can update our get method to use a function that will go and retrieve the requested album and then pass the data through to the get. This will mean that the fetching of the album from the DB becomes a reusable function.

```py
    # This will be used by all of the routes
    def get_show(self, pk):
        try:
            return Show.objects.get(pk=pk)
        # Django Models have a DoesNotExist exception that occurs when a query returns no results
        # link: https://docs.djangoproject.com/en/4.0/ref/models/class/#model-class-reference
        # we can import that here for use
        except Show.DoesNotExist:
            # We'll raise a NotFound and pass a custom message on the detail key
            # NotFound returns a 404 response
            # link: https://www.django-rest-framework.org/api-guide/exceptions/#notfound
            # raise and return can both be used inside an exception, but NotFound has to be raised
            # raising an exception is when you're indicating a specific behaviour or outcome like NotFound
            # returning an exception is for something generic like Response above
            raise NotFound(detail="🆘 Can't find that show!")

    # show this one after
    def get(self, _request, pk):
        show = self.get_show(pk=pk) # using key word arguments here
        # querying using a primary key is always going to return a single result.
        # this will never be a list, so no need to add many=True on the serializer
        serialized_show = ShowSerializer(show)
        return Response(serialized_show.data, status=status.HTTP_200_OK)
```

6. Now we can add the edit/put method:

```py
    def put(self, request, pk):
        show_to_edit = self.get_show(pk=pk)
        # passing request data and the instance to update through the serializer
        # we specify the key data because we aren't adhering to the order of the arguments, same as pk=pk above and many=True
        updated_show = ShowSerializer(show_to_edit, data=request.data)
        try:
            updated_show.is_valid()
            updated_show.save()
            return Response(updated_show.data, status=status.HTTP_202_ACCEPTED)
        # Exception for when we pass the wrong type for a field
        except AssertionError as e:
            return Response({"detail": str(e)}, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # any other exception
        except:
            res = {
                "detail": "Unprocessable Entity"
            }
            return Response(res, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
```

7. And finally we can add the delete

```py
    def delete(self, _request, pk):
        show_to_delete = self.get_show(pk=pk)
        show_to_delete.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

8. Our final views.py file should look like this:

```py
# this imports rest_frameworks APIView that we'll use to extend to our custom view
from rest_framework.views import APIView
# Response gives us a way of sending a http response to the user making the request, passing back data and other information
from rest_framework.response import Response
# status gives us a list of possible response codes
from rest_framework import status
# This provides a default response for a not found
from rest_framework.exceptions import NotFound
# this imports a Django core exception: ValidationError
from django.db import IntegrityError

from .models import Show
from .serializers import ShowSerializer


class ShowListView(APIView):

    def get(self, _request):
        shows = Show.objects.all()  # get everything from the shows table in the db
        print('shows', shows)
        # run everything through the serializer
        serialized_shows = ShowSerializer(shows, many=True)
        print('serialized shows', serialized_shows)
        # return the response and a status
        return Response(serialized_shows.data, status=status.HTTP_200_OK)

    def post(self, request):
        show_to_add = ShowSerializer(data=request.data)
        try:
            show_to_add.is_valid()
            show_to_add.save()
            return Response(show_to_add.data, status=status.HTTP_201_CREATED)
        # exceptions are like a catch in js, but if we specify an exception like we do below then the exception thrown has to match to fall into it
        # For example the below is the exception thrown when we miss a required field
        # link: (this documentation entry is empty but shows it exists) https://docs.djangoproject.com/en/4.0/ref/exceptions/#django.db.IntegrityError
        except IntegrityError as e:
            res = {
                "detail": str(e)
            }
            # IntegrityError is an exception thrown when fields are missing
            return Response(res, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # AssertionError occurs when the incorrect type is passed as a value for an existing field
        # AssertionError is a native python error
        # link: https://docs.python.org/3/library/exceptions.html#AssertionError
        except AssertionError as e:
            return Response({"detail": str(e)}, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # If we leave it blank, (except:) then all exceptions will fall into it
        # We will add this as a fallback.
        except:
            return Response({"detail": "Unprocessable Entity"}, status=status.HTTP_422_UNPROCESSABLE_ENTITY)


class ShowDetailView(APIView):

    # show this one after
    # This will be used by all of the routes
    def get_show(self, pk):
        try:
            return Show.objects.get(pk=pk)
        # Django Models have a DoesNotExist exception that occurs when a query returns no results
        # link: https://docs.djangoproject.com/en/4.0/ref/models/class/#model-class-reference
        # we can import that here for use
        except Show.DoesNotExist:
            # We'll raise a NotFound and pass a custom message on the detail key
            # NotFound returns a 404 response
            # link: https://www.django-rest-framework.org/api-guide/exceptions/#notfound
            # raise and return can both be used inside an exception, but NotFound has to be raised
            # raising an exception is when you're indicating a specific behaviour or outcome like NotFound
            # returning an exception is for something generic like Response above
            raise NotFound(detail="🆘 Can't find that show!")

    # show this one after
    def get(self, _request, pk):
        show = self.get_show(pk=pk)  # using key word arguments here
        # querying using a primary key is always going to return a single result.
        # this will never be a list, so no need to add many=True on the serializer
        serialized_show = ShowSerializer(show)
        return Response(serialized_show.data, status=status.HTTP_200_OK)

    # # show this one first

    def get(self, _request, pk):
        try:
            # different API methods https://docs.djangoproject.com/en/4.0/ref/models/querysets/#methods-that-do-not-return-querysets
            show = Show.objects.get(pk=pk)
            serialized_show = ShowSerializer(show)
            return Response(serialized_show.data, status=status.HTTP_200_OK)
        except Show.DoesNotExist:
            raise NotFound(detail="🆘 Can't find that show!")

    def put(self, request, pk):
        show_to_edit = self.get_show(pk=pk)
        # passing request data and the instance to update through the serializer
        # we specify the key data because we aren't adhering to the order of the arguments, same as pk=pk above and many=True
        updated_show = ShowSerializer(show_to_edit, data=request.data)
        try:
            updated_show.is_valid()
            updated_show.save()
            return Response(updated_show.data, status=status.HTTP_202_ACCEPTED)
        # Exception for when we pass the wrong type for a field
        except AssertionError as e:
            return Response({"detail": str(e)}, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # any other exception
        except:
            res = {
                "detail": "Unprocessable Entity"
            }
            return Response(res, status=status.HTTP_422_UNPROCESSABLE_ENTITY)

    def delete(self, _request, pk):
        show_to_delete = self.get_show(pk=pk)
        show_to_delete.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

## ADDING A SECOND APP

1. Let's add a `genres` app to our project.

```bash
django-admin startapp genres
```

2. Register the genres app in our `project/settings.py`

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',  # register rest framework above our own apps
    'albums',  # we add the name of the app here so the project knows it exists
    'genres',
]
```

3. Go to `genres/models.py` and create a model

```py
from django.db import models


class Genre(models.Model):
    name = models.CharField(max_length=25)

    def __str__(self):
        return f"{self.name}"
```

4. Now let's register the model in the admin app. Go to `genres/admin.py`

```py
from django.contrib import admin
from .models import Genre

admin.site.register(Genre)
```

5. Now let's make some serializers. We are going to need 2 - one as a common serializer and one to populate the albums. In the genres app, create a new folder called serializers and in there, create a new file called `common.py`

```bash
mkdir serializers && cd serializers && touch common.py
```

6. In the `common.py` file, create a serializer for the genres

```py
from rest_framework import serializers # import serializers
from ..models import Genre # import the model (note the 2 dots on this to go up a level in the tree)

class GenreSerializer(serializers.ModelSerializer):

    class Meta: # determines the shape of our JSON
        model = Genre # name of the model it needs to make some JSON from
        # fields = ('id', 'title') # <- can specify specific fields with a tuple
        fields = '__all__'
```

7. Now let's make a `PopulatedGenreSerializer`. Create a new file in the serializers directory called `populated.py` We will import the AlbumSerializer and use it to populate a key called albums.

```py
from .common import GenreSerializer
from albums.serializers import AlbumSerializer


class PopulatedGenreSerializer(GenreSerializer):

    albums = AlbumSerializer(many=True)
```

8. HOLD UP! We have a change to make. We need to update the albums model to have a genres key. It's also going to be what is known as a `Foreign Key` and it's going to relate to the `albums` key that we have on the `PopulatedGenreSerializer`. Update the albums model

```py
class Album(models.Model):  # the Album class is inheriting properties from models.Model
    # this creates the title column and says the data type is a Char(50)
    title = models.CharField(max_length=50)
    artist = models.CharField(max_length=50)
    cover_image = models.CharField(max_length=300)
    genres = models.ManyToManyField('genres.Genre', related_name="albums") # <- THIS RELATED NAME IS THE KEY

    # this function represents the class objects as a string in the admin app
    def __str__(self):
        return f"{self.title} - {self.artist}"
```

8. Now let's create some views for the genres. Go to `genres/views.py`

```py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .serializers.populated import PopulatedGenreSerializer
from .models import Genre


class GenreListView(APIView):

    def get(self, _request):
        genres = Genre.objects.all()
        serialized_genres = PopulatedGenreSerializer(genres, many=True)
        return Response(serialized_genres.data, status=status.HTTP_200_OK)
```

9. And now let's create the `urls.py` file

```py
from django.urls import path
from .views import GenreListView

urlpatterns = [
    path('', GenreListView.as_view())
]
```

10. Finally, let's add the genres urls to the `project/urls.py`

```py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/albums/', include('albums.urls')),
    path('api/genres/', include('genres.urls'))
]
```
