---
layout: post
title: "Django on Heroku - db:pull"
date: 2012-07-02 10:57
comments: true
categories:
- Django
- Heroku
- db:pull
- pgbackups
- custom management commands
---

[Heroku](http://www.heroku.com) has
[officially supported Django](https://devcenter.heroku.com/articles/django) for
a while now and although Heroku is awesome, there are still a few pain points.
One in particular that's been causing me problems is `heroku db:pull` - it's a
great convenience method, but it can be fragile and slow, especially on larger
data sets.

Heroku offers another convenient way to move data around (assuming you're using
Postgres): [pgbackups](https://devcenter.heroku.com/articles/pgbackups). Here's
the basic process: <!--more-->

``` bash
    # Activate the pgbackups addon
    $ heroku addons:add pgbackups

    # Create a backup
    $ heroku pgbackups:capture

    # pgbackups:url temporarily exposes a url for accessing a captured database
    # The capture can be downloaded using curl
    $ curl -o latest.dump `heroku pgbackups:url`

    # Restore from the dump
    $ pg_restore --verbose --clean --no-acl --no-owner -h myhost -U myuser -d mydb latest.dump
```

Not bad, but certainly a lot more verbose than `heroku db:pull`. If you find
yourself doing it often, you should definitely automate.

## Custom Management Commands
Django allows you to
[define custom management commands](https://docs.djangoproject.com/en/dev/howto/custom-management-commands/),
so it's fairly simple to include a command to replace

``` bash
    $ heroku db:pull
```

with

``` bash
    $ ./manage.py db:pull
```

Here's the command I'm using. Obviously, you may want to tweak some of
the options to suit your needs:

``` python management/commands/db:pull.py
    from optparse import make_option
    import os
    import subprocess
    import urllib2

    from django.conf import settings
    from django.core.management.base import BaseCommand


    class Command(BaseCommand):
        help = 'Replaces `heroku db:pull` by using `pgbackups` to import data'
        option_list = BaseCommand.option_list + (
            # Options for this management command
            make_option('-x',
                action='store_true',
                dest='delete',
                default=False,
                help='Delete the pg_dump file when complete.'
            ),
            make_option('-p', '--path',
                dest='path',
                default='local',
                help='Where to store the pg_dump file.'
            ),
            # Options passed on to Heroku
            make_option('--app',
                dest='app',
                default=None,
                help='Which Heroku app to use'
            )
        )

        def handle(self, backup_id='', **options):
            def _make_cmd(arg):
                cmd = ['heroku', 'pgbackups:%s' % arg]
                if options['app']:
                    cmd += ['--app', options['app']]
                return cmd

            # Capture a new backup if needed
            if options['capture']:
                subprocess.call(_make_cmd('capture'))

            # Expose and store a public backup url
            arg = 'url %s' % backup_id if backup_id else 'url'
            url = subprocess.check_output(_make_cmd(arg)).strip()

            # Download the dump to the location specified by -p
            filename = url.split('/')[-1].split('?')[0]
            filepath = os.path.join(
                settings.PROJECT_ROOT, options['path'], filename)
            dump = open(filepath, 'wb')
            dump.write(urllib2.urlopen(url).read())
            dump.close()

            # Load in the dump
            db = settings.DATABASES['default']
            subprocess.call(['pg_restore', '--verbose', '--clean', '--no-acl',
                '--no-owner', '-h', db['HOST'], '-U', db['USER'], '-d', db['NAME'],
                filepath])

            # Delete the dump, if required
            if options['delete']:
                subprocess.call('rm ' + filepath)
```

Note that `management` needs to be a submodule of an installed app and
`commands` needs to be a submodule of `management` (so don't forget the
`__init__.py`s).

Here's a quick rundown of what this is doing:

* By defining `Command` as a subclass of `BaseCommand`, this command gets
  registered as a `manage.py` command, with its name determined by the filename
  (in my case, `db:pull.py`).
* The `make_options` calls register several command line flags that can be
  passed in from the command line. Here's the documentation that it generates:
``` text
    $ ./manage.py help db:pull
    Usage: manage.py db:pull [options]

    Replaces `heroku db:pull` by using `pgbackups` to import data

    Options:
      -v VERBOSITY, --verbosity=VERBOSITY
                            Verbosity level; 0=minimal output, 1=normal output,
                            2=verbose output, 3=very verbose output
      --settings=SETTINGS   The Python path to a settings module, e.g.
                            "myproject.settings.main". If this isn't provided, the
                            DJANGO_SETTINGS_MODULE environment variable will be
                            used.
      --pythonpath=PYTHONPATH
                            A directory to add to the Python path, e.g.
                            "/home/djangoprojects/myproject".
      --traceback           Print traceback on exception
      -x                    Delete the pg_dump file when complete.
      -p PATH, --path=PATH  Where to store the pg_dump file.
      --app=APP             Which Heroku app to use
      --version             show program's version number and exit
      -h, --help            show this help message and exit
```
* The handle method actually executes the command, in this case by executing
  the sequence of command line arguments from above using `subprocess`.

### Paths and Settings

Since we're using a file dump, there are some path settings that may differ
from system to system. The snippet above assumes that you have `PROJECT_ROOT`
defined in your `settings.py` file; I always define `PROJECT_ROOT` like this:

``` python settings.py
    import os
    PROJECT_ROOT = os.path.dirname(os.path.realpath(__file__))
```

and then use
[relative paths](http://www.morethanseven.net/2009/02/11/django-settings-tip-setting-relative-paths.html)
for other path settings to keep things more portable:

``` python settings.py
    TEMPLATE_DIRS = (
        os.path.join(PROJECT_ROOT, 'templates'),
    )
```

It may seem suboptimal to have to deal with the filesystem and local paths, one
incidental benefit to saving the dump files locally is that you can always
restore from an old backup without using your live database (or internet
connection) at all.

### What's Next?

From here it should be clear how to implement a `db:push` command as well, but
I don't use them as often in my workflow, so I haven't written one yet. I'll
leave that as an exercise for the reader. If you have other ideas for making
Heroku deployment easier, I'd love to hear them.