---
title: Production ready Universal PWA with SSR using Nuxt, Django, Docker and AWS (Part 1)
description: Create an universal PWA with SSR and deploy it on AWS usign Nuxt, Django and Docker. Part 1, Django API
writtenBy: "Florian Bigot"
---

In this article, we will setup a production ready universal PWA using Nuxt, Django, Docker and AWS.

This is the part 1 of the tutorial where we will setup the REST API using Django.

Part 2 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part2_Nuxt_Frontend

Part 3 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part3_AWS_Deployment

You can find the source code on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

## 1) Setting up our project

To setup our project, we will use Docker. So the only dependency needed id docker itself!

If you do not have Docker installed on your machine, you can get it here https://docs.docker.com/get-docker/.

let's start by creating the following files:

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
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
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
    env_file:
      - .env
```

```txt
### .env
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432
```

Then we run the following command to build our images and create our Django project.

```bash
docker-compose run api django-admin startproject _project .
```

We now now have your `api` image built and your django project setup thanks to the volume we defined!

We can run our services in Docker using the following command.

```bash
docker-compose up
```

If we go to http://localhost:8000/, we should see our `api` up and running:

<div><blog-img src="django_success.png" alt="Django landing page" width="100%" height="auto" class="shadow mb-3"/></div>

Currently, our app is using the default `sqlite` database.

Will use a postgrSQL database even during production to mimic our production environement as much as possible.

Thanksfully, the official `postgresql` docker image make things extrimely easy for us!

We only need to modify our `settings.py` like below.

```python
### api/_project/settings.py
...
impost os
...
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get("POSTGRES_DB"),
        'USER': os.environ.get("POSTGRES_USER"),
        'PASSWORD': os.environ.get("POSTGRES_PASSWORD"),
        'HOST': os.environ.get("POSTGRES_HOST"),
        'PORT': os.environ.get("POSTGRES_PORT"),
    }
}
...
```

If your containers are still running, Django should have picked up that change and have reloaded. The `api` service now uses the the containerized posgreSQL database.

## 2) Modify the User model

We are going to implement email login as this is more convinient for users than username.

First, we create a `users` app. For this we shutdown our containers using the following command.

```bash
docker-compose down
```

Then, run the following command to create the `users` app:

```bash
docker-compose run api python manage.py startapp users
```

We now can do the following changes to implement email login.

```bash
touch api/users/forms.py
```

```python
### api/users/models.py
from django.contrib.auth.base_user import AbstractBaseUser
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

We then migrate our database for our changes to take effect. We also create a superuser to check our changes in the admin page:

```bash
docker-compose run api python manage.py makemigrations users
docker-compose run api python manage.py migrate
docker-compose run api python manage.py createsuperuser
```

Finally we spin up our docker containers.

```bash
docker-compose up
```

We should now be able to login to the admin page using email instead of username!
We also can create, modify and delete users from the admin page (http://localhost:8000/admin/).

## 3) JWT authentication

For authentication, we will use the `djangorestframework-simplejwt` library. This library provide out of the box JWT authentication.

We will also install the `djangorestframework` library.

To do this, we just have to modify our `requirements.txt` file.

```txt
### requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
djangorestframework-simplejwt==4.7.2
```

Then, we can shut down our containers and rebuild our image using the `--build` option.

```bash
docker-compose down
docker-compose up --build
```

The new dependencies have now been installed and our Django api is up and running again!

To use `djangorestframework` and `dj-rest-auth`, add the following code to you `settings.py`.

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

AUTH_USER_MODEL = 'users.CustomUser'

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}

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

Finally, we can migrate our database to apply our changes.

```bash
docker-compose run api python manage.py migrate
```

