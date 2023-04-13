# CUSTOM USERS AND AUTHENTICATION

We want to use jwt tokens for auth again. Django isn't set up to do this out of the box so it will take a bit of work, but after implementing it, we will have registration, login and secure route functionality like we had in our previous node applications.

1. Firstly, we need to start a new app called `jwt_auth`

```sh
django-admin startapp jwt_auth
```

2. Next, we need to register our `jwt_auth` app in our projects settings.

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
    'comments',
    'artists',
    'jwt_auth',
]
```

3. Django already has a user model, it's what it uses under the hood to add a superuser. In the project settings.py, add the below to specifiy which model we intend to use: AUTH_USER_MODEL = 'jwt_auth.User'

```py
AUTH_USER_MODEL = 'jwt_auth.User' # This will point to our User model
```

You can put this line anywhere in the file, but I like to put it after the DATABASES list and before the AUTH_PASSWORD_VALIDATORS dict. That way, it keeps the users stuff together.

4. in jwt_auth/models.py we'll add our new model.

Django already has a user model called `AbstractUser` which we want to extend. To add new fields and override existing ones.

- Django already has password, password confirmation & username so we don't need to add them.
- It also doesnt actually make email required so we want to override that.
- By defining these fields and not adding null=True or blank=True we make them required.

```py
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    email = models.CharField(max_length=50, unique=True)
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    profile_image = models.CharField(max_length=300)
```

5. Now we have a new model we need to make some migrations and migrate. **BUT** there will be an error when we migrate.

Django will get confused here because a user table exist,s but we've just created a new model. The migration history will be all over the place so we want to reset the db so no migrations exist. We are only having to do this as we're doing auth last. During project and when we build out our apps, we would most likely build auth first, but it doesn't make sense to learn it first.

Before we makemigrations and migrate, we are going to backup our data.

6. Create backups in seed files:

Before we dump our database and lose all the data we have created, we can tell Django to create seed files using the data in our tables. We need to do one for each app, but we can run them concurrently.

```sh
python manage.py dumpdata albums --output albums/seeds.json --indent=2;
python manage.py dumpdata artists --output artists/seeds.json --indent=2;
python manage.py dumpdata genres --output genres/seeds.json --indent=2;
python manage.py dumpdata comments --output comments/seeds.json --indent=2;
```

You'll see that the data is now in a file called seeds.json in each app. We'll use these files later to populate our database tables.

7. Drop database:

```sh
dropdb django-albums
```

8. Delete migration files

**Remember** we want delete all of the migration files except the \***\*init**.py\*\* file.

9. Recreate db:

```sh
createdb django-albums
```

10. Now we can make and apply our migrations

```sh
python manage.py makemigrations
```

```sh
python manage.py migrate
```

11. Create superuser:

```sh
python manage.py createsuperuser
```

11. Register User model on admin:

get_user_model is a method that when invoked returns whatever model that our app is set up to use. In settings.py we specified that we will use our own custom model 'jwt_auth.User' so this is what will be returned.

```py
from django.contrib import admin
from django.contrib.auth import get_user_model

User = get_user_model()
admin.site.register(User) # then we'll register this to the admin as usual
```

# CREATING USERS THROUGH THE API

12. To handle jwt tokens in our API, we are going to install a package called `pyjwt`

```sh
pipenv install pyjwt
```

13. We are going to create a file to handle our tokens for login/register. Let's create a new file called `authentication.py` in our `jwt_auth` app.

The steps to implement tokens is very similar to what we did before in our node applications, but obviously this time we are going to write it in python.

```py
from rest_framework.authentication import BasicAuthentication # Class to inherit that has pre-defined validations
from rest_framework.exceptions import PermissionDenied # throws an exception
from django.contrib.auth import get_user_model # method that returns the current auth model
from django.conf import settings # import settings so we can get secret key (we'll come back to this)
import jwt # import jwt so we can decode the token in the auth header

User = get_user_model() # saving auth model to a variable

