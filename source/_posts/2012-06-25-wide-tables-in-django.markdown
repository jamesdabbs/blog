---
layout: post
title: "Wide Tables in Django"
date: 2012-06-25 00:05
comments: true
categories: 
- Django
- profiling
- hotshot
- dynamic models
---

In this post, I'll discuss an exploration I recently did examining how Django
handles the sort of wide tables that may arise from dynamically created models.

## Dynamic Models
I've recently been working on a Django project which requires an extensible 
user profile model: different organizations need to track different information
about users, and need to be able to configure that information on the fly. 
There are several ways to approach this, including 
[eav](http://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model) 
(presumably using [django-eav](https://github.com/mvpdev/django-eav)), but I 
wanted to consider a solution based on dynamically creating models - among
other considerations, having `ModelForms` "just work" would be a huge boon. I 
hope to further explore dynamic models in a future post, but for now I'll just 
summarize enough to tackle the question at hand: what kind of performance 
should we expect as columns are added and the Profile table gets wider? 
<!-- more --> For further reading on dynamic models, refer to Will Hardy's 
[dynamic-models repository](https://github.com/willhardy/dynamic-models) 
and [accompanying docs](http://dynamic-models.rtfd.org/).

### The Basics
Django's models aren't magic. Well, 
[they are](https://groups.google.com/forum/?fromgroups#!searchin/comp.lang.python/metaclasses$20are$20deeper$20magic/comp.lang.python/o61KSF77IWs/s09TqSEvTQgJ), 
but that magic is well encapsulated: Django checks your models module(s) for 
any subclasses of `django.db.models.Model` and maps each one to a table 
in the database. For each such model, every subclass of 
`django.db.models.Fields` attached to the model is mapped to a column in that 
table. The model then provides a host of utilities for moving conveniently 
between the python and database layers, but all it really needs is that 
mapping. So while you would typically define a `Profile` model by doing
something like 

``` python models.py 
from django.db import models

class Profile(models.Model):
    user = models.ForeignKey('User')
    birthday = models.DateField()
```

you could just as well use python's 
[type](http://docs.python.org/library/functions.html#type) function to re-write
that definition as

``` python models.py
from django.db import models

Profile = type('Profile', (models.Model,), {
    'user': models.ForeignKey('User'),
    'birthday': models.DateField(),
    '__module__': 'app.models'
})
```
(The `'__module__'` key is a technical requirement. 
[RTFD](http://dynamic-models.readthedocs.org/) for more info.)

The basic idea for a dynamically extensible `Profile`, then, is to define a 
`Column` model with a `to_field` method which creates a 
`django.db.models.Field` instance corresponding to the column in question. 
Then we can define

``` python models.py
...

attrs = {'__module__': 'app.models'}
for c in Column.objects.all():
    attrs[c.attr_name] = c.to_field()

Profile = type('Profile', (models.Model,), attrs)
```

There are a few potential pitfalls to be aware of, but that's a post for
another day. For now, we'll focus on the question at hand.

## Wide Tables
The (potential) problem is that as `Column` objects are created, the `Profile`
model becomes more complicated and we may incur some non-trivial and 
unnecessary overhead in instantiating each `Profile` object - especially if we 
only care about a small subset of the database columns at any point in time. In 
order to test just how significant those effects are, I've created a demo app 
and tested several different functions using the 
[hotshot](http://docs.python.org/library/hotshot.html) module. For the purpose
of these tests, consider the following models:

{% gist 2986726 %}

A copy of the project is available 
[on github](https://github.com/jamesdabbs/django-experiments/tree/master/wide_tables).
A few choices were made to deliberately reflect the project I intend to use a
dynamic Profile in - a PostgreSQL database, simple but extant model field
defaults, no more than 1000 columns, etc. Feel free to tweak the tests to suit
your needs and re-run them with `manage.py wide_table_benchmarks`.

Each method below was run 1000 times against each model. The full hotshot
profiling results are available
[in the repository](https://github.com/jamesdabbs/django-experiments/blob/master/wide_tables/benchmarks.txt),
but I'll summarize the highlights. All times below are given in milliseconds (ms).

### Saving
``` python
    model().save()
```

There isn't a whole lot to optimize here, but I had to create the objects
anyway to run the next tests, so I figured I may as well profile the initial
save and see where the time was being spent.

<table class="data">
  <tr>
    <th>Model</th>
    <th>Time</th>
    <th colspan=2>_commit</th>
    <th colspan=2>execute</th>
    <th colspan=2>__init__</th>
  </tr>
  <tr>
    <td>Small</td>
    <td>13.340</td>
    <td>10.77</td>
    <td>80.7%</td>
    <td>1.259</td>
    <td>9.44%</td>
    <td>0.162</td>
    <td>1.21%</td>
  </tr>
  <tr>
    <td>Medium</td>
    <td>12.791</td>
    <td>6.196</td>
    <td>48.4%</td>
    <td>2.092</td>
    <td>16.4%</td>
    <td>0.758</td>
    <td>5.93%</td>
  </tr>
  <tr>
    <td>Large</td>
    <td>46.894</td>
    <td>10.245</td>
    <td>21.8%</td>
    <td>16.251</td>
    <td>34.7%</td>
    <td>4.815</td>
    <td>10.3%</td>
  </tr>
</table>

Rather unsurprisingly, as the size of the model increases, the overhead of the 
default transaction behavior (`_commit`) becomes less significant, while the 
database calls themselves (`exectue`) and the model initialization (`__init__`) 
take up increasingly large portions of the total execution time. Somewhat more 
surprisingly, it actually took longer to save the `Small` model than the 
`Medium` model; that warrants future investigation.

### Retrieving

Django provides the 
[values](https://docs.djangoproject.com/en/dev/ref/models/querysets/#values) 
method to allow you to select from only a limited subset of columns. I tested 
the usual retrieval method:

``` python
    list(model.objects.all())  # list() forces evaluation of the QuerySet
```

against

``` python
    list(model.objects.all().values(<n columns>))
```

for varying numbers of columns.

<table class="data">
  <tr>
    <th>Model</th>
    <th>Full</th>
    <th>n=10</th>
    <th>n=100</th>
    <th>n=1000</th>
  </tr>
  <tr>
    <td>Small</td>
    <td>26.988</td>
    <td>14.962</td>
    <td>&mdash;</td>
    <td>&mdash;</td>
  </tr>
  <tr>
    <td>Medium</td>
    <td>111.755</td>
    <td>14.932</td>
    <td>69.681</td>
    <td>&mdash;</td>
  </tr>
  <tr>
    <td>Large</td>
    <td>855.156</td>
    <td>16.778</td>
    <td>67.085</td>
    <td>407.569</td>
  </tr>
</table>

As you may expect, the time it takes to retrieve and build an object
from the database grows fairly quickly with the number of columns, but if you
select a fixed number of columns, the time stays nearly flat. It's worth noting
that the time it takes to select _all_ values from an object (returning a
`dict`) is about half the time it takes to retrieve the actual model. This
suggests that

* If you don't need to use model methods on retreived data, you can mitigate
  the slowdown from wide tables by using `values`.
* If you don't need to use model methods, you should probably be using `values`
  no matter how wide your table is.

### Updating

Django 1.5 (in development as of this writing), 
[adds](https://docs.djangoproject.com/en/dev/releases/1.5/) an `update_fields` 
argument to models' `save` methods, which only updates the specified columns. 
This is useful in general, as it could help avoid some threading problems, but 
you'd definitely expect to see some speedups when updating a few columns in
a wide table. As above, I tested the usual

``` python
    obj.save()
```

against

``` python
    obj.save(update_fields=<n columns>)
```

for varying numbers of columns.

<table class="data">
  <tr>
    <th>Model</th>
    <th>Full</th>
    <th>n=10</th>
    <th>n=100</th>
    <th>n=1000</th>
  </tr>
  <tr>
    <td>Small</td>
    <td>12.434</td>
    <td>12.542</td>
    <td>&mdash;</td>
    <td>&mdash;</td>
  </tr>
  <tr>
    <td>Medium</td>
    <td>23.812</td>
    <td>12.924</td>
    <td>20.21</td>
    <td>&mdash;</td>
  </tr>
  <tr>
    <td>Large</td>
    <td>46.161</td>
    <td>23.628</td>
    <td>23.301</td>
    <td>41.381</td>
  </tr>
</table>

Here, unlike the retrieval test, the time taken for a full save and a save 
manually specifying each field is comparable (which makes sense, since they're
doing essentially the same thing). If you know for certain that you've only
updated a certain subset of fields, though, you can realize some significant
gains by using `update_fields`.

## In Closing ...

This experiment indicates that large dynamic models can perform reasonably well,
with judicious use of Django's lower-level database tools. It also suggests
some techniques that may be useful, regardless of how you're defining your
models:

* Use `values` if you only need an object's attributes, not a full model object
* Use the `update_fields` argument on a model's `save` function if you know
  that a certain subset of fields have been altered (e.g. if you're saving a 
  `ModelForm` that has defined a `fields` attribute)
  
Hopefully, you've also gotten some ideas about how you can approach questions 
that come up in your work. If you have any questions, comments, or suggestions 
for future topics, chime in below.
