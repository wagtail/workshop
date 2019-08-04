# Wagtail workshop

# Installation
1. `virtualenv --python=python3 wagtailenv`
2. `source wagtailenv/bin/activate`
    - On Windows, the equivalent activate script is in the Scripts folder:
    - `> wagtailenv\Scripts\activate`
3. `pip install --upgrade pip`
4. `pip install wagtail`
5. `wagtail start workshop`
6. `cd workshop`
7. `python manage.py migrate`
8. `python manage.py runserver 0:8000`
9. make a new terminal tab
10. `python manage.py createsuperuser`
11. Go to http://localhost:8000/
12. Click the Admin Interface link or go directly to https://localhost:8000/admin/
 
# Edit the homepage model
## Add a body field

```python
# home/models.py
from wagtail.core.models import Page
from wagtail.core.fields import RichTextField
from wagtail.admin.edit_handlers import FieldPanel
    
    
class HomePage(Page):
    body = RichTextField(blank=True)
```

1. `python manage.py makemigrations`
2. `python manage.py migrate`

## Make the body field editable
```python
class HomePage(Page):
    body = RichTextField(blank=True)
    
    content_panels = Page.content_panels + [
        FieldPanel('body', classname="full"),
    ]
```

# Update the template

Edit `home/templates/home/home_page.html`

```python
{% extends "base.html" %}
{% load wagtailcore_tags %}
    
{% block content %}
    <h1>{{ self.title }}</h1>
    {{ page.body|richtext }}
{% endblock %}
```

# Create a blog
## Define the model for the blog index page

1. `python manage.py startapp blog`
2. add `blog` to `INSTALLED_APPS` in `workshop/settings/base.py`
3. Edit `blog/models.py`:

```python  
# blog/models.py
from wagtail.core.models import Page
from wagtail.core.fields import RichTextField
from wagtail.admin.edit_handlers import FieldPanel
    
    
class BlogIndexPage(Page):
    intro = RichTextField(blank=True)
    
    content_panels = Page.content_panels + [
        FieldPanel('intro', classname="full")
    ]
```

4. `python manage.py makemigrations` and `python manage.py migrate`

## Create a template for the blog index

Make a file at `blog/templates/blog/blog_index_page.html` with this content:

```html
{% extends "base.html" %}
{% load wagtailcore_tags %}
    
{% block content %}
    <h1>{{ page.title }}</h1>
    <div class="intro">{{ page.intro|richtext }}</div>
    {% for post in page.get_children %}
        <h2><a href="{% pageurl post %}">{{ post.title }}</a></h2>
        {{ post.first_published_at }}
    {% endfor %}
{% endblock %}
```

(Restart the server if your template isn’t found.)

## Create a model for blog posts

In `blog/models.py`:

```python
from django.db import models
    
from wagtail.core.models import Page
from wagtail.core.fields import RichTextField
from wagtail.admin.edit_handlers import FieldPanel
  
# Keep the definition of BlogIndexPage, and add:
    
class BlogPage(Page):
    intro = models.CharField(max_length=250)
    body = RichTextField(blank=True)
    
    content_panels = Page.content_panels + [
        FieldPanel('intro'),
        FieldPanel('body', classname="full"),
    ]
```

`python manage.py makemigrations` and `python manage.py migrate`

## Create a template for blog posts

Make a file at `blog/templates/blog/blog_page.html` with this content:

```html
{% extends "base.html" %}
{% load wagtailcore_tags %}
    
{% block content %}
    <h1>{{ page.title }}</h1>
    <p class="meta">{{ page.first_published_at }}</p>    
    <div class="intro">{{ page.intro }}</div>
    {{ page.body|richtext }}
    <p><a href="{{ page.get_parent.url }}">Return to blog</a></p>
{% endblock %}
```

## Improve the blog listing

Posts should be in reverse chronological order, and we should only list published content. Edit `blog/models.py`, adding this `get_context` method to the `BlogIndexPage` class:

```python
class BlogIndexPage(Page):
    # .. pre-existing fields 

    def get_context(self, request):
        context = super().get_context(request)
        live_blogpages = self.get_children().live()
        context['blogpages'] = live_blogpages.order_by('-first_published_at')
        return context
```

and update your blog index template to loop over `blogpages` instead of `page.get_children`.

## Add an image to your blog post model
    
```python
# blog/models.py
# Add this to your imports
from wagtail.images.edit_handlers import ImageChooserPanel

# Add this to your in your BlogPage class
    image = models.ForeignKey(
        'wagtailimages.Image',
        null=True,
        blank=True,
        on_delete=models.SET_NULL
    )
        
# Add this to your content_panels for BlogPage
ImageChooserPanel('image'),
```

`python manage.py makemigrations` and `python manage.py migrate`

## Update the blog post template to output images

Open up your `blog/templates/blog/blog_page.html` template and make the following edits:

```html
{% extends "base.html" %}
{% load wagtailcore_tags wagtailimages_tags %}
    
{% block content %}
    <h1>{{ page.title }}</h1>
    <p class="meta">{{ page.first_published_at }}</p>
    {% image page.image fill-320x320 %}
    <div class="intro">{{ page.intro }}</div>
    {{ page.body|richtext }}
    <p><a href="{{ page.get_parent.url }}">Return to blog</a></p>
{% endblock %}
```

`fill` is just one of Wagtail's image resizing methods. Details of the others are listed in [the docs](http://docs.wagtail.io/en/latest/topics/images.html). Try them out!

# Basic styling

Add Milligram to `workshop/templates/base.html`, before `{# Global stylesheets #}`:

```html
<!-- CSS Reset -->
<link rel="stylesheet" href="//cdn.rawgit.com/necolas/normalize.css/master/normalize.css">
<!-- Milligram CSS minified -->
<link rel="stylesheet" href="//cdn.rawgit.com/milligram/milligram/master/dist/milligram.min.css">
```

Wrap the content block in a container div:

```html
<div class="container">
  {% block content %}{% endblock %}
</div>
```

And add some margins to `static/css/workshop.css`:

```css
.container {
    padding-top: 4rem;
}
    
h2 {
    margin-bottom: 0rem;
    margin-top: 2rem;
}
```
# Deploy

Install wagtail-bakery with 
```
pip install --upgrade git+https://github.com/moorinteractive/wagtail-bakery
```
Then add `bakery` and `wagtailbakery` to your `INSTALLED_APPS`.

Make sure your `setuptools` is up to date: `pip install -U setuptools`.

Edit `workshop/settings/base.py`:

```python
BUILD_DIR = '/tmp/build/'
BAKERY_VIEWS = ('wagtailbakery.views.AllPublishedPagesView',)
```

Build your pages:

```
python manage.py build
```

Install netlify using `npm`. You will need to have [Node.js](https://nodejs.org/en/) installed on your computer.

```
npm install netlify-cli -g
```

Once `netlify-cli` is installed, we need to login in via the command line tool. To do that, run `netlify login`. It'll take you to the login page. If you don't have a Netlify account, now is the time to create a free netlify account.

Once you're logged in you can type `netlify status` to verify your credentials are working in your command line tool. 

Next you'll need to run the `netlify init` command to start it inside our project and select the "Yes, create and deploy site manually" option. Then select your team. Then provide a custom subdomain URL (optional, and changeable later).

You can do a test deploy with a staging URL by using:

```
netlify deploy --dir=/tmp/build
```

Henceforth:

```
python manage.py build && netlify deploy --dir=/tmp/build
```

And lastly, to officially deploy your site to your Netlify subdomain, run this final command:
```
python manage.py build && netlify deploy --prod --dir=/tmp/build/
```

## Deploy automatically

In `models.py`:

```python
from wagtail.core.signals import page_published
from django.core.management import call_command
from subprocess import Popen

def deploy(sender, **kwargs):
    call_command('build')
    Popen(['netlify', 'deploy'])

page_published.connect(deploy)
```

# StreamField

To convert the body field of your blog post from a rich text field to a StreamField, update `blog/models.py`

```python
# to your imports, add:

from wagtail.core.fields import StreamField
from wagtail.core import blocks
from wagtail.admin.edit_handlers import StreamFieldPanel
from wagtail.embeds.blocks import EmbedBlock
  
# convert your BlogPage's body to a StreamField:
body = StreamField([
    ('heading', blocks.CharBlock(classname="full title", icon="title")),
    ('paragraph', blocks.RichTextBlock(icon="pilcrow")),
    ('embed', EmbedBlock(icon="media")),
])
 
# and, in content_panels, convert BlogPage's FieldPanel into a StreamFieldPanel:
StreamFieldPanel('body')
```

Update your blog page template to output the new field type, replacing `{{ page.body|richtext }}` with

```html
{% for child in self.body %} 
    {% if child.block_type == 'heading' %}
        <h2>{{ child }}</h2>
    {% else %}
        {{ child }}
    {% endif %} 
{% endfor %}
```

Note the [docs for responsive embeds](http://docs.wagtail.io/en/latest/topics/writing_templates.html?highlight=responsive#responsive-embeds).

# Third party apps
## Awesome Wagtail

https://github.com/springload/awesome-wagtail

## Wagtail menus

Follow the installation instructions at [wagtailmenus.readthedocs.io](http://wagtailmenus.readthedocs.io/en/latest/installation.html).

Basic menu styling:

```css
.menu ul {
    list-style-type: none;
    margin: 0;
    padding: 0;
    overflow: hidden;
}

.menu li {
    float: left;
}

.menu li a {
    display: block;
    color: black;
    padding-left: 0px;
    padding-right: 16px;
    text-decoration: none;
}
```