class JWTAuthentication(BasicAuthentication):

    # This will act as the middleware that authenticates our secure routes
    def authenticate(self, request):
        # Get the Authorization header from the incoming request object and save it to a variable.
        auth_header = request.headers.get('Authorization')

        # Check if header has a value. If it doesn't, return None.
        if not auth_header:
            return None

        # Check that the token starts with Bearer
        if not auth_header.startswith('Bearer'):
            raise PermissionDenied(detail="Invalid Auth Token Format")

        # remove Bearer from beginning of Authorization header
        token = auth_header.replace('Bearer ', '')

        # Get payload, take the sub (the user id) and make sure that user exists
        try:
            # 1st arg is the token itself
            # 2nd arg is the secret
            # 3rd argument is kwarg that takes the algorithm used
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])

            # find user
            user = User.objects.get(pk=payload.get('sub'))

        # if jwt.decode errors, this except will catch it
        except jwt.exceptions.InvalidTokenError:
            raise PermissionDenied(detail='Invalid Token')

        # If no user is found in the db matching the sub, the below will catch it
        except User.DoesNotExist:
            raise PermissionDenied(detail='User Not Found')

        # If all good, return the user and the token
        return (user, token)
```

14. We now need to add REST_FRAMEWORK into settings.py.

The first part is telling Django to render in JSON, although the serializers are doing this for us we can confirm this behaviour here.

The second part is telling rest_framework and django that we are using the JWTAuthentication class we just created as the default.

```py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'jwt_auth.authentication.JWTAuthentication'
    ]
}
```

# RBAC (ROLE BASED ACCESS CONTROL)

15. Next we'll control who see's what. Go to `albums/views.py` and add imports for permissions from rest_frameworks.permissions.

We are going to import something called `IsAuthenticatedOrReadOnly` which has a method that enforces every method except GET to throw a permissions error.

_there's another one too, IsAuthenticated, that applies to all methods_

Add the snippet below to the `views.py` imports:

```py
from rest_framework.permissions import IsAuthenticatedOrReadOnly # IsAuthenticatedOrReadOnly specifies that a view is secure on all methods except get requests
```

16. inside the AlbumListView class add the following line. \*\*it shouldn't be in a method, so it's easier to put it as the first line of the class so it looks like this:

```py
class AlbumListView(APIView):
    permission_classes = (IsAuthenticatedOrReadOnly, ) # sets the permission levels of the specific view by passing in the rest framework authentication class. Needs to be a tupple with the trailing comma.

    def get(self, _request):
        albums = Album.objects.all()
    # .... the rest of the file is omitted as this is just for demo purposes of where to put the permissions classes line
```

17. The serializer for the user:

We're going to create our serializer but add a new function called `validate` to run our validation checks.

Create a directory called `serializers` and in there, create a file called `common.py`. we're going to put this code into the common.py file:

```py
from rest_framework import serializers
# password_validation is the same method being used to check our password is valid when creating a superuser
from django.contrib.auth import get_user_model, password_validation
from django.contrib.auth.hashers import make_password  # hashes our password for us
from django.core.exceptions import ValidationError

User = get_user_model()  # this is our user model


class UserSerializer(serializers.ModelSerializer):
    # when User is being converted back to JSON to return data to user, password & confirmation are not going to be returned
    password = serializers.CharField(write_only=True)
    password_confirmation = serializers.CharField(write_only=True)

    # validate function is going to:
    # check our passwords match
    # hash our passwords
    # update password on data object that is passed through from the request in the views
    def validate(self, data):
        # remove the fields from the request aand save to vars
        password = data.pop('password')
        password_confirmation = data.pop('password_confirmation')

        # check if the passwords match
        if password != password_confirmation:
            raise ValidationError(
                {'password_confirmation': 'Does not match password field'})

        # We're going to first make sure the password is valid
        try:
            password_validation.validate_password(password=password)
        except ValidationError as err:
            raise ValidationError({'password': err.messages})

        # reassign the value of data.password as the hashed password we create
        # make_password hashes a plain text string and returns it
        data['password'] = make_password(password)

        return data  # returns updated data dictionary

    class Meta:
        model = User
        fields = '__all__'
```

18. Now let's sort out the `jwt_auth/views.py`

```py
from rest_framework.views import APIView  # main API controller class
# response class, like res object in express
from rest_framework.response import Response
from rest_framework import status
from rest_framework.exceptions import PermissionDenied
from datetime import datetime, timedelta  # creates timestamps in dif formats
from django.contrib.auth import get_user_model  # gets user model we are using
from django.conf import settings  # import our settings for our secret
import jwt  # import jwt