We are now able to sign in user using JWT and refresh them so the user stays loggged in (http://localhost:8000/token/ and http://localhost:8000/token/refresh/)!

## 4) Signup, email and password change

Let's now create the basics for our user to access the app:
- Signup new users
- Access and change user's email email
- Change user's password

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

That is it! Users are now able to signup, change their email and their password.

## 5) Password reset

For our user to properly use our app, one more feature is nice (if not mandatory) to have. Password reset.

Password reset is pretty hard to implement. Fortunately, the `django-rest-passwordreset` library makes such implementation a breeze.

To use this library, will need to implement sending email as well. I will use a gmail account for this. However, steps are similar for all email providers

```txt
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.2
djangorestframework-simplejwt==4.7.2
django-rest-passwordreset==1.2.0
```

```txt
### .env
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432

EMAIL_HOST=smtp.gmail.com
EMAIL_HOST_USER=<Your email here>
EMAIL_HOST_PASSWORD=<Your password here>
EMAIL_PORT=587
DEFAULT_FROM_EMAIL=<Your email here>
FRONTEND_URL=http://localhost:3000
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
    'recipes',
    #3rd parties
    'rest_framework',
    'django_rest_passwordreset',
]
...
EMAIL_BACKENDS = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER =  os.environ.get("EMAIL_HOST_USER"),
EMAIL_HOST_PASSWORD = os.environ.get("EMAIL_HOST_PASSWORD"),
EMAIL_PORT = 587
EMAIL_USE_TLS = True
DEFAULT_FROM_EMAIL = os.environ.get("DEFAULT_FROM_EMAIL"),
...
```

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

We now create a signal to send email to user

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

```python
### api/users/models.py
from django.db import models
from django.contrib.auth.base_user import BaseUserManager
from django.contrib.auth.models import AbstractUser
from django.core.mail import EmailMultiAlternatives
from django.dispatch import receiver
from django.template.loader import render_to_string
from django.urls import reverse
from django_rest_passwordreset.signals import reset_password_token_created
...
@receiver(reset_password_token_created)
def password_reset_token_created(sender, instance, reset_password_token, *args, **kwargs):
    url = os.environ.get("FRONTEND_URL")
    context = {
        'email': reset_password_token.user.email,
        'reset_password_url': f"{url}/password/reset/confirm/?token={reset_password_token.key}"
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

Email reset is now implemented! We now can spin up our containers and try it out (http://localhost:8000/password/reset/, http://localhost:8000/password/reset/verify_token/ and http://localhost:8000/password/reset/confirm/)!

## 6) Create recipes API views

We are now going to create the API views that are going to deliver the main content of our app.

First, we create the `recipes` app.

```bash
docker-compose run api python manage.py startapp recipes
```

Then we add the folowing content to create our views.

```bash
touch api/recipes/serializers.py
```

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

```python
### api/recipes/admin.py
from django.contrib import admin
from .models import Recipe


@admin.register(Recipe)
class RecipeAdmin(admin.ModelAdmin):
    list_display = ['id', 'name', 'created_at', 'updated_at',]
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

AUTH_USER_MODEL = 'users.CustomUser'

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}

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
MEDIA_URL = '/api/media/'
MEDIA_ROOT = os.path.join(BASE_DIR.parent, 'media').replace('\\', '/')
...
```

Images will get uploaded to the `media` folder we defined above.

However, on image edit or deletion, the image will remain in the `media` folder.

Fortunately, we can install the `django_cleanup` library. This library will delete media files on instance modification and deletion using Django signals.

We also need to install `Pillow` library to use Django `ImageField`.

Finally, we need to install `django-cors-headers` as our frontend will not be served using the same host as the Django API (at least during development).

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

```txt
### .env
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432

EMAIL_HOST=smtp.gmail.com
EMAIL_HOST_USER=<Your email here>
EMAIL_HOST_PASSWORD=<Your password here>
EMAIL_PORT=587
DEFAULT_FROM_EMAIL=<Your email here>
FRONTEND_URL=http://localhost:3000
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
CORS_ALLOWED_ORIGINS = [os.environ.get("FRONTEND_URL")]
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
    env_file:
      - .env
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

We can now create, modify and delete recipes!

## 7) Final modifications

By default, `ALLOWED_HOSTS` is set to an empty array. That allow the app to be serve from `localhost:8000`.

However, in [Part 2](https://blog.florianbgt.com/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part2_Nuxt_frontend) of this tutorial, we are going to setup a Nuxt app with server side rendering. For the server side rendering API call, we are going to establish a direct connection from container to container using a docker network.

Our django host is not going to be serve from `localhost:8000` anymore but from `api`.

Let include it in our `settings.py`.

```python
### api/_project/settings.py
...
ALLOWED_HOSTS = ['localhost', 'api']
...
```

## Conclusion

Our API is now setup!

In the [Part 2](https://blog.florianbgt.com/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part2_Nuxt_frontend) of this tutorial, we will see how to set up our frontend using Nuxt.

You can find the source code of this article on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com