### SESSION 1 (RELATIONSHIPS EXPLAINED & SQL)

1. Explain what the relationships are - OneToMany and ManyToMany

- Equivalent of Referenced and Embedded models
- OneToMany first
- OneToMany would represent something like comments.
  So a comment can only relate to one show but a
  show can relate to many comments. In this example,
  the show is the one and the comments are the many

2. go to https://app.quickdatabasediagrams.com

- Explain the difference and similarities between embedded and oneToMany
- "So in a relational database, instead of being embedded in a parent
  document, the comments would have their own table and they would have
  a column that relates directly to it's parent, or in this case it's one.
  In non-relational databases this schema would be embedded straight into
  the document itself"
- "You're welcome to use something like this to represent your database
  visually, but you don't have to. Good old fashioned pen and paper is okay"

3. Go into TablePlus and create a new db called cheesebored-relationships,
   then add the following: (GO TO relationships.sql)

### SESSION 2 (DJANGO RELATIONSHIPS - )

1. Create a comments app using `django-admin startapp comments`
2. Add 'comments' to INSTALLED APPS list in settings.py
3. Create model in comments/models.py:

```py
class Comment(models.Model):
    # Explain that it's text as it's a bigger box in admin view
    text = models.TextField(max_length=300)
    created_at = models.DateTimeField(auto_now_add=True)
    album = models.ForeignKey(
        "albums.Album",  # this defines where the relationship is - in the shows app on the Show model
        related_name="comments",  # This is what the column will be called on the show lookup
        # This specifies that the comment should be deleted if the show is deleted
        on_delete=models.CASCADE
    )
```

4. Add this model to admin.py:

```py
from .models import Comment
admin.site.register(Comment)
```

5. Make migrations, then start server:

```bash
python manage.py makemigrations
```

```bash
python manage.py migrate
```

```bash
python manage.py runserver
```

6. Go to TablePlus and switch db to django-albums and view the comments table to show what has been created:

"So we can see that this final column has been created, and this was done for us by django in the model" - album models.py

7. go to http://localhost:8000/admin and add a comment, show that the dropdown has all the shows in it. Then go to TablePlus and view the comment that's just been added.

8. "Next thing we want to do is essentially populate the response that we get when we make requests to return all albums. You might remember when we used .populate() in mongo to get all the owner's info returning instead of just their id, well we're going to be doing something very similar here, just in a slighlty different way with a few more steps. So we need to create a Serializer for our comments."

Let's create a serializers folder and common.py file inside.

```py
from rest_framework import serializers
from ..models import Comment

class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = '__all__'
```

9. In the albums app, let's edit the populated serializer. Import the CommentSerializer and add it to the PopulatedAlbumSerializer.

```py
class PopulatedAlbumSerializer(AlbumSerializer):
    genres = GenreSerializer(many=True)
    comments = CommentSerializer(many=True)
```

As we use the PopulatedAlbumSerializer when we request the AlbumDetailView, we should be able to see our comments populated when we make a request for a single album.

### SESSION 2 CONTINUED...

10. "So now we can see our comments in our /shows get requests, we're going to add the functionality in so we can add the comments. So unlike mongo where we had a really long url, so it was like /cheeses/:id/comments/:id we don't need to go through the shows to get the comments as this is a relational database and they have their own table so that connection is being handled elsewhere. So we just need /comments. So firstly lets add our post route."

11. inside comments app, go to `views.py`:

```py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.db import IntegrityError
from .serializers.common import CommentSerializer


class CommentListView(APIView):

    def post(self, request):
        comment_to_create = CommentSerializer(data=request.data)
        try:
            comment_to_create.is_valid()
            comment_to_create.save()
            return Response(comment_to_create.data, status=status.HTTP_201_CREATED)
        # below is the exception thrown when we miss a required field
        except IntegrityError as e:
            return Response({
                "detail": str(e),
            }, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # AssertionError occurs when the incorrect type is passed as a value for an existing field
        except AssertionError as e:
            return Response({"detail": str(e)}, status=status.HTTP_422_UNPROCESSABLE_ENTITY)
        # catch all exception
        except:
            return Response("Unprocessable Entity", status=status.HTTP_422_UNPROCESSABLE_ENTITY)
```

15. create urls.py in comments folder and add:

```py
from django.urls import path
from .views import CommentListView

urlpatterns = [
path('', CommentListView.as_view())
]
```

16. Add the comments urls to the project urls:

```py
path('api/comments/', include('comments.urls'))
```

17. Go to Insomnia and run the `post` request to api/comments/:

```json
{
  "text": "This is a good album.",
  "album": 1
}
```

You'll see in the terminal that it's been added by making a POST request to /api/comments. The main difference here is that we are passing the id through the body because the relationship between the two tables is in the album_id column in the comments table. We pass the key as album in the body because that's what we called it in the comments model, and django is doing the rest for us by adding \_id to the end when adding to the database. That goes for when it created the table and when a comment is added.

18. Let's work on some more comments functionality (edit and delete)

add another import to the `comments/views.py`

```py
from rest_framework.exceptions import NotFound
```

and then we'll add the CommentDetailView view:

```py
class CommentDetailView(APIView):

    def delete(self, _request, pk):
        try:
            comment_to_delete = Comment.objects.get(pk=pk)
            comment_to_delete.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)
        except Comment.DoesNotExist:
            raise NotFound(detail="Comment not found")
```

19. Let's update the `comments/urls.py` by adding and endpoint for a single comment:

```py
path('<int:pk>/', CommentDetailView.as_view())
```

20. Let's test it in postman.

# MANY TO MANY RELATIONSHIPS

A good example of ManyToMany relationships would be if we had something like events, so users coul ve multiple events, and events can have multiple users. So they're pretty similar to OneToMany, except whereas comments can only have one show, users arent limited to just one event

Unlike OneToMany, ManyToMany have a join table where both the id's are stored, rather than a column in the One side of the relationship.

1. Add categories table in cheesebored-relationships db in TablePlus, then add some data into it:

2. OPEN relationships.py

3. Explain the above = "So we specify the fields we want, and as there's two name columns we rename catgeories.name from name to category.