from .serializers.common import UserSerializer
User = get_user_model()  # Save user model to User var


class RegisterView(APIView):

    def post(self, request):
        user_to_create = UserSerializer(data=request.data)
        print('USER CREATE', user_to_create)
        if user_to_create.is_valid():
            user_to_create.save()
            return Response({'message': 'Registration successful'}, status=status.HTTP_202_ACCEPTED)
        return Response(user_to_create.errors, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
```

19. Create the `jwt_auth/urls.py` and add the urls:

```py
from django.urls import path
from .views import RegisterView

urlpatterns = [
path('register/', RegisterView.as_view())
]
```

20. Add the auth urls to the project urls

```py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/albums/', include('albums.urls')),
    path('api/genres/', include('genres.urls')),
    path('api/comments/', include('comments.urls')),
    path('api/artists/', include('artists.urls')),
    path('api/auth/', include('jwt_auth.urls')),
]
```

21. Make a register request in postman. The endpoint will be `/auth/register`.

Try to make a user with a short password. You'll see that Django has validation for passwords which is coming from this line in our UserSerializer:

```py
password_validation.validate_password(password=password)
```

22. Now let's create a login view. Go to `jwt_auth/views.py`

We're going to create a post request that:

- Checks user exists
- Checks passwords match
- Creates expiry for token using timestamps
- Creates jwt token
- Returns token back to user

```py
class LoginView(APIView):

    def post(self, request):
        # get data from the request
        email = request.data.get('email')
        password = request.data.get('password')
        try:
            user_to_login = User.objects.get(
                email=email)  # get user with email
        except User.DoesNotExist:
            raise PermissionDenied(detail='Invalid Credentials')  # throw error
        if not user_to_login.check_password(password):
            raise PermissionDenied(detail='Invalid Credentials')

        # timedelta can be used to calculate the difference between dates - passing 7 days gives you 7 days represented as a date that we can add to datetime.now() to get the date 7 days from now
        dt = datetime.now() + timedelta(days=7)  # validity of token
        token = jwt.encode(
            # strftime -> string from time and turning it into a number
            {'sub': user_to_login.id, 'exp': int(dt.strftime('%s'))},
            settings.SECRET_KEY,
            algorithm='HS256'
        )
        return Response({'token': token, 'message': f"Welcome back {user_to_login.username}"})
```

23. Add the path to `jwt_auth/urls/py`

```py
path('login/', LoginView.as_view())
```

24. Add an album in postman and you should get back an authentication error:

```json
{
  "detail": "Authentication credentials were not provided."
}
```

# RELATIONSHIPS

24. relationship between user and comments:

The relationship between a user and a comment is `one to many`, meaning a user can have many comments, but a comment can only have one owner.

Let's implement this relationship.

Go to `comments/models.py` and let's add a new key.

```py
owner = models.ForeignKey( # if you call it user, I think it can clash with django fields so I tend to use owner
"jwt_auth.User",
related_name="comments",
on_delete=models.CASCADE
)
```

25. makemigrations. You're going to get a prompt because existing reviews don't have an owner. It'll look like this:

```sh
It is impossible to add a non-nullable field 'owner' to comment without specifying a default. This is because the database needs something to populate existing rows.
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Quit and manually define a default value in models.py.
Select an option:
```

Let's select option 1 - Provide a one-off default now.

It'll now prompt you to give a default value.

```sh
Please enter the default value as valid Python.
The datetime and django.utils.timezone modules are available, so it is possible to provide e.g. timezone.now as a value.
Type 'exit' to exit this prompt
```

We need to pass it a user id, so as long as we created a super user earlier, we have a user id which will be 1. Enter 1 into the prompt and press return. You should see it complete making the migrations for comments.

Now you can migrate!

26. Test it in postman - you should be able to perform a GET request to an album that has comments. If all is well, you'll see that the comment has am `owner` key with the value of 1.

We currently have to pass the owner into the comment which won't work when we are sending from React as the user won't know their id, so it's going to have to come from the view (a little bit like how we add things to the user object in our secure route in node apps)

27. Adding auth to the comments view. We want the owner to come from the token so we need to add authentication

Go to `comments/views.py` and add

```py
from rest_framework.permissions import IsAuthenticated
```

Then add the permissions to the CommentListView:

```py
permission_classes = (IsAuthenticated,) # We use IsAuthenticated because it secures all routes and we only have a post
```

28. On the post method in CommentListView update the owner on the request data. This will be updated to be the user id passed through from the JWTAuthentication middleware:

```py
def post(self, request):
    # this will add the "owner" onto the request data
    request.data["owner"] = request.user.id
    comment_to_create = CommentSerializer(data=request.data)
    try:
        comment_to_create.is_valid()
        comment_to_create.save()
