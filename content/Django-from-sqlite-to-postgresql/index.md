---
title: Django from sqlite to posgreSQL
description: Migrate an existing sqlite database to a postgreSQL database
image: "django.png"
writtenBy: "Florian Bigot"
---

In this tutorial, we will migrate the default django's sqlite database to a postgreSQL database.

You can find the source code of this article on [my github](https://github.com/florianbgt/Django-sqlite-to-posgreSQL)

## 1) Setting up our project

I am going to use an existing project from my github. You can do it as well if you do not have an existing project to migrate.

```bash
git clone https://github.com/florianbgt/Django-sqlite-to-posgreSQL.git
```

Then, we can make sure our project is working.

I am going to use Docker for this. However you can follow this tutorial without using it as well.

```bash
docker-compose up
```

Our container should now be up and running. We can go to http://localhost:8000/admin and login using the admin user (username: `admin` and password: `testpass123`).

Once login, you should see some existing reccords on the admin site.

<div><blog-img src="django-admin.png" alt="Django admin site with some data" width="100%" height="auto" class="shadow mb-3"/></div>

That is it, we now have a django project using a sqlite database!

## 2) Make a copy of our reccords

To migrate our data in the new database, we need first need to dump our data into a json file.

for this, we use the following command.

```bash
docker-compose run web python manage.py dumpdata db.json
```

On some OS, the file will be encoded with UTF-16. However, we need the file to be encoded with UTF-8 here. if it happen to you, you can simply open the file with notepad and save it using UTF-8 encoding.

## 3) Setup our new database

To make thing easier, I am going to use Docker to setup our postgreSQL database.

However, this step will work for any postgreSQL database.

To setup our new database in our Django project, just modify our `settings.py` as below.

```python
### _projects/settings.py
...
# DATABASES = {
#     'default': {
#         'ENGINE': 'django.db.backends.sqlite3',
#         'NAME': BASE_DIR / 'db.sqlite3',
#     }
# }

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

## 4) Migrate the new database and upload our data

First thing that we need to do with our new database is to apply our migrations in order to create the database schemas.

```bash
docker-compose run web python manage.py migrate
```

Then, we need to do exclude contenttypes data before we upload our data from the json file.

If we do not do this, we will end up with not so nice errors!

```bash
docker-compose run web python manage.py shell
>>> from django.contrib.contenttypes.models import ContentType
>>> ContentType.objects.all().delete()
>>> quit()
```

Once the command above done, we can finally upload our data.

Note that the following will put all the data in memory. Be sure that your computer is powerfull enough if you have a big dataset.

```bash
docker-compose run web python manage.py loaddata db.json
```

## 5) Check that our data are in the database

If we spin up our django container, we should be able to login using the same login as before (username: `admin`, password: `testpass123`) and our data should still be here.

<div><blog-img src="django-admin.png" alt="Django admin site with some data" width="100%" height="auto" class="shadow mb-3"/></div>

Congratulation, you have successfully migrated your data to the postgreSQL database!

## Conclusion

Changing database in an existing project is never recommanded in web development. However, it is sometime inevitable.

In some niche scenario, the commands we have done in this tutorial will not suffice.

If you face some problem try to follow this tutorial but replace `python manage.py dumpdata db.json` by `python manage.py dumpdata --exclude=contenttypes --exclude=auth.Permission > datadump.json`.

In my case, this command had worked with a complex django project that I had face problem with while migrating from sqlite to posgreSQL.

You can find the source code of this article on [my github](https://github.com/florianbgt/Django-sqlite-to-posgreSQL)

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com