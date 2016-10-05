---
layout: post
title:  "Django Migrations Misunderstood"
date:   2016-05-02 12:23:05
categories: django engineering
---

At Paytm, we use [Django](https://www.djangoproject.com/), especially its [Admin](https://docs.djangoproject.com/en/1.10/ref/contrib/admin/) for operational purposes. We have models for many of our tables, access to which are controlled through Django Acl. Up till now, we used to use unmanaged models, which worked for most of the cases, since we needed to expose them on the admin panel. The reason for using unmanaged models is primarily because any alters on production or staging environment has to go through a DBA.
As we are growing at a very fast pace, this actually becomes cumbersome and tedious for a developer to sync his local database with production environment.
In order to solve this problem, we used django migrations and its settings concept. In this article, we will explore how django migrations solved our problem of syncing table structures across different environments.

### Migrations in Django

> Migrations are Django’s way of propagating changes you make to your models (adding a field, deleting a model, etc.) into your database schema - [Django Migrations](https://docs.djangoproject.com/en/1.10/topics/migrations/#module-django.db.migrations)

Django migration ships with a very useful feature namely [managed](https://docs.djangoproject.com/en/1.10/ref/models/options/#managed) flag. This flag decides whether the model will be managed by Django itself or not. Managing a model by Django means that django will apply any create/alter operation specific to this model when `migrate` is run. We however use unmanaged models, as all our alters/creations has to go through a DBA. As an example, a model with `managed` flag as False

```
class Test(models.Model):

    name = models.CharField(max_length=50)

    class Meta:
        managed = False

```

### Problem

Managing our tables by DBA saves us from the trouble of any dangerous alter or any stray table insert being run. But it also posses the problem for each developer to sync his development environment table structure with production or staging environment. One thing which actually helped us here, is that for most of the tables, we used to have its corresponding model, but nothing was achieved as its `managed` flag was _False_. Turning on the managed flag would apply all the alters for the table on development, but this comes with a cost. What if someone applies them on production. Also we need migration to be run on production, because that is how we can use Django Acl. Migration with managed flag as False when applied, used to give us the permission(add, edit and delete) for each model. Django provides us with another beautiful concept of [settings](https://docs.djangoproject.com/en/1.10/topics/settings/), which can be different for each environment. We initially thought of defining a boolean flag in each settings and then using it, like

```
class Test(models.Model):

    name = models.CharField(max_length=50)

    class Meta:
        managed = settings.MANAGED_FLAG
```

and in settings, like

```
# dev.py
MANAGED_FLAG = True

# live.py
MANAGED_FLAG = False

```

`MANAGED_FLAG` will be _True_ for development settings and _False_ for production settings. This would have worked wonderfully but it didn't. For each model, Django creates a migration file(which lives in migrations folder under the current app) by running `makemigrations`. This migration file will be created once and it will simply apply the value of `MANAGED_FLAG` from settings once. In more simpler terms, if these migrations are created on development environment, where `MANAGED_FLAG` is _True_, then the corresponding migration will have _True_, like

```
class Migration(migrations.Migration):

    dependencies = [
    ]

    operations = [
        migrations.CreateModel(
            name='Test',
            fields=[
                ('id', models.AutoField(primary_key=True, serialize=False, verbose_name='ID', auto_created=True)),
                ('name',  models.CharField(max_length=50)),
            ],
            options={
                'db_table': 'test',
                'managed': True
            },
        ),
    ]
```

One of the highly discouraged solution can be running migration by passing settings as argument like

```
python manange.py makemigrations --settings=settings.live
```

This solves the problem for production, but then again this wont apply any changes by running migrations on development environment since `managed` flag is _False_.

### How we solved this problem ?

We knew somewhere down the line, there must a dynamic hook, which is controlling how the migrations are run. And yes, we found [Database routers](https://docs.djangoproject.com/en/1.10/topics/db/multi-db/#automatic-database-routing) as a solution for this problem. Database routers are the easiest way to deal with multiple database. It works as a routing scheme and ensures that the object remain sticky to the original databases.

A database router has 4 function, out of which [allow_migrate](https://docs.djangoproject.com/en/1.10/topics/db/multi-db/#allow_migrate) saved us. `allow_migrate` takes as argument db, app_label, model_name as list arguments. It is called for each model in each app, which actually is logical, since one may have two model tied to to two different databases in the same app and it checks whether for a model of a particular app, are the migration allowed to run against `db`. `db` by default has `DEFAULT` as its value, which makes things more clear for a person not dealing with multiple databases.

So we decided that our hook for running migration and creating tables on development and not on production will live in `allow_migrate`. To make it more clear, we decided that for each model, its managed flag will be _True_. A flag will be exposed in `dev` and `live` settings which will specify whether the migrations are allowed to create tables or not.


### Our Solution Finally

To apply the above proposed solution, we figured out that, a common database router needs to written. Instead of the normal functionality of the database router where it returns the name of the database corresponding to some conditions like

```

if app_label == 'my_app' or model_name == 'my_special_model':
    return db == 'my_very_special_db'
```

we will simply check these type of condition from a map defined in settings. Hence we defined a variable named `DATABASE_ROUTER_MAPPING` in settings, which may look like

```
DATABASE_ROUTER_MAPPING = {

    # default db
    "admin" : "default",
    "auth" : "default",
    "contenttypes" : "default",
    "sites" : "default",
    "sessions" : "default",

    # book_store db
    "book": "book_store"
}
```

One very important thing that we observed here is that it is necessary to specify database for apps that are shipped with django like [auth](), [admin](), etc. If not specified, this will creates tables corresponding to these apps in the database, which is not required at all. So for example if we run

```
python manage.py migrate --database=book_store
```

then this will create table like `auth_user`, `django_content_types` etc in `book_store` database. To make this even more enforcing, we wrote a function which checks and throws error if any app has no entry in `DATABASE_ROUTER_MAPPING`.

```
def is_database_connection_in_settings(appname):

    if not hasattr(settings, 'DATABASE_ROUTER_MAPPING'):
        raise CommandError("DATABASE_ROUTER_MAPPING mapping missing from"
                           " settings")

    if appname.lower() in settings.DATABASE_ROUTER_MAPPING:
        return True
    else:
        raise CommandError("Database not specified for app {}. Please make"
                           " an entry in DATABASE_ROUTER_MAPPING in "
                           "settings".format(appname.lower()))
```

Now to maintain whether migrations are allowed to run or not, we defined a variable `ALLOW_MIGRATE_FALSE` per settings(_True_ in development and _False_ in production settings). A utility function for this looks something like

```
def is_migrate_allowed():

    '''
    If settings is having ALLOW_MIGRATE_FALSE flag and its value is false,
    then return false, else return none.
    dev settings has no such flag, hence will return None in that case
    live setting has this flag as `False` hence will return False
    '''

    if hasattr(settings, 'ALLOW_MIGRATE_FALSE') and settings.ALLOW_MIGRATE_FALSE == False:
        return False
    return None
```

Including all these, our `allow_migrate` looks like

```
def allow_migrate(self, db, app_label, model=None, **hints):
    '''
        gets managed flag value,
        if it is false, return false
        if it is None, means it is dev environment
    '''
    managed_flag = is_migrate_allowed()
    if managed_flag == False:
        return False

    if is_database_connection_in_settings(app_label):
        return db == settings.DATABASE_ROUTER_MAPPING[app_label]

    return None
```

The solution proposed till here, looks absolutely fine but there is one very important thing which is actually missing. We have disabled migrations on production for each and every app, which includes app likes `contenttypes` and `auth`. The end result of such a solution is that no default permission(add, edit and delete) will be created for any model now. The solution to this glitch is simple, we will  migrations to be run on production for some app(basically apps that we trust!). We defined a new variable `ALLOW_DB_MIGRATE` which will contain those database where we want migrations to be run and this variable will be defined in settings where we want migrations to run(production in our case).

```
ALLOW_DB_MIGRATE = {
    'default': True
}
```

However this involved updating both `is_migrate_allowed` and `allow_migrate`.

```
def is_migrate_allowed(db):

    '''
    If settings is having ALLOW_MIGRATE_FALSE flag and its value is false,
    then return false, else return none.
    dev settings has no such flag, hence will return None in that case
    live setting has this flag as `False` hence will return False
    '''

    if hasattr(settings, 'ALLOW_DB_MIGRATE') and db in settings.ALLOW_DB_MIGRATE:
        return settings.ALLOW_DB_MIGRATE[db]

    if hasattr(settings, 'ALLOW_MIGRATE_FALSE') and settings.ALLOW_MIGRATE_FALSE == False:
        return False
    return None


def allow_migrate(self, db, app_label, model=None, **hints):
    '''
        gets managed flag value,
        if it is false, return false
        if it is None, means it is dev environment
    '''
    managed_flag = is_migrate_allowed(db)
    if managed_flag == False:
        return False

    if is_database_connection_in_settings(app_label):
        return db == settings.DATABASE_ROUTER_MAPPING[app_label]

    return None

```

This solved our problem to a greater extent, but it brought a caveat with it. We now need to run migrations for each database like

```
python manage.py migrate --database=book_store
python manage.py migrate --database=default
```

This can be easily solved by running migrate through a management command for each database key specified in `DATABASES` defined in `settings`. We bundled the above solution and created it as a nice django package namely [supermigrate](https://github.com/paytm/django-supermigrate). Feel free to contribute, fork or simply use it.

