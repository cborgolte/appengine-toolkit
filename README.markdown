gaetk - Google Appengine Toolkit
================================

gaetk is a small collection of tools that tries to make programming Google AppEngine faster, more comfortable and less error prone. It is pre-alpha quality software.

It comes bundled with [gaesession][1] and [webapp2][2]. Check the documentation of these projects for further information.

Creating a Project
------------------

Create a appengine Project, then:

    mkdir lib
    git submodule add git@github.com:mdornseif/appengine-toolkit.git lib/gaetk
    git submodule update --init lib/gaetk
    cp lib/gaetk/examples/Makefile .
    cp lib/gaetk/examples/config.py .
    cp lib/gaetk/examples/appengine_config.py .
    sed -i -e "s/%%PUT_RANDOM_VALUE_HERE%%/`(date;md5 /etc/* 2&>/dev/null)|md5`-5a17/" appengine_config.py
    # If you want to use jinja2
    git submodule add https://github.com/mitsuhiko/jinja2.git lib/jinja2
    git submodule update --init lib/jinja2


You might also want to install https://github.com/hudora/huTools for some additional functionality.


Functionality
=============

Sequence generation
-------------------

Generation of sequential numbers ('autoincrement') on Google appengine is hard. See [Stackoverflow](http://stackoverflow.com/questions/3985812) for some discussion of the issues. `gaetk` implements a sequence number generation based on transactions. This will yield only a preformance of half a dozen or so requests per second but at least allows to alocate more than one number in a single request.

    >>> from gaeth.sequences import * 
    >>> init_sequence('invoce_number', start=1, end=0xffffffff)
    >>> get_numbers('invoce_number', 2)
    [1, 2]
    

Login / Authentication
----------------------

gaetk offers a hybrid Google Apps / Credential based authentication via session Cookies and HTTP Auth. This approach tries to accomodate command-line/API clients like `curl` and browser based clients. If a client does provide an `Accept`-Header containing `text/html` (all Browsers do) the client is redirected to `/_ah/login_required` where the user can Enter `uid` and `secret` (Session-Based Authentication) or login via OpenID/Google Apps. If there is no `Accept: text/html` the client is presented with a 401 status code, inviting the client to offer HTTP-Basic-Auth.

To use OpenID with Google Apps configure your Application to use "Federated Login":

![Federated Login](http://static.23.nu/md/Pictures/ZZ77D022D0.png)

Users are represented by `gaetk.handler.Credential` objects. For OpenID users a uid ("Username") and secret ("Password") are autogenerated. For Session or HTTP-Auth based ssers uid and secret should also be autogenerated. I *strongly* avaise against user settable passwords and suggest you might also consider avoiding user settable usernames. Use `Credential.create()` to create new Crdentials including auto-generated uid and secret. If you want to set the `uid` mannually , give the `uid` parameter.

    gaetk.handler.Credential.create(text='API Auth for Schulze Server', email='ops@example.com')
    gaetk.handler.Credential.create(uid='user5711', text='...', email='ops@example.com')


To use it add the following to `app.yaml`:

    handlers:
    - url: /_ah/login_required
      script: lib/gaetk/gaetk/login.py
    
    - url: /logout
      script: lib/gaetk/gaetk/login.py


Now in your views/handlers you can easyly force authentication like this:

    from gaetk.handler import BasicHandler
    
    class HomepageHandler(BasicHandler):
        def get(self):
            user = self.login_required() # results in 401/403 if can't login
            ...


Unless you call `login_required(deny_localhost=False)` access from localhost is always considered authenticated.

Statistics
----------

Add the following to app.yaml:

    - url: /gaetk/.*
      script: lib/gaetk/gaetk/defaulthandlers.py

Now you can get some Application statistics at http://EXAMPLE.appspot.com/gaetk/stats.json:

    {"datastore": {"count": 174789,
                   "kinds": 16,
                   "bytes": 102391232},
     "memcache": {"hits": 1665726,
                  "items": 1171,
                  "bytes": 4588130,
                  "oldest_item_age": 2916,
                  "misses": 50674,
                  "byte_hits": 833839440}}


JSONviews
---------

`gaetk.handler.JsonResponseHandler` helps to generate nice JSON and JSONP responses. A valid view will look like this:

    class VersandauslastungHandler(gaetk.handler.JsonResponseHandler):
        def get(self):
            entity = BiStatistikZeile.get_by_key_name('versandauslastung_aktuell')
            ret = dict(werte=hujson.loads(entity.jsonValue),
                       ...)
            return (ret, 200, 60)

This will generate a JSON reply with 60 Second caching and a 200 status code. The reply will support [JSONP](http://en.wikipedia.org/wiki/JSONP#JSONP) via an optional `callback` parameter in the URL.


Pre-Made Views
--------------

Add the following lines to your `app.yaml`:

    - url: /gaetk/.*
      script: lib/gaetk/gaetk/defaulthandlers.py

This will allow you to get JSON encoded statistics at `/gaetk/stats.json`:

    curl http://localhost:8080/gaetk/stats.json
    {"datastore": {"count": 149608,
                   "kinds": 16,
                   "bytes": 95853319},
     "memcache": {"hits": 0,
                  "items": 0,
                  "bytes": 0,
                  "oldest_item_age": 0,
                  "misses": 0,
                  "byte_hits": 0}}

You might want eo use Munin to graph these values.


LoggedModel
------------

In models.py you find a superclass `LoggedModel` for implementing audit logs for a model:
All create, update and delete operations are logged via the `AuditLog` model.

Example:

    import gaetk.models
    from google.appengine.ext import db

    class MyModel(gaetk.models.LoggedModel):
        name = db.StringProperty()

    obj = MyModel(name=u'Alex')
    print obj.logentries()


Please note: Changes in (subclasses of) UnindexedProperty (hence TextProperty and BlobProperty) are not logged.


Tools
-----

Tools contians general helpers. It is independent of the rest of gaetk.

`tools.split(s)` "Splits a string at space characters while respecting quoting.

    >>> split('''A "B and C" D 'E or F' G " "''')
    ['A',
     'B and C',
     'D',
     'E or F',
     'G',
     '']

Infrastructure
--------------

Infrastructure contians helpers for accessing the GAE infrastructure. It is independent of the rest of gaetk.


`taskqueue_add_multi` batch adds jobs to a Taskqueue:

    tasks = []
    for kdnnr in kunden.get_changed():
        tasks.append(dict(kundennr=kdnnr))
    taskqueue_add_multi('softmq', '/some/path', tasks)


client side functionality
=========================

In addition to the server side functionality gaetk also includes various javascript helper methods. They must be used in combination with ExtJS and provide methods for easy handling of often-used tasks. To use the javascript helpers you will have to link the `web` directory into the directory declared the `static_dir` directory in your `app.yaml`. Then include the `gaetk-extjs-helpers.js` and `gaetk-extjs-helpers.css` files into your HTML page template.

Currently the helpers include two methods:

 * `Hudora.Helpers.spinnerMessageBox(message)`: display a non-closable messagebox with a spinner indicating progress
 * `Hudora.Helpers.errorMessageBox(title, message)`: display a error message box without having to write five lines of code every time you need an error messagebox.


Thanks
======

Axel Schlüter for suggestions on abstracting login and JsonResponseHandler.

Contains [gaesession.py][1] by David Underhill - http://github.com/dound/gae-sessions
Updated 2010-10-02 (v1.05), Licensed under the Apache License Version 2.0.

Contains code from [webapp2][2], Copyright 2010 Rodrigo Moraes.
Licensed under the Apache License, Version 2.0

gaetk code is Copyright 2010 Hudora GmbH and dual licensed under GPLv3 and the
Apache License Version 2.0.


[1]: https://github.com/dound/gae-sessions
[2]: http://code.google.com/p/webapp-improved/

