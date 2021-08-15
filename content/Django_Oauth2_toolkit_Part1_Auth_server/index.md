---
title: Django Oauth2 server with separate resource server (Part 1)
description: Django Oauth 2 setup using Oauth toolkit library with separate resource server. Part 1, authentication server
image: "django.png"
writtenBy: "Florian Bigot"
---

In this article, we will setup an Oauth2 authenticartion server with Django.

This server will be a REST API usable by other services for authentication using email and password.

This is the part 1 of the tutorial where we will setup the authentication server

Part 2 here: https://blog.florianbgt.com/Django_Oauth2_toolkit_Part2_Resource_server

You can find the source code on my github: https://github.com/florianbgt/django-rest-oauth2-authserver

## 1) Setting up our project

In this section, we will setu p our Django project and customize the default user model to implement email authentication

First, we create our virtualenvironement:

```bash
mkdir Django-rest-oauth2-auth-server
cd Django-rest-oauth2-auth-server
python -m venv env
### if you are using Windows
env\scripts\activate
### if you are using Mac or Linux
source env/bin/activate
```

We then install our dependencies and create our project

```bash
pip install django
django-admin startproject _project .
```

We now implement user authentication using email. First, we create an app `users` and a few files:

c

The first thing we are going to modify is Django user manager. This is where we tell Django how to create users. We do so by extending `BaseUserManager`:

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

We then define our model. For this, we extend `AbstractUser`.  
Here, we set all field the default field of the default user model to None exept for the email address.  
We also ensure that the email address is unique as this field will be use for authentication (`USERNAME_FIELD = 'email'`).
Here i the code below:

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

Our custom user will not be shown in the admin by default. Let's fix that by appending `UserCreationForm` and `UserChangeForm`:

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

We can now register these 2 forms and customize the admin page:

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

Finally, we add our `users` app and register our `CustomUser` in our settings:

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

We can now migrate (**It is very important not to migrate your database before the creation of our `CustomUser`**)  
We can also create our superuser, only email and password should be prompted:

```bash
python manage.py makemigrations users
python manage.py migrate
python manage.py createsuperuser
Email: admin@email.com
Password: testpass123
Password (again): testpass123
```

We now can spin up our server and go to http://localhost:8000/admin.  
You should be prompted to login using your email instead of the default username.

```bash
python manage.py runserver
```

In the admin, we should also see our `CustomUser` display correctly:

<div><blog-img src="admin_custom_user.png" alt="Django admin with custom user" width="100%" height="auto" class="shadow mb-3"/></div>

## 2) Setting up Oauth2 using Django Oauth Toolkit