```

29. Now let's handle the update and delete methods on CommentDetailView to check that it's the owner who is making the request, again checking the owner on the model against the request.user passed from JWTAuthentication. We will also need to import the PermissionDenied exception from the same package we import NotFound from (rest_framework.exceptions).

```py
class CommentDetailView(APIView):

    def delete(self, _request, pk):
        try:
            comment_to_delete = Comment.objects.get(pk=pk)
            # we add the check here
            if comment_to_delete.owner != request.user:
                raise PermissionDenied()
            comment_to_delete.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)
        except Comment.DoesNotExist:
            raise NotFound(detail="Comment not found")
```

30. We now have a fully working API with authentication and secure routes!

Go ahead and spend a few minutes creating data through the admin app and postman, so you have a database full of data made by admin and API users. Make sure you:

- add genres (as admin from the admin app)
- add albums as admin from admin app and as a user from the API
- add comments to albums as admin from the admin app and as a user from the API

31. Let's now take the data and re-create those seed files, this time with the users.

```sh
python manage.py dumpdata albums --output albums/seeds.json --indent=2;
python manage.py dumpdata artists --output artists/seeds.json --indent=2;
python manage.py dumpdata genres --output genres/seeds.json --indent=2;
python manage.py dumpdata comments --output comments/seeds.json --indent=2;
python manage.py dumpdata jwt_auth --output jwt_auth/seeds.json --indent=2;
```

# NESTED SERIALIZERS - POPULATING THE OWNER IN A COMMENT

31. We want to see the actual owner of the comments on the album detail serializer, but if we look at the `PopulatedAlbumSerializer`, we can see that the comments come from the `CommentSerializer`. We need to make a nested serializer.

from jwt_auth.serializers.common import UserSerializer
from .common import CommentSerializer

```py
class PopulatedCommentSerializer(CommentSerializer):
    owner = UserSerializer()
```

32. Update the `PopulatedAlbumSerializer` to use the `PopulatedCommentSerializer` instead of the common `CommentSerializer`

33. You can see that the comment owner key is now populated, but it's also returning data that we don't really want to return. Let's update the UserSerializer in `jwt_auth/serializers/common.py` let's update the last line and specify the fields that we want to return, instead of '**all**'.

# update **all** to:

```py
fields = ('id', 'email', 'username', 'first_name', 'last_name', 'profile_image', 'password', 'password_confirmation')
```

by doing this, we can see that the owner key on a comment is now limited to:

```json
"owner": {
    "id": 2,
    "email": "tristan@tristan.com",
    "username": "tristan_hall",
    "first_name": "Tristan",
    "last_name": "Hall",
    "profile_image": "tristansface.jpg"
},
```

34. If we want to add an owner to an album, we need to update the album model. Go to `albums/models.py` and add a new key to the model.

```py
owner = models.ForeignKey(
    'jwt_auth.User',
    related_name='albums',
    on_delete=models.CASCADE
)
```

35. Again, we've made a change to a model so we need to makemigrations and migrate. And again, we will get the same prompt to add a default value. We can provide any valid user id. I'm going to use admin, which would be id 1.

36. Lets now add the owner from the request onto the data that we will hand to the serializer in the view. Go to `albums/views.py` and update the post method in the AlbumListView to include the following line:

```py
def post(self, request):
    request.data['owner'] = request.user.id # <- this is the line we need to add
    print(request.data)
    album_to_add = AlbumSerializer(data=request.data)
    try:
        album_to_add.is_valid()
```

37. Now we need to update the PopulatedAlbumSerializer. Go to `albums/serializers/populated.py`, import the `UserSerializer` and add the owner key.

```py
from rest_framework import serializers
from .models import Album
from genres.serializers.common import GenreSerializer
from comments.serializers.populated import PopulatedCommentSerializer
from artists.serializers.common import ArtistSerializer
from jwt_auth.serializers.common import UserSerializer # import the UserSerializer


