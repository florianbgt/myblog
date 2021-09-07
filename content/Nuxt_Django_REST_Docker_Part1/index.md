---
title: Nuxt Django app - Part 1
description: Create a dockerized Nuxt app with PWA and SSR support using a python REST API built with Django. Part 1, create the REST API
image: "nuxt-python.png"
writtenBy: "Florian Bigot"
---

In this tutorial, we will setup a dockerized Nuxt app with PWA and SSR capabilities. This app will consume a python REST API built with Django.

This is the part 1 of the tutorial where we will setup the python REST API

[Part 2 here](https://blog.florianbgt.com/Nuxt_Django_REST_Docker_Part2)

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-REST-Docker)

## 1) Setting up our project

To setup our project, we will use Docker. So the only dependency needed is docker itself!

If you do not have Docker installed on your machine, you can get it [here](https://docs.docker.com/get-docker/).

let's start by creating our api directory and the following files:

```bash
mkdir api
touch api/requirements.txt
touch api/Dockerfile
touch docker-compose.yml
```

`requirements.txt` will be use by docker to install the dependency we need.

```python
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
```

`Dockerfile` will be use by docker to build the image which our API will run in.

```yaml
### api/Dockerfile
FROM python:3.8
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

`docker-compose.yml` will be use during our developement to orchestrate our containers.
For now, we are going to use 2 containers:
- A postgreSQL database (official image from Docker hub)
- Our custom API image

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres:13.4
    volumes:
      - ./db:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
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

Using these files and Docker, we simply run the following command to create our Django project

```bash
docker-compose run api django-admin startproject _project .
```

Our project is now setup! We should now have Django files creaated inside the `api` directory thanks to the bind mount we defined in the `docker-compose.yml` file.

We can make sure our API is working well by spinning up our container using the following command.

```bash
docker-compose up
```

If we go to http://localhost:8000/, we should see our Django app up and running:

<div><blog-img src="django_success.png" alt="Django landing page" width="100%" height="auto" class="shadow mb-3"/></div>

Currently, our app is using the default `sqlite` database Django provides. We could use this database for developement. However, I like to use a postgreSQL database during developement to mimic our production environement as much as possible and avoid potential headache later on.

Thanksfully, the official `postgresql` docker image make things extrimely easy for us!

A `db` directory should also have been created. This is the files used by the postgres official image that run our postgreSQL database.

We only need to modify our `settings.py` like below to connect to the dockerized postgreSQL database.

```python
### api/_project/settings.py
...
impost os
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

If your containers are still running, Django should have picked up that change and have reloaded.

The `api` service now uses the the dockerized posgreSQL database.

## 2) Modify the User model to implement email login

We are going to implement email login as this is more convinient for users than username.

For this, we create a `users` app using the following command.

```bash
docker-compose run api python manage.py startapp users
```

We do the following changes to implement email login.

We start by overiding Django's user model and create our own.

```python
### api/users/models.py
from django.db import models
from django.contrib.auth.base_user import BaseUserManager
from django.contrib.auth.models import AbstractUser


class CustomUserManager(BaseUserManager):
    def create_user(self, email, password, **extra_fields):
        if not email:
            raise ValueError('email must be set')
        email = self.normalize_email(email)
        user = self.model(email = email, **extra_fields)
        user.set_password(password)
        user.save()
        return user
    
    def create_superuser(self, email, password, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)
        if extra_fields.get('is_staff') is not True:
            raise ValueError('superuser must have is_staff = True')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('superuser must have is_superuser = True')
        return self.create_user(email, password, **extra_fields)


class CustomUser(AbstractUser):
    username = None
    first_name = None
    last_name = None
    email = models.EmailField(max_length=50, unique = True)

    REQUIRED_FIELDS = []
    USERNAME_FIELD = 'email'

    objects = CustomUserManager()

    def __str__(self):
        return self.email
```

Then, we create custom forms to overide Django's.

```bash
touch api/users/forms.py
```

```python
### api/users/forms.py
from django.contrib.auth.forms import UserCreationForm, UserChangeForm
from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = CustomUser
        fields = ('email',)


class CustomUserChangeForm(UserChangeForm):
    class Meta:
        model = CustomUser
        fields = ('email',)
```

We also, modify `admin.py` so our user model is easily accessible from the admin page.

```python
### api/users/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .forms import CustomUserCreationForm, CustomUserChangeForm
from .models import CustomUser


class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = CustomUser
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


admin.site.register(CustomUser, CustomUserAdmin)
```

Finaly, we register our user app in our installed app and point `AUTH_USER_MODEL` to our new custom user model.

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

For our change to take effect, we need to migrate our database. We do this using the following 2 commands.

```bash
docker-compose run api python manage.py makemigrations users
docker-compose run api python manage.py migrate
```

Then, we create our superuser to be able to access the admin page.

```bash
docker-compose run api python manage.py createsuperuser
```

Finally we spin up our docker containers using docker-compose.

```bash
docker-compose up
```

We should now be able to login to the admin page (http://localhost:8000/admin) using email instead of username!

<div><blog-img src="django_admin_email_login.png" alt="Django admin email login" width="100%" height="auto" class="shadow mb-3"/></div>

We also can create, modify and delete users from the admin page (http://localhost:8000/admin/users/).

## 3) Setting up JWT authentication

For authentication, we will use the `djangorestframework-simplejwt` library. This library provide out of the box JWT authentication.

To do this, we just have to modify our `requirements.txt` file.

```txt
### requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.4
djangorestframework-simplejwt==4.7.2
```

Then, we need to rebuild our image. We can do this by using `docker-compose up --build`.

The new dependencies have now been installed and our Django api is up and running again!

To use and configure `djangorestframework` and `djangorestframework-simplejwt`, we add the following code to you `settings.py`.

```python
### api/_project/settings.py
...
from datetime import timedelta
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',
    #local
    'users',
    #3rd party
    'rest_framework',
]
...
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}
...
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
...
```

We also add `djangorestframework-simplejwt` routes to our project in order to get access token and request refresh token to our API.

```python
### api/_project/urls.py
from django.contrib import admin
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('token/', TokenObtainPairView.as_view()),
    path('token/refresh/', TokenRefreshView.as_view()),
]

```

We are now able to sign in user using JWT and refresh them so the user stays loggged in.

- Get access token: http://localhost:8000/token/
- Refresh token: http://localhost:8000/token/refresh/

<div><blog-img src="get_token_and_refresh_token.png" alt="Get token and refresh token API view" width="100%" height="auto" class="shadow mb-3"/></div>

## 4) Setting signup, email change and password change

Let's now create some API view for our user to:
- Signup
- Access and change their email
- Change their password

We start by creating our serializers.

```bash
touch api/users/serializers.py
```

```python
### api/serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model
from rest_framework.validators import UniqueValidator
from django.contrib.auth.password_validation import validate_password


class SignUpSerializer(serializers.ModelSerializer):
    email = serializers.EmailField(required=True, validators=[UniqueValidator(queryset=get_user_model().objects.all())])
    password = serializers.CharField(required=True, write_only=True, validators=[validate_password])
    password2 = serializers.CharField(required=True, write_only=True)

    class Meta:
        model = get_user_model()
        fields = ('email', 'password', 'password2',)

    def validate(self, value):
        if value['password'] != value['password2']:
            raise serializers.ValidationError({"password2": "Password fields did not match"})
        return value

    def create(self, validated_data):
        user = get_user_model().objects.create(email=validated_data['email'])
        user.set_password(validated_data['password'])
        user.save()
        return user


class EmailSerializer(serializers.ModelSerializer):
    class Meta:
        model = get_user_model()
        depth = 1
        fields = ('id', 'email',)


class PasswordChangeSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, required=True, validators=[validate_password])
    password2 = serializers.CharField(write_only=True, required=True)
    old_password = serializers.CharField(write_only=True, required=True)

    class Meta:
        model = get_user_model()
        fields = ('old_password', 'password', 'password2')

    def validate_old_password(self, value):
        user = self.context['request'].user
        if not user.check_password(value):
            raise serializers.ValidationError({'old_password': 'Old password is incorrect'})
        return value

    def validate(seld, value):
        if value['password'] != value['password2']:
            raise serializers.ValidationError({'password2': 'Password fields did not match'})
        return value

    def update(self, instance, validated_data):
        instance.set_password(validated_data['password'])
        return instance
```

Then, we create our view. Using djangorestframework's generics view make things extremely easy for us!

```python
### api/users/views.py
from rest_framework import generics
from rest_framework.permissions import AllowAny
from django.contrib.auth import get_user_model
from .serializers import SignUpSerializer, EmailSerializer, PasswordChangeSerializer


class SignUp(generics.CreateAPIView):
    permission_classes = [AllowAny]
    serializer_class = SignUpSerializer


class Email(generics.RetrieveUpdateAPIView):
    queryset = get_user_model()
    serializer_class = EmailSerializer
    def get_object(self):
        return self.request.user


class PasswordChange(generics.UpdateAPIView):
    queryset = get_user_model()
    serializer_class = PasswordChangeSerializer
    def get_object(self):
        return self.request.user
```

Finally, we add the crreated view to our `urls.py`.

```python
### api/_project/urls.py
from django.contrib import admin
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)
from users.views import SignUp, Email, PasswordChange

urlpatterns = [
    path('admin/', admin.site.urls),
    path('token/', TokenObtainPairView.as_view()),
    path('token/refresh/', TokenRefreshView.as_view()),
    path('signup/', SignUp.as_view()),
    path('user/', Email.as_view()),
    path('password/change/', PasswordChange.as_view())
]
```

Users are now able to signup at http://localhost:8000/signup/.

<div><blog-img src="signup.png" alt="Signup API view" width="100%" height="auto" class="shadow mb-3"/></div>


Change their email at http://localhost:8000/user/.

<div><blog-img src="email_change.png" alt="Email change API view" width="100%" height="auto" class="shadow mb-3"/></div>

And change their password at http://localhost:8000/password/change/.

<div><blog-img src="password_change.png" alt="Password change API view" width="100%" height="auto" class="shadow mb-3"/></div>

## 5) Setting up Password reset by email

For our user to properly use our app, one more feature is nice (if not mandatory) to have. Password reset by email.

Password reset is pretty hard to implement by our own. Fortunately, the `django-rest-passwordreset` library makes such implementation a breeze.

To use this library, will need to implement sending email as well. I will use a gmail account for this. However, steps are similar for all email providers.

We start by installing `django-rest-passwordreset` by adding it into our `requirements.txt`.

```txt
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
djangorestframework-simplejwt==4.7.2
django-rest-passwordreset==1.2.0
```

We add `django_rest_passwordreset` and setup our email provider in our `settings.py`. 

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
    'recipes',
    #3rd parties
    'rest_framework',
    'django_rest_passwordreset',
]
...
EMAIL_BACKENDS = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = <Your email here>,
EMAIL_HOST_PASSWORD = <Your password here>,
EMAIL_PORT = 587 #for gmail only
EMAIL_USE_TLS = True
DEFAULT_FROM_EMAIL = <Your email here>,
...
```

We also add `django-rest-passwordreset` view to our `urls.py` file.

```python
### api/_project.urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)
from users.views import SignUp, Email, PasswordChange

urlpatterns = [
    path('admin/', admin.site.urls),
    path('token/', TokenObtainPairView.as_view()),
    path('token/refresh/', TokenRefreshView.as_view()),
    path('signup/', SignUp.as_view()),
    path('user/', Email.as_view()),
    path('password/change/', PasswordChange.as_view()),
    path('password/reset/', include('django_rest_passwordreset.urls')),
]
```

By default, `django-rest-passwordreset` will not send email to user. We are going to use a signal for this.

First, we can start by creating a the template of the email sent to our users. In these template, we will inject the user's email as well as the password reset url.

```bash
touch api/users/templates/email/user_reset_password.html
touch api/users/templates/email/user_reset_password.txt
```

```html
### api/users/templates/email/user_reset_password.html
<html>
  <p>Hi {{email}}</p>
  <p>
    You are receiving this email because you have requested a password reset.
  </p>
  <p>
    Please click on the link below to reset your password.
    <br />
    <a href="{{reset_password_url}}">{{reset_password_url}}</a>
  </p>
  <p>
    If you did not requested this password reset, please change your password
    ASAP
  </p>
  <p>
    Regards
    <br />
    Awesome Recipes
  </p>
</html>
```

```txt
### api/users/templates/email/user_reset_password.txt
Hi {{email}},

You are receiving this email because you have requested a password reset.

Please click on the link below to reset your password.
{{reset_password_url}}

If you did not requested this password reset, please change your password ASAP

Regards
Awesome Recipes
```

Inside our `users/models.py`, will create a function to send the eamail to the users that will be fired everytime a password reset token is created.

```python
### api/users/models.py
from django.db import models
from django.contrib.auth.base_user import BaseUserManager
from django.contrib.auth.models import AbstractUser
from django.core.mail import EmailMultiAlternatives
from django.dispatch import receiver
from django.template.loader import render_to_string
from django_rest_passwordreset.signals import reset_password_token_created
...
@receiver(reset_password_token_created)
def password_reset_token_created(sender, instance, reset_password_token, *args, **kwargs):
    context = {
        'email': reset_password_token.user.email,
        'reset_password_url': f"http://localhost:3000/password/reset/confirm/?token={reset_password_token.key}"
    }
    email_html_message = render_to_string('email/user_reset_password.html', context)
    email_plaintext_message = render_to_string('email/user_reset_password.txt', context)
    msg = EmailMultiAlternatives("Password Reset", email_plaintext_message, [], [reset_password_token.user.email])
    msg.attach_alternative(email_html_message, "text/html")
    msg.send()
```

We now can now re build our image and migrate our database to include our changes

```bash
docker-compose build api
docker-compose run api python manage.py migrate
```

Password reset is now implemented!

We now can spin up our containers and try it out!

```bash
docker-compose up
```

We can request password reset to http://localhost:8000/password/reset/

<div><blog-img src="password_reset_request.png" alt="Password reset request API view" width="100%" height="auto" class="shadow mb-3"/></div>

Verify token validity to http://localhost:8000/password/reset/verify_token/

<div><blog-img src="password_reset_verify.png" alt="Password reset verify API view" width="100%" height="auto" class="shadow mb-3"/></div>

And reset password to http://localhost:8000/password/reset/confirm/

<div><blog-img src="password_reset_confirm.png" alt="Password reset confirm API view" width="100%" height="auto" class="shadow mb-3"/></div>

## 6) Create recipes API views

We are now going to create the API views that are going to deliver the main content of our app.

First, we create the `recipes` app.

```bash
docker-compose run api python manage.py startapp recipes
```

Then we create the following models.

```python
### api/recipes/models.py
from django.db import models
from django.contrib.auth import get_user_model


class Recipe(models.Model):
    name = models.CharField(max_length=50)
    image = models.ImageField(upload_to='recipes/%Y/%m/%d')
    instruction = models.TextField(max_length=10000)
    user = models.ForeignKey(get_user_model(), on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.name
```

We also edit `admin.py` to be able to manage this recipe model from the admin page.

```python
### api/recipes/admin.py
from django.contrib import admin
from .models import Recipe


@admin.register(Recipe)
class RecipeAdmin(admin.ModelAdmin):
    list_display = ['id', 'name', 'created_at', 'updated_at',]
```

We create the serializers.

```bash
touch api/recipes/serializers.py
```

```python
### api/recipes/serializers.py
from rest_framework import serializers
from .models import Recipe
from django.contrib.auth import get_user_model


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = get_user_model()
        fields = ['id', 'email']


class RecipeSerializer(serializers.ModelSerializer):
    user = UserSerializer(read_only=True)
    class Meta:
        model = Recipe
        fields = '__all__'

    def create(self, validated_data):
        validated_data['user'] = self.context['request'].user
        return Recipe.objects.create(**validated_data)
```

And then the views.

```python
### api/recipes/views.py
from rest_framework import generics
from rest_framework import permissions
from django.contrib.auth import get_user_model
from .models import Recipe
from .serializers import RecipeSerializer


class OwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS or obj.user == request.user:
            return True


class RecipeList(generics.ListCreateAPIView):
    queryset = Recipe.objects.all()
    serializer_class = RecipeSerializer


class RecipeDetail(generics.RetrieveUpdateDestroyAPIView):
    permission_classes = [permissions.IsAuthenticated, OwnerOrReadOnly]
    queryset = Recipe.objects.all()
    serializer_class = RecipeSerializer
```

Finally, we add the views to our `urls.py`. Here, we also add a static route to serve our media files that are uploaded by users.

```python
### api/_project/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf.urls.static import static
from django.conf import settings
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)
from users.views import SignUp, Email, PasswordChange
from recipes.views import RecipeList, RecipeDetail

urlpatterns = [
    path('admin/', admin.site.urls),
    path('token/', TokenObtainPairView.as_view()),
    path('token/refresh/', TokenRefreshView.as_view()),
    path('signup/', SignUp.as_view()),
    path('user/', Email.as_view()),
    path('password/change/', PasswordChange.as_view()),
    path('password/reset/', include('django_rest_passwordreset.urls')),
    path('recipes/', RecipeList.as_view()),
    path('recipes/<int:pk>/', RecipeDetail.as_view()),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

We also need to add this new app to our installed app. We also setup our `MEDIA_URL` and `MEDIA_ROOT` to serve our media files.

```python
### api/_project/settings.py
...
from datetime import timedelta
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
    #local
    'users',
    'recipes',
    #3rd party
    'rest_framework',
    'django_rest_passwordreset',
]
...
MEDIA_URL = '/api/media/'
MEDIA_ROOT = os.path.join(BASE_DIR.parent, 'media').replace('\\', '/')
...
```

Images will get uploaded to the `media` folder we defined above.

However, on image edit or deletion, the image will remain in the `media` folder.

Fortunately, we can install the `django_cleanup` library. This library will delete media files on instance modification and deletion using Django signals.

We also need to install the `Pillow` library to use Django `ImageField` in our recipe model.

Finally, we need to install `django-cors-headers` as our frontend will not be served using the same host as the Django API.

```txt
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
djangorestframework-simplejwt==4.7.2
django-cleanup==5.2.0
Pillow==8.3.1
django-cors-headers==3.8.0
```

We also need to modify our `settings.py` to include these library

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
    #local
    'users',
    'recipes',
    #3rd party
    'rest_framework',
    'django_rest_passwordreset',
    'corsheaders',
    'django_cleanup',   #need to be last
]
...
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware', #need to be first
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
...
CORS_ALLOWED_ORIGINS = ['http://localhost:3000']
...
```

At the project level, we create the `media` folder that we bind mount to our docker container using volumes in order to persist data.

```bash
mkdir media
```

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres:13.4
    volumes:
      - ./db:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
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

We now rebuild our image and migrate our database for our change to take effect:

```bash
docker-compose build api
docker-compose run api python manage.py makemigrations recipes
docker-compose run api python manage.py migrate
```

And spin up our containers:

```bash
docker-compose up
```

We can now create to http://localhost:8000/recipes/

<div><blog-img src="recipes_create.png" alt="Create recipe API view" width="100%" height="auto" class="shadow mb-3"/></div>

Modify and delete existing recipes to http://localhost:8000/recipes/id/

<div><blog-img src="recipes_modify.png" alt="Modify and delete recipe API view" width="100%" height="auto" class="shadow mb-3"/></div>

## Conclusion

Our REST API is now setup!

In the [Part 2](https://blog.florianbgt.com/Nuxt_Django_REST_Docker_Part2) of this tutorial, we will see how to set up our frontend using Nuxt.

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-REST-Docker)

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com