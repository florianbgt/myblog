---
title: Production ready Universal PWA with SSR using Nuxt, Django, Docker and AWS (Part 1)
description: Create an universal PWA with SSR and deploy it on AWS usign Nuxt, Django and Docker. Part 1, Django API
writtenBy: "Florian Bigot"
---

In this article, we will setup a production ready universal PWA using Nuxt, Django, Docker and AWS.

This is the part 1 of the tutorial where we will setup the authentREST API using Django

Part 2 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part2_Nuxt_Frontend

Part 3 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part3_AWS_Deployment

You can find the source code on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

## 1) Setting up our project

To setup our project, we will use Docker in order not to install anything on our machine. To follow this tutorial, the only thing needed is then to havve Docker installed on your machine (more info here: https://docs.docker.com/get-docker/)

We start by creating the following files:

```bash
mkdir api
touch api/requirements.txt
touch api/Dockerfile
touch docker-compose.yml
touch .env
```

```python
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
```

```yaml
### api/Dockerfile
FROM python:3.8
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt /code/
RUN pip install -r requirements.txt
COPY . /code/
```

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres
    volumes:
      - ./db:/var/lib/postgresql/data
    env_file:
      - .env
  api:
  restart: always
    build:
      context: api
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./api:/code
    ports:
      - "8000:8000"
    depends_on:
      - db
```

```txt
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

We then can run the following command to build our image and create our Django project:

```bash
docker-compose run api django-admin startproject _project .
```

We now now have your `api` image built and your django project setup! We can run the `api` service in Docker using the following command:

```bash
docker-compose up
```

If we go to http://localhost:8000/, we should see our `api` up and running:

<div><blog-img src="django_success.png" alt="Django landing page" width="100%" height="auto" class="shadow mb-3"/></div>

Currently, our app is using the default `sqlite` database. To use the Dockerized `postgresql` database we have set up, we need to modify our `settings.py` like below:

```python
### api/_project/settings.py
...
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
...
```

If your container is still running, your `api` should have picked up that change and have reloaded. The `api` service now use the the containerized posgreSQL database.

## 2) Modify the User model

Let's modify the default user model to remove `first_name` and `last_name`.

First, we can create an app called `users`. For this shutdown your server and run the following command to properly remove your containers:

```bash
docker-compose down
```

Then, run the following command to create the new `users` app:

```bash
docker-compose run api python manage.py startapp users
```

We now can do the following changes to implement email login:

```bash
touch api/users/forms.py
```

```python
### api/users/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser


class CustomUser(AbstractUser):
    first_name = None
    last_name = None

    def __str__(self):
        return self.email
```

```python
### api/users/forms.py
from django.contrib.auth.forms import UserCreationForm, UserChangeForm
from django.contrib.auth import get_user_model


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = get_user_model()
        fields = ('email', 'username',)


class CustomUserChangeForm(UserChangeForm):
    class Meta:
        model = get_user_model()
        fields = ('email', 'username',)
```

```python
### api/users/admin.py
from django.contrib import admin
from django.contrib.auth import get_user_model
from django.contrib.auth.admin import UserAdmin

from .forms import CustomUserCreationForm, CustomUserChangeForm


class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = get_user_model()
    list_display = ('email', 'is_staff', 'is_active',)
    list_filter = ('email', 'is_staff', 'is_active',)
    fieldsets = (
        ('Credentials', {'fields': ('email', 'password')}),
        ('Permissions', {'fields': ('is_staff', 'is_active', 'groups')}),
    )
    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'password1', 'password2', 'is_staff', 'is_active')}
        ),
    )
    search_fields = ('email',)
    ordering = ('email',)


admin.site.register(get_user_model(), CustomUserAdmin)
```

```python
### api/_project/settings.py
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    #local
    'users',
]

AUTH_USER_MODEL = 'users.CustomUser'
...
```

We can now migrate our database for our change to take effect. We also create a superuser to check our changes in the admin page:

```bash
docker-compose run api python manage.py makemigrations users
docker-compose run api python manage.py migrate
docker-compose run api python manage.py createsuperuser
```

We can then spin up our docker container:

```bash
docker-compose up
```

You should now be able to login to the admin page, create, modify and delete users.
http://localhost:8000/admin/

## 3) JWT and google authentication

For authentication, will use the `dj-rest-auth` library. This library provide out of the box JWT authentication as well as allauth support for social authentication. In this tutorial, will use JWT and google to signup our users.

We will also install the `djangorestframework` and `djangorestframework-simplejwt` packages.

To do this, we just have to modify our `requirements.txt` file and rebuild our image:

```txt
### requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
dj-rest-auth[with_social]==2.1.11
djangorestframework-simplejwt==4.7.2
```

Then, we can shut down our server and rebuild our image using the `--build` option:

```bash
docker-compose down
docker-compose up --build
```

The new dependencies have now been installed and our `api` is up and running again!

To use `djangorestframework` and `dj-rest-auth`, add the following code to you `settings.py`:

```python
### api/_project/settings.py
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',
    #3rd party
    'rest_framework',
    'rest_framework.authtoken',
    'allauth',
    'allauth.account',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount',
    'dj_rest_auth',
    'dj_rest_auth.registration',
    #local
    'users',
]

AUTH_USER_MODEL = 'users.CustomUser'

#dj-rest-auth
REST_USE_JWT = True
JWT_AUTH_COOKIE = 'access'
JWT_AUTH_REFRESH_COOKIE = 'refresh'
from datetime import timedelta
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
}

#Allauth
SITE_ID = 1
ACCOUNT_UNIQUE_EMAIL = True
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_AUTHENTICATION_METHOD = 'email'
ACCOUNT_EMAIL_VERIFICATION  = 'none'

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'dj_rest_auth.jwt_auth.JWTCookieAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]
...
```

```python
### api/users/views.py
from allauth.socialaccount.providers.google.views import GoogleOAuth2Adapter
from allauth.socialaccount.providers.oauth2.client import OAuth2Client
from dj_rest_auth.registration.views import SocialLoginView

class GoogleLogin(SocialLoginView):
    adapter_class = GoogleOAuth2Adapter
    client_class = OAuth2Client
```

```python
### api/_project/urls.py
from django.contrib import admin
from django.urls import path, include
from users.views import GoogleLogin

urlpatterns = [
    path('admin/', admin.site.urls),
    path('auth/', include('dj_rest_auth.urls')),
    path('auth/registration/', include('dj_rest_auth.registration.urls')),
    path('auth/google/', GoogleLogin.as_view())
]
```

Finally, we can migrate our database to apply our changes

```bash
docker-compose run api python manage.py migrate
```

We are now able to sign up and sign in user!

More config are needed for google authentication. Will set that up later on in the part 3 of this tutorial.

## 4) Create more API views

Time to create some API views that are going to be used as main ccontent for our application!

First, we create a new `receipe` app:

```bash
docker-compose run api python manage.py startapp receipe
```

Then add the folowing content to create our views:

```touch
api/receipe/serializers.py
```

```python
### api/receipe/models.py
from django.db import models
from django.contrib.auth import get_user_model

class Receipe(models.Model):
    name = models.CharField(max_length=50)
    image = models.ImageField(upload_to='receipe/%Y/%m/%d')
    instruction = models.TextField(max_length=10000)
    user = models.ForeignKey(get_user_model(), on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.name
```

```python
### api/receipe/admin.py
from django.contrib import admin
from .models import Receipe


@admin.register(Receipe)
class ReceipeAdmin(admin.ModelAdmin):
    list_display = ['id', 'name', 'created_at', 'updated_at',]
```

```python
### api/receipe/serializers.py
import os
from rest_framework import serializers
from .models import Receipe
from django.contrib.auth import get_user_model


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = get_user_model()
        fields = ['id', 'email']


class ReceipeSerializer(serializers.ModelSerializer):
    user = UserSerializer(read_only=True)
    class Meta:
        model = Receipe
        fields = '__all__'

    def create(self, validated_data):
        validated_data['user'] = self.context['request'].user
        return Receipe.objects.create(**validated_data)
```

```python
### api/receipe/views.py
from rest_framework import generics
from django.contrib.auth import get_user_model
from .models import Receipe
from .serializers import ReceipeSerializer


class RequestList(generics.ListCreateAPIView):
    queryset = Receipe.objects.all()
    serializer_class = ReceipeSerializer


class RequestDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Receipe.objects.all()
    serializer_class = ReceipeSerializer
```

```python
### api/_project/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf.urls.static import static
from django.conf import settings
from users.views import GoogleLogin
from receipe.views import ReceipeList, ReceipeDetail

urlpatterns = [
    path('admin/', admin.site.urls),
    path('auth/', include('dj_rest_auth.urls')),
    path('auth/registration/', include('dj_rest_auth.registration.urls')),
    path('auth/google/', GoogleLogin.as_view()),
    path('receipes/', ReceipeList.as_view()),
    path('receipes/<int:pk>/', ReceipeDetail.as_view()),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

```

```python
### api/_project/settings.py
from pathlib import Path
import os
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',
    #3rd party
    'rest_framework',
    'rest_framework.authtoken',
    'allauth',
    'allauth.account',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount',
    'dj_rest_auth',
    'dj_rest_auth.registration',
    #local
    'users',
    'receipe',
    #3rd party
    'django_cleanup',   #need to be last
]
...
STATIC_URL = '/static/'

MEDIA_URL = '/api/media/'
MEDIA_ROOT = os.path.join(BASE_DIR.parent, 'media').replace('\\', '/')
...
```

At the project level, we create a `media` folder that we bind mount to our docker container using volumes:

```bash
mkdir media
```

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres
    volumes:
      - ./db:/var/lib/postgresql/data
    env_file:
      - .env
  api:
    restart: always
    build:
      context: api
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./api:/code
      - ./media:/media
    ports:
      - "8000:8000"
    depends_on:
      - db
```

In our `settings.py`, we added a 3rd party app `django_cleanup` (this library automatically delete media files on instance deletion). We need to install that library. Django `ImageField` also require the `Pillow` library. We can install these libraries by modifying our `requirements.txt`:

```txt
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
dj-rest-auth[with_social]==2.1.11
djangorestframework-simplejwt==4.7.2
django-cleanup==5.2.0
Pillow==8.3.1
```

We now rebuild our image and migrate our database for our change to take effect:

```bash
docker-compose build api
docker-compose run api python manage.py makemigrations receipe --build
docker-compose run api python manage.py migrate
```

And spin up our containers:

```bash
docker-compose up
```

We can now create, modify and delete receipes using the below urls:
- http://localhost:8000/receipes/
- http://localhost:8000/receipes/Id/

## Conclusion

Our API is now setup!

In the part 2 od this tutorial, we will see how to set up our frontend using Nuxt.

You can find the source code of this article on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com