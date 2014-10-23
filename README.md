Django Shopify Auth
===================

This Django package makes it easy to integrate Shopify authentication into your Django app. It shares some similarities
with the [shopify_django_app](https://github.com/Shopify/shopify_django_app) project, but with a couple of key
differences:


* It provides a custom Django Authentication scheme based on `AbstractBaseUser` and `RemoteUserBackend`, meaning shops
  will be authenticated as "users" of your Django app. This makes it easier to use common Django patterns and libraries
  (such as accessing the currently authenticated store as `request.user`).

* It persists users' Shopify access tokens in the database, rather than in the Session, meaning your app will be able
  to make API calls on behalf of a user when they're not logged in.

* It supports the authentication flow for new-style "Embedded SDK" Shopify apps.


This project provides one package, `shopify_auth`.
A demonstration Django project using this package is available [here](https://github.com/discolabs/auth_demo).


Requirements
------------

Django v1.7 or higher is required.

As with the original `shopify_django_app` package, you'll need a [Shopify partner account](http://shopify.com/partners)
and to have created an app in order to get an API key and secret.


Installation & Setup
--------------------

There are a few moving parts to set up, but hopefully the following instructions will make things straightforward.

We're assuming in this setup that you're using a standard Django project layout (the sort that's created with the
`django-admin.py startproject` command). We're also assuming that our project is called `auth_demo` and that the primary
Django app inside our project is going to be called `auth_app`.

If you ever get lost or aren't really sure what to do, you can refer to the [demo app](https://github.com/discolabs/auth_demo).


### 1. Install package

Installation is super easy via `pip`:

```shell
> pip install django-shopify-auth
```

Once you have the package installed, add `shopify_auth` to your `INSTALLED_APPS`.


### 2. Add custom user model

Because `shopify_auth` makes use of Django's authentication system, it provides a custom authentication backend
(`shopify_auth.backends.ShopUserBackend`) which allows authentication through Shopify's OAuth flow.

This backend requires that the user model for your app (specified by `AUTH_USER_MODEL` in your `settings.py`) inherits
from `shopify_auth.models.AbstractShopUser`. To do this, just add something like this to the `models.py` for your
Django app:

```python
# auth_demo/auth_app/models.py
from shopify_auth.models import AbstractShopUser

class AuthAppShopUser(AbstractShopUser):
    pass
```

Then make sure that you have the following line or similar in `settings.py`:

```python
AUTH_USER_MODEL = 'auth_app.AuthAppShopUser'
```


### 3. Configure settings

In addition to setting `AUTH_USER_MODEL`, there are a few more required additions to `settings.py`:

```python
# Configure Shopify Application settings
SHOPIFY_APP_NAME = 'Your App Name'
SHOPIFY_APP_API_KEY = os.environ.get('SHOPIFY_APP_API_KEY')
SHOPIFY_APP_API_SECRET = os.environ.get('SHOPIFY_APP_API_SECRET')
SHOPIFY_APP_API_SCOPE = ['read_products', 'read_orders']
SHOPIFY_APP_IS_EMBEDDED = True
SHOPIFY_APP_DEV_MODE = False

# Use the Shopify Auth authentication backend as the sole authentication backend.
AUTHENTICATION_BACKENDS = (
    'shopify_auth.backends.ShopUserBackend',
)

# Add the Shopify Auth template context processor to the list of processors.
# Note that this assumes you've defined TEMPLATE_CONTEXT_PROCESSORS earlier in your settings.
TEMPLATE_CONTEXT_PROCESSORS += (
    'shopify_auth.context_processors.shopify_auth',
)

# Use the Shopify Auth user model.
AUTH_USER_MODEL = 'auth_app.AuthAppShopUser'

# Set the login redirect URL to the "home" page for your app (where to go after logging on).
LOGIN_REDIRECT_URL = '/'
```

Note that in the example above, the application API key and API secret are pulled from environment settings, which is a
best practice for Django apps that helps avoid the accidental check-in of sensitive information to source files.

Set `SHOPIFY_APP_IS_EMBEDDED` to `True` if your app has been configured as an Embedded app (you choose this option at
the time of app creation). Setting this will make the app provide a Javascript-based redirect that breaks out of an
embedded app's `<iframe>` during the authentication flow as per [the Shopify documentation](http://docs.shopify.com/embedded-app-sdk).
If `SHOPIFY_APP_IS_EMBEDDED` is `False`, the normal authentication flow for non-Embedded apps will be used.

Setting `SHOPIFY_APP_DEV_MODE` to `True` allows you to test your apps locally by skipping the external OAuth phase for
your app. As it means you can log into your app as any store, you should obviously ***never*** set this to `True` in
production.

Now that all of the settings are configured, you can run `migrate` to set up the database for your new user model:

```shell
> python manage.py migrate
```


### 4. Configure URL mappings

Include `shopify_auth` URLs in your project's `urls.py`:

```python
# urls.py
from django.conf.urls import patterns, include, url

urlpatterns = patterns('',
    url(r'login/', include('shopify_auth.urls')),

    # ... remaining configuration here ...
)

```


### 5. Create application views

Now that you've gotten the configuration out of the way, you can start building your application.

All views inside your application should be decorated with `@login_required`.
This decorator will check that a user has authenticated through the Shopify OAuth flow.
If they haven't, they'll be redirected to the login screen.

```python
from django.shortcuts import render
from shopify_auth.decorators import login_required

@login_required
def home(request, *args, **kwargs):
    return render(request, "my_app/home.html")
```


### 6. Using the Embedded App SDK

If you're using the Embedded App SDK, be aware that the HTML your views return must contained some Javascript in the
`<head>` to properly frame your app within the Shopify Admin.

Generally, all pages you'd like embedded in the Shopify Admin should contain something like this in `<head>`:

```html
<script type="text/javascript" src="https://cdn.shopify.com/s/assets/external/app.js"></script>
<script type="text/javascript">
    ShopifyApp.init({
        apiKey: '{{ SHOPIFY_APP_API_KEY }}',
        shopOrigin: 'https://{{ user.myshopify_domain }}'
    });
    ShopifyApp.ready(function() {
        ShopifyApp.Bar.initialize({
            title: '{{ SHOPIFY_APP_NAME }}',
            buttons: {}
        });
    });
</script>
```


### 7. Making Shopify API calls

To make Shopify API calls on behalf of a user, we can use the user's `session` property inside a `with` statement:

```python
def view(request, *args, **kwargs):

    # Get a list of the user's products.
    with request.user.session:
        products = shopify.Product.find()

    # ... remaining view code ...
```

Behind the scenes, using `with request.user.session` sets up a temporary Shopify API session using the OAuth token we
obtained for that specific user during authentication.

All code wrapped within the `with` statement is executed in the context of the specified user. You should always wrap
calls to the Shopify API using this pattern.


Questions or Problems?
----------------------

Browse through the code for the demo app:
<https://github.com/discolabs/auth_demo>

Read up on the possible API calls:
<http://api.shopify.com>

Learn how to use the `shopify_python_api` library:
<http://wiki.shopify.com/Using_the_shopify_python_api>

Ask technical questions on Stack Overflow:
<http://stackoverflow.com/questions/tagged/shopify>

Email me:
[gavin@gavinballard.com](mailto:gavin@gavinballard.com)
