Django Skeleton Project
================

####*A clean project of a Django web-framework running on Heroku with static files served from Amazon S3, attempting to follow the [12-factor](http://www.12factor.net/) design pattern.*

>**Django** is a web framework that simplifies the work required to make a web app in python.
>
>**Heroku** is touted as one of the best 'platform as a service' providers. You can host apps >without having to get too involved with installing your own serving software etc.
>
>**Amazon S3** is a quick, cheap way to host static files (images, js, css etc. that doesn't >change) because heroku doesn't do that.>

This collection is touted as one fo the best ways to serve up django apps, that's free initially and can scale up easily. This repo gets it up and running quickly and securely.

### Pre-requisites
This setup using the excellent virtualenvwrapper to isolate the installed dependencies and environmental variables.

- heroku account and [toolbelt](https://toolbelt.heroku.com/) installed on your computer
- git installed
- [virtualenv](https://pypi.python.org/pypi/virtualenv) and [virtualenvwrapper](https://bitbucket.org/dhellmann/virtualenvwrapper)
- an Amazon Web Services account

## Installation

1. Make a new virtualenv and clone this repo into it
        
    ```sh
    mkvirtualenv [name-of-your-project]
    git clone https://github.com/jordn/heroku-django-s3 [name-of-your-project]
    ```
2. Install all the dependencies (django, psycopg2, gunicorn, dj-database-url, boto and django-storages) with pip. These are specified in requirements.txt (if you edit this file to remove the version numbers it will install the latest versions available)

    ```sh
    pip install -r requirements.txt
    ```

3. All the private or environment-dependant settings in `settings.py` are kept as environmental variables.
    We need to set these variables everytime we enter this virtual environment. virtualenvwrapper does this with a `postactivate` script. Edit this file:

    ```sh
    vim $VIRTUAL_ENV/bin/postactivate 
    ```

    Add the following variables:

    ```sh
    #!/bin/zsh
    # This hook is run after this virtualenv is activated.
    # Django database
    export DATABASE_URL=sqlite:////[path to whereever you'd like to store the sqlite (easiest) database for local dev]
    # Django static file storage
    export AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
    export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXX-XXXXXXXXXX
    # Django debug setting
    export DJ_DEBUG=True
    export DJ_SECRET_KEY=[Any random sequence of 40ish characters  - django uses it for added security]
    ```

4. Reopen the virtualenv to run this script

    ```sh
    workon [name-of-your-project]
    ```

5. Everything should now work for **local development** to check we can see the admin pages at `http://127.0.0.1:8000/admin/`

    ```sh
    python manage.py syncdb
    ...
    python manage.py runserver
    ```


6. Create the app on heroku and push everything there. Heroku will detect it's a python app and install everything in requirements.txt (update this file with `pip freeze > requirements.txt`)

    ```sh
    heroku create [name-of-your-project]
    ...
    git push heroku master
    ```

7. The heroku django won't work without the environmental variables being set (`DATABASE_URL` is already set on heroku):

    ```sh
    heroku config:add AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
    heroku config:add AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    heroku config:add AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXX-XXXXXXXXXX
    heroku config:add DJ_SECRET_KEY=[Any random sequence of around 40 characters django uses for added security]
    ```

    You can turn debug on/off by changing the DJ_DEBUG setting (only do something has gone wrong. *Note: static files aren't served from S3 in debug mode*):

    ```sh
    heroku config:add DJ_DEBUG=True
    ``` 

8. Should now be ready. Go and build that web app!

    ```sh
    python manage.py syncdb
    ...
    heroku open
    ```

    Lastly, go into settings.py and change the trusted hosts to be specifically your domains (for added security)

    ```python
    ALLOWED_HOSTS = ['[your-project-name].herokuapp.com']
    ```



## Differences from a fresh django install
    
####Programs installed
 - django
 - psycopg2 (to be able to talk to postgreSQL databases)
 - gunicorn (python HTTP server to use from heroku)
 - [dj-database-url](https://github.com/kennethreitz/dj-database-url) (to use a URL environmental variable to reference the location of the database)
 - [django-storages](http://django-storages.readthedocs.org/en/latest/backends/amazon-S3.html) (custom storage backends for django, best S3 support makes use of 'boto'...)
 - Boto (Python interface to Amazon Web Services, simplifies the AWS connection to just the access keys)

####Changes Made

####`settings.py`

>Removed `# 'django.contrib.sites'` add-on that can enable multiple sites to use the same back->end but confuses matters here.
>
>As part of the plan to make the settings.py file to be transferable and secure did the >following:
>
>Make `DEBUG` an environmental variable by putting the following at the top of settings.py:
>
>```python
>import os 
>
># Added to help use env variables
>def env_var(key, default=None):
>    """Retrieves env vars and makes Python boolean replacements"""
>    val = os.environ.get(key, default)
>    if val == 'True':
>        val = True
>    elif val == 'False':
>        val = False
>    return val
>
>DEBUG = env_var('DJ_DEBUG', False) #Unless env var is set to True, debug is off
>```
>
>`SECRET_KEY` stored as as `os.environ['DJ_SECRET_KEY']` environmental variable.
>```
>
>`ALLOWED_HOSTS` accepting any `['.herokuapp.com']` subdomain. Only these domains can host the >site when `DEBUG = False`. User should **remember to change this to be most specific**
>
>Database settings are set by env var:
>    
>```python   
># Parse database configuration from $DATABASE_URL
>import dj_database_url
>DATABASES['default'] =  dj_database_url.config(default=os.environ.get('DATABASE_URL'))
>
># Honor the 'X-Forwarded-Proto' header for request.is_secure()
>SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
>```
>
>`'admin'` and `'admindocs'` enabled  and `'storages'` to `INSTALLED_APPS`
>
>Added settings to store statics on S3
>    
>```python
>#Storage on S3 settings are stored as os.environs to keep settings.py clean 
>if not DEBUG:
>    AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
>    AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
>    AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
>    STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
>    S3_URL = 'http://%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
>    STATIC_URL = S3_URL
>```

####`urls.py`
>Uncommented lines 4, 5, 13 and 16 from urls to enable admin urls

####`Procfile`
> Runs gunicorn process for heroku

####`.gitignore`
> Ignores common ignorables for python and django development
