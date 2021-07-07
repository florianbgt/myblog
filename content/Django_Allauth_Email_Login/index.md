---
title: Django email authentication using Allauth
description: Simple Django project to help you setting up authentication the right way using allauth
image: 'django.png'
writtenBy: 'Florian Bigot'
---

I love Django and I use it as a backend for most of my projects. However, it uses username as authentication method. While it work great, it is a little bit outdated...

Email authentication would be much better right? Fortunately, there is a easy solution, [Django Allauth](https://django-allauth.readthedocs.io)

I am using Python 3.9 for this tutorials

You can find the final code on my Github: https://github.com/florianbgt/django-allauth   

## 1) Setting up the project

We start by creating a folder with the name of your project. I will call mine "django-allauth"   
Then, we create a virtual environment. This will help us to keep our dependencies nice and clean:   
```bash
mkdir django-allauth
cd django-allauth
python -m venv env
# If you use linux and mac:
env/scripts/activate
# If you use windows:
source env/bin/activate
```

We now install django and setup our project:
```bash
pip install django
django-admin startproject _project .
python manage.py runserver
```

If you visit [http://localhost:8000](http://localhost:8000) you should see our dev server up and running:
<div><blog-img src="django_success.png" alt="Django landing page" width="100%" height="auto" class="shadow mb-3"/></div>

## 2) Custom user model

Changing the user model inside a on going project can be very difficult. That is why Django's documentation highly recommends to setup a custom user model at the beginning of a project. That will save a lot of time later on if we ever have to modify that user model.   
https://docs.djangoproject.com/en/3.2/topics/auth/customizing/#using-a-custom-user-model-when-starting-a-project

***It is very important to do these steps before your first database migration.*** If you migrated your database already, then you better start over:
```bash
python manage.py startapp users
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
    #local		#new
    'users',		#new
]
...
```

For our custom user model, we simply extend Django default user model:
```python
### users/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser		#new


class CustomUser(AbstractUser):		#new
    pass		#new
```

We then tell Django to use our custom user model:
```python
### _project/settings.py
...
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

AUTH_USER_MODEL = 'users.CustomUser'		#new
...
```

We finally apply our migrations so the database uses our custom user model:
```bash
python manage.py makemigrations users
python manage.py migrate
```

If we create a superuser and check our custom user model in the admin interface, we notice that it is missing:
```bash
python manage.py createsuperuser
```

<div><blog-img src="django_admin.png" alt="Django admin missing custom user" width="100%" height="auto" class="shadow mb-3"/></div>

That is because we no longer use the default user model. And thus, it is not displayed by default in the admin interface.

Let's fix this:
```python
### users/forms.py

from django.contrib.auth import get_user_model		#new
from django.contrib.auth.forms import UserCreationForm, UserChangeForm		#new

class CustomUserCreationForm(UserCreationForm):		#new
    class Meta:		#new
        model = get_user_model()		#new
        fields = ('email', 'username',)		#new


class CustomUserChangeForm(UserChangeForm):		#new
    class Meta:		#new
        model = get_user_model()		#new
        fields = ('email', 'username',)		#new
```

```python
### users/admin.py

from django.contrib import admin
from django.contrib.auth import get_user_model		#new
from django.contrib.auth.admin import UserAdmin		#new
from .forms import CustomUserCreationForm, CustomUserChangeForm		#new

CustomUser = get_user_model()		#new

class CustomUserAdmin(UserAdmin):		#new
    add_form = CustomUserCreationForm		#new
    form = CustomUserChangeForm		#new
    model = CustomUser		#new
    list_display = ['email', 'username',]		#new

admin.site.register(CustomUser, CustomUserAdmin)		#new
```

We now see the cusotm user model displayed correctly:
<div><blog-img src="django_admin2.png" alt="Django admin with custom user" width="100%" height="auto" class="shadow mb-3"/></div>

## 3) Creating our Homepage

It is time to create our homepage! We create a new app for this:
```bash
python manage.py startapp pages
```

```python
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.sites',
    'django.contrib.staticfiles',
    #local
    'users',
    'pages',		#new
]
...
```

We place our html templates inside a folder in the root of our projects. I will use [Bootstrap 5](https://getbootstrap.com/) to style these html templates:
```bash
mkdir templates
touch templates/_base.html
touch templates/header.html
mkdir templates/pages
touch templates/pages/home.html
```

```html
### templates/_base.html
<!DOCTYPE html>
<html>
    <head>
        <title>Django Allauth</title>
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">
    </head>
    <body>
        {% include "header.html" %}
        <div class="container mt-3">
            <div class="card">
                <div class="card-header">
                    <h1>{% block title %}{% endblock title %}</h1>
                </div>
                <div class="card-body">
                    {% block content %}{% endblock content %}
                </div>
            </div>
        </div>
        <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js" integrity="sha384-MrcW6ZMFYlzcLA8Nl+NtUVF0sA7MsXsP1UyJoMp4YLEuNSfAP+JcXn/tWtIaxVXM" crossorigin="anonymous"></script>
    </body>
</html>
```

```html
### templates/header.html
<nav class="navbar navbar-expand-lg navbar-dark bg-primary">
    <div class="container-fluid">
        <a class="navbar-brand" href="/">Django Allauth</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarNav">
            <ul class="navbar-nav">
                {% if user.is_authenticated %}
                <li class="nav-item">
                    <a class="nav-link" aria-current="page" href="/">Home</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="/accounts/logout">Sign Out</a>
                </li>
                {% endif %}
            </ul>
        </div>
    </div>
</nav>
```

```html
### templates/pages/home.html
{% extends "_base.html" %}

{% block title %}Home Page{% endblock title %}

{% block content %}
<p>Welcome to the homepage <strong>{{ user.email }}</strong></p>
{% endblock content %}
```

We also need to tell Django to use this new "templates" directory:
```python
### _project/settings.py

from pathlib import Path
import os		#new
...
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],		#new
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
...
```

We now create our home page view and route:
```python
# pages/views.py
from django.views.generic import TemplateView
from django.contrib.auth.mixins import LoginRequiredMixin

class HomeView(LoginRequiredMixin, TemplateView):
    template_name="pages/home.html"

    def get_context_data(self):
        return {'pageTitle': 'Home Page'}
```

```bash
touch pages/urls.py
```

```python
### pages/urls.py
from django.urls import path
from .views import HomeView

urlpatterns = [
    path('', HomeView.as_view(), name="home")
]
```

```python
# _project/urls.py
from django.contrib import admin
from django.urls import path, include       #new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),       #new
]
```
 
If we visit http://localhost:8000/, we should now see our new home page rendered!
<div><blog-img src="home.png" alt="Home page" width="100%" height="auto" class="shadow mb-3"/></div>

## 4) Switching to Django Allauth

It is now time to switch to Allauth for email login! Let's install it and set it up:
```bash
pip install django-allauth
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
    'django.contrib.sites',		#new
    'django.contrib.staticfiles',
    #local
    'users',
    'pages',
    #third party
    'allauth',		#new
    'allauth.account',			#new
    'allauth.socialaccount',		#new
]

SITE_ID = 1		#new
ACCOUNT_AUTHENTICATION_METHOD = 'email'		#new
ACCOUNT_EMAIL_REQUIRED = True		#new
ACCOUNT_USERNAME_REQUIRED  = False		#new
ACCOUNT_EMAIL_VERIFICATION = 'none'		#new
LOGIN_REDIRECT_URL = 'home'     #new
...
AUTHENTICATION_BACKENDS = [		#new
    'django.contrib.auth.backends.ModelBackend',		#new
    'allauth.account.auth_backends.AuthenticationBackend',		#new
]		#new
...
```

```bash
from django.contrib import admin
from django.urls import path, include		#new

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),
    path('accounts/', include('allauth.urls')),		#new
]
```

We can now (using email and password) sign up a new user here http://localhost:8000/accounts/signup/, log them in here http://localhost:8000/accounts/login/ and log them out here http://localhost:8000/logout/:
<div><blog-img src="allauth_signup.png" alt="Signup page" width="100%" height="auto" class="shadow mb-3"/></div>

You can see here http://localhost:8000/accounts/ the different Allauth urls accessible (with DEBUG=True only):
<div><blog-img src="allauth_urls.png" alt="Allauth routes" width="100%" height="auto" class="shadow mb-3"/></div>

## 5) Setting up email verification on signup

For email verification, we need an email backend. We will not cover it in this article. We will instead fake an email backend by outputting the email in the console:
```python
### _project/settings.py

...
SITE_ID = 1
ACCOUNT_AUTHENTICATION_METHOD = 'email'
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED  = False
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'		#new
LOGIN_REDIRECT_URL = 'home'

EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'		#new
...
```

If we now sign up a new user **(make sure to enter a valid email)** and go to your console, we should see the confirmation link sent to that user in our console:
<div><blog-img src="email.png" alt="Email in console" width="100%" height="auto" class="shadow mb-3"/></div>

## 6) Custumizing Allauth templates

We will overide Allauth default template by creating our own templates inside `templates/account/` folder **(No "s" to account)**. We will also use Crispy-form to style our Allauth forms according to our Bootstrap style:
```bash
pip install django-crispy-forms
mkdir templates/account
touch templates/account/signup.html
touch templates/account/login.html
touch templates/account/logout.html
touch templates/account/email_comfirm.html
touch templates/account/verification_sent.html
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
    'django.contrib.sites',
    'django.contrib.staticfiles',
    #local
    'users',
    'pages',
    #third party
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'crispy_forms',     #new
]

CRISPY_TEMPLATE_PACK = 'bootstrap4'     #new
...
```

```html
### templates/account/signup.html
{% extends "_base.html" %}

{% load crispy_forms_tags %}

{% block title %}Sign Up{% endblock title %}

{% block content %}
<form method="post">
    {% csrf_token %}
    {{ form|crispy }}
    <button class="btn btn-primary" type="submit">Sign Up</button>
    <a class="btn btn-secondary" href="/accounts/login">Log In instead</a>
</form>
{% endblock content %}
```

```html
### templates/account/login.html
{% extends "_base.html" %}

{% load crispy_forms_tags %}

{% block title %}Log In{% endblock title %}

{% block content %}
<form method="post">
    {% csrf_token %}
    {{ form|crispy }}
    <button class="btn btn-primary" type="submit">Log In</button>
    <a class="btn btn-secondary" href="/accounts/signup">Sign Up instead</a>
</form>
{% endblock content %}

```

```html
### templates/account/logout.html
{% extends "_base.html" %}

{% load crispy_forms_tags %}

{% block title %}Log Out{% endblock title %}

{% block content %}
<form method="post">
    {% csrf_token %}
    {{ form|crispy }}
    <p>You are about to log out. Are you sure?</p>
    <button class="btn btn-danger" type="submit">Log Out</button>
    <a class="btn btn-secondary" href="/">Home</a>
</form>
{% endblock content %}
```

```html
### templates/account/verification_sent.html
{% extends "_base.html" %}

{% load crispy_forms_tags %}

{% block title %}Comfirm Your Email{% endblock title %}

{% block content %}
<p>We have sent you an email. Please follow the link provided to finalize the signup process</p>
{% endblock content %}
```

```html
### templates/account/email_confirm.html
{% extends "_base.html" %}

{% load account %}
{% load crispy_forms_tags %}

{% block title %}Email Confirmation{% endblock title %}

{% block content %}
<form method="post">
    {% csrf_token %}
    {{ form|crispy }}
    <p>Please confirm that <strong>{{ confirmation.email_address.email }}</strong> is your email address</p>
    <button class="btn btn-primary" type="submit">Comfirm</button>
</form>
{% endblock content %}
```

Allauth view should now be rendered with our new style:
<div><blog-img src="allauth_signup_custom.png" alt="Allauth custom signup template" width="100%" height="auto" class="shadow mb-3"/></div>

If you want to further customize these templates, I highly recommand you check the source code of Allauth to see how these templates are built (link below).

https://github.com/pennersr/django-allauth/tree/master/allauth/templates/account

## Conclusion

As you can see, Django Allauth makes email authentication super easy for us! Templates are also relatively easy to customize as well.

You can find the complete source code of the python project on my github https://github.com/florianbgt/django-allauth

If you have any question or just want to chat, feel free to email me <a href="mailto:florian.bigot321@gmail.com">florian.bigot321@gmail.com</a>