To setup Oauth2, we are going to use the Django Oauth Toolkit library (https://django-oauth-toolkit.readthedocs.io/en/latest/)

First, we start by installing the library. We also install django rest framework to create extra API endpoints that will be usefull in the Part 2 of this article:

```bash
pip install djangorestframework django-oauth-toolkit
```

We then configure Django rest framework and Django Oauth Tookit according to Oauth Toolkit documentation (https://django-oauth-toolkit.readthedocs.io/en/latest/rest-framework/getting_started.html#step-1-minimal-setup).  
I will also include session authentication for the admin  
Finally, I include the `introspection` scope. This scope will be used by the resource server we will setup in Part 2 of this article

```python
### _project/settings.py
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
        'introspection': 'Introspect token scope',     #new
    }       #new
}       #new

REST_FRAMEWORK = {      #new
    'DEFAULT_AUTHENTICATION_CLASSES': (     #new
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',     #new
        'rest_framework.authentication.SessionAuthentication',     #new
    ),     #new
    'DEFAULT_PERMISSION_CLASSES': (      #new
        'rest_framework.permissions.IsAuthenticated',      #new
    )      #new
}      #new
```

We then can register Oauth Toolkit in our project urls:

```python
### _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),       #new
]
```

We now need to apply our migrations and spin up our server:

```bash
python manage.py migrate
python manage.py runserver
```

We now need to create an access to our Oauth. For this, we need to register an new applicartion in Oauth.  
Django Oauth Toolkit provide a great interface for it at http://localhost:8000/o/applications/register/  
CLick on register and fill the form as below:

- Name: resource-server
- Cliend id: rs-id,
- Client secret: rs-secret
- Client type: confidential
- Authorization grant type: Resource owner password-based
- Redirect uris: leave this empty
- Algorithm: No OIDC support

We can now request an access token by sending the following http request using Postman:

<div><blog-img src="postman_get_token.png" alt="Postman http request to get token" width="100%" height="auto" class="shadow mb-3"/></div>

To access our future resources, we will attached the access token to all of our request.

Once our token expire, we can request a new one using our long life refren token by sending the following http request:

<div><blog-img src="postman_refresh_token.png" alt="Postman http request to refresh token" width="100%" height="auto" class="shadow mb-3"/></div>

## 3) Setup Signup and password change API views

We are now going to setup a signup and a password change API views.

We start by creating the serializers for these 2 view:

```bash
touch users/serializers.py
```

```python
### users/serializers.py
from rest_framework import serializers      #new
from django.contrib.auth import get_user_model      #new
from django.contrib.auth.password_validation import validate_password       #new
from rest_framework.validators import UniqueValidator       #new

User = get_user_model()     #new


class SignUpSerializer(serializers.ModelSerializer):     #new
    email = serializers.EmailField(validators=[UniqueValidator(queryset=User.objects.all())])     #new
    password = serializers.CharField(write_only=True, validators=[validate_password])     #new
    password2 = serializers.CharField(write_only=True)     #new

    class Meta:     #new
        model = User     #new
        fields = ['email', 'password', 'password2',]     #new

    def validate(self, value):     #new
        if value['password'] != value['password2']:     #new
            raise serializers.ValidationError({'password2': 'Password fields did not match'})     #new
        return value     #new
    
    def create(self, validated_data):     #new
        user = User.objects.create(email = validated_data['email'])     #new
        user.set_password(validated_data['password'])     #new
        user.save()     #new
        return user     #new


class PasswordChangeSerializer(serializers.ModelSerializer):        #new
    password = serializers.CharField(write_only=True, required=True, validators=[validate_password])        #new
    password2 = serializers.CharField(write_only=True, required=True)       #new
    old_password = serializers.CharField(write_only=True, required=True)        #new

    class Meta:        #new
        model = User        #new
        fields = ('old_password', 'password', 'password2')        #new

    def validate_old_password(self, value):        #new
        user = self.context['request'].user        #new
        if not user.check_password(value):        #new
            raise serializers.ValidationError({'old_password': 'Old password is incorrect'})        #new
        return value        #new

    def validate(self, value):        #new
        if value['password'] != value['password2']:        #new
            raise serializers.ValidationError({'password2': 'Password fields did not match'})        #new
        return value        #new

    def update(self, instance, validated_data):        #new
        instance.set_password(validated_data['password'])        #new
        instance.save()        #new
        return instance        #new
```

We then create the views:
```python
### users/views.py
from rest_framework import generics     #new
from rest_framework.permissions import AllowAny     #new
from django.contrib.auth import get_user_model     #new
from .serializers import SignUpSerializer, PasswordChangeSerializer     #new

User = get_user_model()     #new


class SignUp(generics.CreateAPIView):       #new
    permission_classes = [AllowAny]       #new
    serializer_class = SignUpSerializer       #new


class PasswordChange(generics.UpdateAPIView):     #new
    queryset = User     #new
    serializer_class = PasswordChangeSerializer     #new
    def get_object(self):     #new
        return self.request.user     #new
```

And finally, we register these 2 views in our urls:

```bash
touch users/urls.py
```

```python
### users/urls.py
from django.urls import path        #new
from .views import SignUp, PasswordChange        #new


urlpatterns = [        #new
    path('signup/', SignUp.as_view()),
    path('password/', PasswordChange.as_view()),        #new
]        #new
```

```python
### _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),       #new
    path('users/', include('users.urls'))        #new
]
```

We can now try to signup a new user using the following http request:

<div><blog-img src="postman_signup.png" alt="Postman http request to signup a new user" width="100%" height="auto" class="shadow mb-3"/></div>

We also now get a token with that new user:

<div><blog-img src="postman_get_token_user1.png" alt="Postman http request to get token for new user" width="100%" height="auto" class="shadow mb-3"/></div>

Finally, we can change the password of this new user. Here, we need to attached the access token in a authorization header (type Bearer):

<div><blog-img src="postman_change_password.png" alt="Postman http request to change password for new user" width="100%" height="auto" class="shadow mb-3"/></div>

## Conclusion

We now have our Oauth server setup!

In the part 2 of this tutorial (https://blog.florianbgt.com/Django_Oauth2_toolkit_Part2_Resource_server) we will see how to setup a separate resource server that use this Oauth2 API for authentication

You can find the source code of this article on my github: https://github.com/florianbgt/django-rest-oauth2-authserver

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com