class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        model = Album  # the model that our serializer should use to transform from the database
        fields = '__all__'  # specify which fields we want to return


class PopulatedAlbumSerializer(AlbumSerializer):
    genres = GenreSerializer(many=True)
    comments = PopulatedCommentSerializer(many=True)
    artist = ArtistSerializer()
    owner = UserSerializer() # We populate the owner key with the UserSerializer
```

# SEEDING THE DATABASE IN DJANGO

Django provides us with a `loaddata` function within `manage.py`. We can execute this function and pass a seed file as an argument. For example, if we wanted to seed our `jwt_auth` app then we could run:

```sh
python manage.py loaddata jwt_auth/seeds.json
```

we would need to run this command for each app that we want to seed. This is a bit repetitive so fortunately for us, we can write a custom script to execute commands for us. Let's take a quick look at scripting.

# SCRIPTING IN BASH

Bash scripts are very useful for taking a set of repetitive commands and writing them into a small self contained program at will execute the commands for us when we trigger the script.

We are going to make 2 scripts:

- the first will be to dump the data into seeds files
- the second will be to seed the database

1. Create a file in the root of the project called `makeseeds.sh` and populate the file with the following:

```sh
#!/bin/bash

echo "creating albums/seeds.json"
python manage.py dumpdata albums --output albums/seeds.json --indent=2;

echo "creating artists/seeds.json"
python manage.py dumpdata artists --output artists/seeds.json --indent=2;

echo "creating genres/seeds.json"
python manage.py dumpdata genres --output genres/seeds.json --indent=2;

echo "creating comments/seeds.json"
python manage.py dumpdata comments --output comments/seeds.json --indent=2;

echo "creating jwt_auth/seeds.json"
python manage.py dumpdata jwt_auth --output jwt_auth/seeds.json --indent=2;
```

you can see that this is simply the commands that we would run to dump the data from each app into a seeds file.

2. Create a file called seed.sh and populate the file with the following:

```sh
#!/bin/bash

echo "dropping database django-albums"
dropdb django-albums

echo "creating database django-albums"
createdb django-albums

python manage.py makemigrations

python manage.py migrate

echo "inserting users"
python manage.py loaddata jwt_auth/seeds.json

echo "inserting genres"
python manage.py loaddata genres/seeds.json

echo "inserting artists"
python manage.py loaddata artists/seeds.json

echo "inserting albums"
python manage.py loaddata albums/seeds.json

echo "inserting comments"
python manage.py loaddata comments/seeds.json
```

Again, you can see that this is simply using the `loaddata` function from manage.py to seed each apps database tables.

3. We need to make the files _executable_ so in the terminal, run:

```sh
chmod u+x ./seed.sh;
chmod u+x ./makeseeds.sh
```

4. Now, whenever you want to run the makeseeds command, just run `./makeseeds.sh` from the root of the project. You can then drop the database, recreate it, run the migrations and then when the database is ready to be loaded up with the data, you can simply run `./seed.sh`

# SCRIPTING POSTMAN

Postman is great for testing our APIs, but constantly having to copy/paste the auth token into requests is a bit of a pain.

We can actually write a script that will trigger before a request, which we can use to get a token and then set it to an environment variable. Then we can update the requests to use the token in the environment and we will never have to copy/paste the tokens!

In your environment, create a new variable called `AUTH_TOKEN` and set its initial value to `initValue`.

Now, go to your collection and click on the 3 dots in the collections menu. Select edit and then go to the tab for `Pre-request Script`.

Paste this code in:

```js
const loginRequest = {
  url: pm.environment.get('BASE_URL') + '/auth/login/',
  method: 'POST',
  timeout: 0,
  header: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: {
    mode: 'urlencoded',
    urlencoded: [
      { key: 'password', value: 'admin' },
      { key: 'email', value: 'admin@admin.com' }
    ]
  }
};

pm.sendRequest(loginRequest, function (err, res) {
  var responseJson = res.json();
  console.log(responseJson);
  pm.environment.set('AUTH_TOKEN', responseJson['token']);
});
```

Now in each request that you need the token, click on the `Authroization` tab, select `Bearer` and put the value as `{{AUTH_TOKEN}}`.
