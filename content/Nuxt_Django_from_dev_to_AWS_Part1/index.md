---
title: Nuxt Django app from developement to AWS - Part 1
description: Deploy a Django Nuxt application to AWS using RDS, S3 and ECS services. Part 1, prepare the app for production using RDS, S3 and traefik
image: "aws.png"
writtenBy: "Florian Bigot"
---

In this tutorial, we will deploy a production ready app to the cloud using AWS ECS

This is the part 1 of the tutorial where we will prepare our app for production using RDS, S3 and traefik.

[Part 2 here](https://blog.florianbgt.com/Nuxt_Django_from_dev_to_AWS_Part2)

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-ECS)

## 1) Setting up our project

Will use an existing project as a base for this tutorial.
You can find it [here](https://github.com/florianbgt/Nuxt-Django-REST-Docker)

This project is a Nuxt app and a Django REST API.
I have a made a 2 part tutorial where we build this application.

- [Part 1 here](https://blog.florianbgt.com/Nuxt_Django_REST_Docker_Part1)
- [Part 2 here](https://blog.florianbgt.com/Nuxt_Django_REST_Docker_Part2)

Let's start by cloning the project.

```bash
git clone https://github.com/florianbgt/Nuxt-Django-REST-Docker
```

One the project cloned, we can run docker compose to make the database migrations and create a superuser to access Django's admin page

```bash
docker-compose run api python manage.py migrate
docker-compose run api python manage.py createsuperuser
```

We then can spin up our containers using `docker-compose up` and the app should be up and running!

- frontend accessible at http://localhost:3000/
- backend accessible at http://localhost:8000/

<div><blog-img src="app.png" alt="Nuxt app landing page" width="100%" height="auto" class="shadow mb-3"/></div>

## 2) Using RDS instead of our dockerized database

For production, we are going to use AWS RDS database.

For this start to login into AWS console and go to `services > Database > RDS > Create database`

Then, we use the following settings and create the database.

<div><blog-img src="RDS-settings_1.png" alt="RDS settings 1/3" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="RDS-settings_2.png" alt="RDS settings 2/3" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="RDS-settings_3.png" alt="RDS settings 3/3" width="100%" height="auto" class="shadow mb-3"/></div>

We now need to tell Django to use our AWS database. However, we will not expose the database information in the code. instead, we will environement variables.

We start by creating a `.env` file to keep our environement variables.

```bash
touch .env
```

```txt
### .env
USE_RDS=1
RDS_NAME=prod_db
RDS_USER=postgres
RDS_PASSWORD=<your password>
RDS_HOST=<db endpoint>
RDS_PORT=5432
```

We then modify our `settings.py` to include RDS database.

```python
### api/_project/settings.py
...
if bool(int(os.environ.get('USE_RDS'))):
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': os.environ.get('RDS_NAME'),
            'USER': os.environ.get('RDS_USER'),
            'PASSWORD': os.environ.get('RDS_PASSWORD'),
            'HOST': os.environ.get('RDS_HOST'),
            'PORT': 5432,
        }
    }
else:
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

Finaly, we modify our `docker-compose.yml` to include our environement variables file.

```yaml
### docker-compose.yml
...
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
...
```

As we are now using a new database, we need to apply our migrations, recreate a superuser again.

```bash
docker-compose run api python manage.py migrate
docker-compose run api python manage.py createsuperuser
```

We now can spin up our container using `docker-compose up` and now use our app with the RDS database!

## 3) Using S3 to store our media files

For production, we are going to use AWS S3 to store and retrieve our media files.

For this start to login into AWS console and go to `services > Storage > S3 > Create bucket`

Then, we use the following settings and create the S3 bucket.

<div><blog-img src="S3-settings_1.png" alt="S3 settings" width="100%" height="auto" class="shadow mb-3"/></div>

To access our bucket, we then need to create a user and give it the right to edit the bucket.

We are doing it through the AWS IAM.

Go to `services > Security, Identity, & Compliance > IAM > Policies > Create Policy`

<div><blog-img src="IAM-create_policy_1.png" alt="IAM policy 1/2" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="IAM-create_policy_2.png" alt="IAM policy 2/2" width="100%" height="auto" class="shadow mb-3"/></div>

Then, go to `services > Security, Identity, & Compliance > IAM > Users > Add users`

<div><blog-img src="IAM-create_user_1.png" alt="IAM user 1/2" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="IAM-create_user_2.png" alt="IAM user 2/2" width="100%" height="auto" class="shadow mb-3"/></div>

Once our user created, we can store the access key ID and secret access key inside our environement variables.

```txt
### .env
USE_RDS=1
RDS_NAME=prod_db
RDS_USER=postgres
RDS_PASSWORD=<your password>
RDS_HOST=<db endpoint>
RDS_PORT=5432

USE_S3=1
AWS_ACCESS_ID=<your access id>
AWS_ACCESS_SECRET=<your access secret>
AWS_S3_BUCKET_NAME=awesome-recipe-bucket-1
AWS_S3_BUCKET_REGION=us-east-2
```

To setup S3 bucket in our Django project, we are going to use the `django-storages` library.

We include this library in our `requirements.txt`.

```txt
### api/requirements.txt
Django==3.2
psycopg2-binary==2.9.1
djangorestframework==3.12.4
djangorestframework-simplejwt==4.7.2
django-rest-passwordreset==1.2.0
django-cleanup==5.2.0
Pillow==8.3.1
django-cors-headers==3.8.0
django-storages==1.11.1
```

We then modify our `settings.py` to include S3.

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
    # local
    'users',
    'recipes',
    # 3rd party
    'rest_framework',
    'django_rest_passwordreset',
    'corsheaders',
    'storages'
    'django_cleanup',   #need to be last
]
...
MEDIA_URL = '/api/media/'
if bool(int(os.environ.get('USE_S3'))):
    AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_ID')
    AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_ACCESS_SECRET')
    AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_S3_BUCKET_NAME')
    AWS_S3_REGION_NAME = os.environ.get('AWS_S3_BUCKET_REGION')
    AWS_S3_SIGNATURE_VERSION = 's3v4'
    AWS_S3_FILE_OVERWRITE = False
    AWS_S3_ADDRESSING_STYLE = 'virtual'
    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
else:
    MEDIA_ROOT = os.path.join(BASE_DIR.parent, 'media').replace('\\', '/')
...
```

We now can rebuild our container using `docker-compose up --build` and our app is now using AWS S3 to store media files!

## 4) Setting up traefik

## Conclusion