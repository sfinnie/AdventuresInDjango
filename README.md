# Adventures In Django

Notes, tips, starter code for working with Django.

## Starting a new project

```bash
$ cd ~/projects
$ mkdir my_project
$ cd my_project
$ python -m pip venv venv
$ . venv/bin/activate
$ (venv) python -m pip install -U pip
$ (venv) python -m pip install django
$ django-admin startproject my_project .
$ cp ~/projects/spikes/asset_health_django/.gitignore .
```

Now [create a custom user model](#custom-user-model).  Then:

```bash
$ python manage.py makemigrations
$ python manage.py migrate
$ winpty python manage.py createsuperuser
$ winpty python manage.py runserver # open browser at http://localhost:8000
<Ctrl-C>
$ 
```

## Centralising Templates and Static files at the project level

By default, Django looks for templates and static files (css, etc) in a subdir of each app.  That can be cumbersome.  Sharing templates across apps is simpler for small-medium sized projects.

```bash
$ cd ~/projects/my_project
$ mkdir templates
$ mkdir -p static/css
```

Then edit `my_project/settings.py` to update the TEMPLATES setting:

```python
TEMPLATES = [
    {
        # ...
        'DIRS': [BASE_DIR / "templates"], # new
        # ...
    },
]

#... bottom of file
STATIC_URL = "/static/" # new
STATICFILES_DIRS = [BASE_DIR / "static"] # new
```

Static files need the `{% load_static %}` directive included in templates (normally the base page template).

All project templates can then go into the shared `templates` dir.

## Adding an app

Django projects are composed of *apps*.  It's not possible to have a project without at least one app.  To add one:

```bash
$ python manage.py startapp my_app
```
Then edit `my_project/settings.py` to add the new app:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'my_app.apps.MyAppConfig',    # new
]      
```

<a name="custom-user-model"></a>

### Adding a Custom User Model

Do this by default.  There's little downside if it's not used, and a world of pain if you subsequently find out you need it.

```bash
$ python manage.py startapp accounts
```

Update `my_project/settings.py` to add the new app:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts.apps.AccountsConfig', # new
]  

#...
AUTH_USER_MODEL = 'accounts.CustomUser' # new

```

Update `accounts/models.py` to define the custom User model:

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    # AbstractUser already defines name, username, password etc.
    age = models.PositiveIntegerField(null=True, blank=True) # if nothing else required, replace with `pass`.
```

#### Editing Users in the Admin interface

Needs custom versions of the User Creation & Update forms.  In `accounts/forms.py`:

```python
from django.contrib.auth.forms import UserCreationForm, UserChangeForm

from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):

    class Meta(UserCreationForm):
        model = CustomUser
        fields = UserCreationForm.Meta.fields + ("age",)


class CustomUserChangeForm(UserChangeForm):

    class Meta:
        model = CustomUser
        fields = UserChangeForm.Meta.fields

```

Now update `accounts/admin.py`:

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .forms import CustomUserCreationForm, CustomUserChangeForm
from .models import CustomUser

class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = CustomUser
    list_display = [
        'email',
        'username',
    ]
    fieldsets = UserAdmin.fieldsets
    add_fieldsets = UserAdmin.add_fieldsets

admin.site.register(CustomUser, CustomUserAdmin)
```

### Anatomy of an App

All below are in the context of the `my_app` directory in of the project root

* `views.py`: defines the http handlers, including (where appropriate) rendering templates
* `urls.py`: defines routing
* `models.py`: contains the domain model classes


### Adding a Model

In `my_app/models.py`:

```python
from django.db import models

class MyModelClass(models.Model):
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.name
```

Available field types are listed [here](https://docs.djangoproject.com/en/4.1/ref/models/fields/#field-types), field options are [here](https://docs.djangoproject.com/en/4.1/ref/models/fields/#field-options).

It's good practice to include a `__str__` method as it's used by the `admin` app.

Any time a model is added or changed, the DB needs updated.

```bash
$ python manage.py makemigrations my_app
$ python manage.py migrate
```

### Registering models for the admin site

Edit `my_app/admin.py` as follows:

```python
from django.contrib import admin

from .models import MyModelClass

admin.site.register(MyModelClass)
```

### Preloading the database with data

See docs [here](https://docs.djangoproject.com/en/4.1/howto/initial-data/), in summary:

1. Create a *fixture*file in the `my_app/fixtures` directory, e.g. `fund_universe.json`:

```json

```




### Setting the admin site title

Add the following to `my_project/urls.py`:

```python
admin.site.site_header = "My Site Admin"
admin.site.site_title = "My Site Admin"
```

(Yes, it's a bit weird it goes into `urls.py`.  It'll be possible to move it, but it works).

### Customising the admin site

Read [this](https://realpython.com/customize-django-admin-python/).

## Accessing the admin site

Ensure superuser is set up first (see starting a project).
```bash
$ python manage.py runserver
```

Browse to [http://localhost:8000/admin](http://localhost:8000/admin).


## Adding a View & Template

There are two types of view: function-based and class-based.  There's a convenient template for class-based views that render templates.  In `my_app/views.py`:

```python
from django.views.generic import TemplateView

class HomePageView(TemplateView):
    template_name = 'home.html'
```

Edit `my_project/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('my_app.urls')),
]
```

And `my_app/urls.py`:

```python
from django.urls import path
from .views import HomePageView

urlpatterns = [
    path('', HomePageView.as_view(), name='home'),
]
```

Now create `templates/home.html`:

```html
{% load static %}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <meta http-equiv="X-UA-Compatible" content="ie=edge"/>
    <link rel="stylesheet" href="{% static 'css/my.css' %}"/>
    <title>My App</title>
</head>
<body>
    <h1>Homepage</h1>
</body>

```

Now run the server and check:

```bash
$ python manage.py runserver
```

And load [http://localhost:8000](http://localhost:8000)

## Template Composition

See the [documentation](https://docs.djangoproject.com/en/4.1/ref/templates/language/) for more.  Good examples.

### _base.html

The base template for all pages

```html
<!DOCTYPE html>

{% load static %}
<html lang="en">
<head>
    <title>{% block title %} Title {% endblock %}</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>

    <script src="https://cdn.jsdelivr.net/npm/echarts@5.4.0/dist/echarts.min.js"></script>

    <link href="{% static 'custom.css' %}" rel="stylesheet">

</head>
<body>
    <div id="content" class="row">
        {% block content %}
        {% endblock content %}
    </div>
</body>
</html>
```

There are two blocks that can be overridden by extending templates: `title` and `content`.  Title has a default value ("Title") that will be used if the extending template doesn't override it.  There's no default for `content`, so nothing will be rendered if the extending template doesn't define it.

### page.html

A sample page that extends `base.html` and includes a sub-component

```html
{%  extends "_base.html" %}

{% block title %} Home {% endblock %}

{% block content %}
    <div>
        {% include "_component.html" %}
    </div>
{% endblock content %}

<span>This {{ foo }} will be replaced with the value of the "foo" variable. </span>
```

### _component.html

A component fragment intended for embedding in other components or pages

```html
<h2>Asset Graph</h2>

<span>This is a sub-component</span>
```

