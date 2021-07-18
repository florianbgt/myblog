---
title: Django Oauth2 server with separate resource server (Part 2)
description: Django Oauth 2 setup using Oauth toolkit library with separate resource server. Part 2, resource server
image: "django.png"
writtenBy: "Florian Bigot"
---

In this article, we will setup an Oauth2 authenticartion server with Django.

This server will be a REST API usable by other services for authentication using email and password.

This is the part 2 of the tutorial where we will setup the resource server

Part 1 here: https://blog.florianbgt.com/Django_Oauth2_toolkit_Part1_Auth_server

You can find the source code on my github: https://github.com/florianbgt/django-rest-oauth2-resourceserver

## 1) Setting up our project

In this section, we are going to setup our project the same way we did in the [Part1](https://blog.florianbgt.com/Django_Oauth2_toolkit_Part1_Auth_server) of this article

If you need more details about how things work, do not hesitate to check the [Part1](https://blog.florianbgt.com/Django_Oauth2_toolkit_Part1_Auth_server)

Here is the code below:

```bash
mkdir Django-rest-oauth2-resource-server
cd Django-rest-oauth2-resource-server
python -m venv env
### if you are using Windows
env\scripts\activate
### if you are using Mac or Linux
source env/bin/activate
```

```bash
pip install django
django-admin startproject _project .
```

```bash
python manage.py startapp users
touch users/managers.py
touch users/forms.py
```

```python
### users/managers.py
from django.contrib.auth.base_user import BaseUserManager       #new


class CustomUserManager(BaseUserManager):       #new
    def create_user(self, email, password, **extra_fields):       #new
        if not email:       #new
            raise ValueError('The Email must be set')       #new
        email = self.normalize_email(email)       #new
        user = self.model(email=email, **extra_fields)       #new
        user.set_password(password)       #new
        user.save()       #new
        return user       #new

    def create_superuser(self, email, password, **extra_fields):       #new
        extra_fields.setdefault('is_staff', True)       #new
        extra_fields.setdefault('is_superuser', True)       #new
        extra_fields.setdefault('is_active', True)       #new
        if extra_fields.get('is_staff') is not True:       #new
            raise ValueError('Superuser must have is_staff=True.')       #new
        if extra_fields.get('is_superuser') is not True:       #new
            raise ValueError('Superuser must have is_superuser=True.')       #new
        return self.create_user(email, password, **extra_fields)       #new
```

```python
### users/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser     #new
from .managers import CustomUserManager     #new


class CustomUser(AbstractUser):     #new
    username = None     #new
    first_name = None       #new
    last_name = None        #new
    email = models.EmailField(max_length=50, unique=True)     #new

    USERNAME_FIELD = 'email'     #new

    objects = CustomUserManager()       #new

    def __str__(self):      #new
        return self.email       #new
```

```python
### users/forms.py
from django.contrib.auth.forms import UserCreationForm, UserChangeForm      #new
from .models import CustomUser      #new


class CustomUserCreationForm(UserCreationForm):     #new
    class Meta:     #new
        model = CustomUser      #new
        fields = ('email',)     #new


class CustomUserChangeForm(UserChangeForm):     #new
    class Meta:     #new
        model = CustomUser      #new
        fields = ('email',)     #new
```

```python
### users/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin     #new

from .forms import CustomUserCreationForm, CustomUserChangeForm     #new
from .models import CustomUser     #new


class CustomUserAdmin(UserAdmin):     #new
    add_form = CustomUserCreationForm     #new
    form = CustomUserChangeForm     #new
    model = CustomUser     #new
    list_display = ('email', 'is_staff', 'is_active',)     #new
    list_filter = ('email', 'is_staff', 'is_active',)     #new
    fieldsets = (     #new
        ('Credentials', {'fields': ('email', 'password')}),     #new
        ('Permissions', {'fields': ('is_staff', 'is_active', 'groups')}),     #new
    )     #new
    add_fieldsets = (     #new
        (None, {     #new
            'classes': ('wide',),     #new
            'fields': ('email', 'password1', 'password2', 'is_staff', 'is_active')}     #new
        ),     #new
    )     #new
    search_fields = ('email',)     #new
    ordering = ('email',)     #new


admin.site.register(CustomUser, CustomUserAdmin)     #new
```

```python
### _project/settings.py
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    #local
    'users',        #new
]

AUTH_USER_MODEL = 'users.CustomUser'        #new
...
```

```python
### _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('users.urls')),       #new
]
```

```bash
python manage.py makemigrations users
python manage.py migrate
python manage.py createsuperuser
Email: admin@email.com
Password: testpass123
Password (again): testpass123
```

## 2) Setting up Oauth2 using Django Oauth Toolkit

The setup is going to be similar to the authentication server ([Part1](https://blog.florianbgt.com/Django_Oauth2_toolkit_Part1_Auth_server) of this article).

```bash
pip install djangorestframework django-oauth-toolkit
```

The only difference are the introspection settings (`RESOURCE_SERVER_INTROSPECTION_URL` and `RESOURCE_SERVER_INTROSPECTION_CREDENTIALS`). These 2 settings will allow our resource server to communicate with our authorization server to get information about our users

In `RESOURCE_SERVER_INTROSPECTION_CREDENTIALS`, you need to put the client Id and client Secret we created in the Part1 of this article

Because of this communication between our 2 server, we do not need (or want in this example) to add Oauth Toolkit in our `_project/urls.py`

```python
### _projects/settings.py
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    #local
    'users',     #new
    #3rd party
    'rest_framework',     #new
    'oauth2_provider',      #new
]

AUTH_USER_MODEL = 'users.CustomUser'        #new

OAUTH2_PROVIDER = {     #new
    'SCOPES': {     #new
        'read': 'Read scope',       #new
        'write': 'Write scope',     #new
    },       #new
    'RESOURCE_SERVER_INTROSPECTION_URL': 'http://localhost:8000/o/introspect/',     #new
    'RESOURCE_SERVER_INTROSPECTION_CREDENTIALS': ('rs-id','rs-secret'),     #new
}       #new

REST_FRAMEWORK = {      #new
    'DEFAULT_AUTHENTICATION_CLASSES': (     #new
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',     #new
        'rest_framework.authentication.SessionAuthentication',
    ),     #new
    'DEFAULT_PERMISSION_CLASSES': (      #new
        'rest_framework.permissions.IsAuthenticated',      #new
    )      #new
}      #new
...
```

## 3) Setup the user API views

In this section, we will setup views to allow new user to signup, login and change their password.

The idea here is not to have the user directly interact with our authentication server. We will for that setup API view that will forward the request from the resource server to the authentication server:

```bash
touch users/serializers.py
touch users/urls.py
```

We first start to create 3 simple serializer for each of out views:
```python
### users/serializers.py
from rest_framework import serializers      #new


class SignUpSerializer(serializers.Serializer):      #new
    email = serializers.EmailField()      #new
    password = serializers.CharField()      #new
    password2 = serializers.CharField()      #new


class TokenSerializer(serializers.Serializer):      #new
    email = serializers.EmailField()      #new
    password = serializers.CharField()      #new


class RefreshTokenSerializer(serializers.Serializer):      #new
    refresh = serializers.EmailField()      #new


class PasswordChangeSerializer(serializers.Serializer):      #new
    old_password = serializers.CharField()      #new
    password = serializers.CharField()      #new
    password2 = serializers.CharField()      #new
```

Then, we setup our views:
- For the signup view, we send a http request to our authentication server with the new user information. We then forward back the response from the authentication server to the user
- We do the same thing for the login view (GetTokenView) and refresh view (RefreshTokenView). That also allow us to keep the client id and client secret we created in the Part 1 of this article secret!
- Same thing in the password change view. But here, we also pass the authorization header to the authentication server. That will allow the authentication server to identify the user

```python
### users/views.py
from rest_framework.views import APIView        #new
from rest_framework.response import Response        #new
from rest_framework.permissions import AllowAny        #new

from .serializers import SignUpSerializer, TokenSerializer, RefreshTokenSerializer, PasswordChangeSerializer        #new

import requests        #new


class SignUpView(APIView):        #new
    permission_classes = [AllowAny]        #new
    serializer_class = SignUpSerializer        #new
    def post(self, request):        #new
        response = requests.post(        #new
            'http://localhost:8000/users/signup/',        #new
            data={        #new
                'email': request.data['email'],        #new
                'password': request.data['password'],        #new
                'password2': request.data['password2']        #new
            }        #new
        )        #new
        return Response(response.json())        #new

class GetTokenView(APIView):        #new
    permission_classes = [AllowAny]        #new
    serializer_class = TokenSerializer        #new
    def post(self, request):        #new
        response = requests.post(        #new
            'http://localhost:8000/o/token/',        #new
            data={        #new
                'grant_type': 'password',        #new
                'username': request.data['email'],        #new
                'password': request.data['password'],        #new
                'client_id': 'rs-id',        #new
                'client_secret': 'rs-secret'        #new
            }        #new
        )        #new
        return Response(response.json())        #new


class RefreshTokenView(APIView):        #new
    permission_classes = [AllowAny]        #new
    serializer_class = RefreshTokenSerializer        #new
    def post(self, request):        #new
        response = requests.post(        #new
            'http://localhost:8000/o/token/',        #new
            data={        #new
                'grant_type': 'refresh_token',        #new
                'refresh_token': request.data['refresh'],        #new
                'client_id': 'rs-id',        #new
                'client_secret': 'rs-secret'        #new
            }        #new
        )        #new
        return Response(response.json())        #new


class PasswordChangeView(APIView):        #new
    serializer_class = PasswordChangeSerializer        #new
    def put(self, request):        #new
        response = requests.put(        #new
            'http://localhost:8000/users/password/',        #new
            headers={        #new
                'Authorization': request.META.get('HTTP_AUTHORIZATION')        #new
            },        #new
            data={        #new
                'old_password': request.data['old_password'],        #new
                'password': request.data['password'],        #new
                'password2': request.data['password2']        #new
            }        #new
        )        #new
        return Response(response.json())        #new
```

Finally, setup our routes:

```python
### users/urls.py
from django.urls import path        #new
from .views import SignUpView, GetTokenView, RefreshTokenView ,PasswordChangeView        #new


urlpatterns = [        #new
    path('signup/', SignUpView.as_view()),        #new
    path('token/', GetTokenView.as_view()),        #new
    path('token/refresh/', RefreshTokenView.as_view()),        #new
    path('password/', PasswordChangeView.as_view())        #new
]        #new
```

```python
## _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('users.urls')),       #new
]
```

We can now test our views using postman. For this, we run both the autentication server and the resource server. The authentication is ran on port 8000 and the resource server on port 8001:

```python
python manage.py runserver localhost:8000       ### auth server
python manage.py runserver localhost:8001       ### resource server
```

Let's now create a new user from our resource server using the following http request with postman:

<div><blog-img src="postman_signup.png" alt="Postman http request to signup a new user" width="100%" height="auto" class="shadow mb-3"/></div>

We now authenticate that user from the resource server as well:

<div><blog-img src="postman_get_token.png" alt="Postman http request to get token of the new user" width="100%" height="auto" class="shadow mb-3"/></div>

We finally change the user's password from the resource as well using this http request. Here, we need to attached the access token in a authorization header (type Bearer):

<div><blog-img src="postman_password_change.png" alt="Postman http request to change users's password" width="100%" height="auto" class="shadow mb-3"/></div>

In the background, our resource server will check in its database if a user exist and a valid access token exists:
- If it does, the user will be granted access
- If it does not, the resource server will check if the user exists and if a valid access token exists in the authentication server. If it does, user will be granted access
- If none of the 2 conditions above are met, the user will be denied access

Some time has passed and our access token has expired. We can also renew it using this http request:

<div><blog-img src="postman_refresh_token.png" alt="Postman http request to get refresh the user's access token" width="100%" height="auto" class="shadow mb-3"/></div>

## 4) Create a dummy item API

In this section, we will setup a dummy item API. That will allow us to test the whole Oauth2 setup in a realistic environement

This API will use generics django rest views:

```bash
python startapp items
touch items/urls.py
touch items/serializers.py
```

```python
### items/models.py
from django.db import models
from django.contrib.auth import get_user_model        #new

User = get_user_model()        #new


class Item(models.Model):        #new
    name = models.CharField(max_length=50, unique=True)        #new
    description = models.TextField(max_length=250)        #new
    price = models.DecimalField(max_digits=14, decimal_places=2)        #new
    user = models.ForeignKey(User, on_delete=models.CASCADE)        #new

    def __str__(self):        #new
        return self.name        #new
```

```python
### items/admin.py
from django.contrib import admin        #new

from .models import Item        #new


class ItemAdmin(admin.ModelAdmin):        #new
    pass        #new

admin.site.register(Item, ItemAdmin)        #new
```

```python
### items/serializers.py
from rest_framework import serializers
from rest_framework.fields import ReadOnlyField      #new
from django.contrib.auth import get_user_model      #new

from .models import Item      #new

User = get_user_model()      #new


class UserSerializer(serializers.ModelSerializer):      #new
    class Meta:      #new
        model = User      #new
        fields = ['id', 'email']      #new


class ItemSerializer(serializers.ModelSerializer):        #new
    user = UserSerializer(read_only=True)      #new

    class Meta:        #new
        model = Item        #new
        fields = ['id', 'name', 'description', 'price', 'user',]        #new

    def create(self, validated_data):      #new
        item = Item.objects.create(      #new
            name=validated_data['name'],      #new
            description=validated_data['description'],      #new
            price = validated_data['price'],      #new
            user = self.context['request'].user      #new
        )      #new
        return item      #new
```

```python
### items/views.py
from rest_framework import generics     #new

from .models import Item     #new
from .serializers import ItemSerializer     #new


class ItemList(generics.ListCreateAPIView):     #new
    queryset = Item.objects.all()     #new
    serializer_class = ItemSerializer     #new


class ItemDetail(generics.RetrieveUpdateDestroyAPIView):     #new
    queryset = Item.objects.all()     #new
    serializer_class = ItemSerializer     #new
```

```python
### items/urls.py
from django.urls import path        #new
from .views import ItemList, ItemDetail        #new


urlpatterns = [        #new
    path('', ItemList.as_view()),        #new
    path('<pk>/', ItemDetail.as_view()),        #new
]        #new
```

```python
### _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('users.urls')),       #new
    path('items/', include('items.urls')),        #new
]
```

```python
### _project/settings.py
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    #local
    'users',     #new
    'items',     #new
    #3rd party
    'rest_framework',     #new
    'oauth2_provider',      #new
]
...
```

```bash
python manage.py makemigrations items
python manage.py migrate
```

We can now create new items using the following http request. Here, we need to attached the access token in a authorization header (type Bearer):

<div><blog-img src="postman_item_creation.png" alt="Postman http request to create an item" width="100%" height="auto" class="shadow mb-3"/></div>

We also can list, edit and delete new items according to the views we have setup

## Clonclusion

We have now have a separate Oauth2 resource server setup!

We can create many other resource server using this setup while having one centralized authentication server which create a great user experience

You can find the source code of this article on my github: https://github.com/florianbgt/django-rest-oauth2-resourceserver

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com