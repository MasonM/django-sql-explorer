.. image:: https://travis-ci.org/epantry/django-sql-explorer.png?branch=master
   :target: https://travis-ci.org/epantry/django-sql-explorer

Django SQL Explorer
===================

Django SQL Explorer is inspired by any number of great query and reporting tools out there.

The original idea came from Stack Exchange's `Data Explorer <http://data.stackexchange.com/stackoverflow/queries>`_, but also owes credit to similar projects like `Redash <http://redash.io/>`_ and `Blazer <https://github.com/ankane/blazer>`_.

SQL Explorer wants to make the flow of data between people fast, simple, and confusion-free.

Quickly write and share SQL queries in a clean, usable query builder, preview the results in the browser, share links to download CSV files, and keep the information flowing!

The product & design principles are: simplicity, intuitive use, unobtrusiveness, stability, and being unsurprising.

django-sql-explorer is MIT licensed, and pull requests are welcome.

**A view of a query**

.. image:: https://s3-us-west-1.amazonaws.com/django-sql-explorer/2.png

**Viewing all queries**

.. image:: https://s3-us-west-1.amazonaws.com/django-sql-explorer/query-list.png

**Quick access to DB schema info**

.. image:: https://s3-us-west-1.amazonaws.com/django-sql-explorer/3.png

**Snapshot query results to S3 & download as csv**

.. image:: https://s3-us-west-1.amazonaws.com/django-sql-explorer/snapshots.png


Features
========

- **Security**
    - Let's not kid ourselves - this tool is all about giving people access to running SQL in production. So if that makes you nervous (and it should) - you've been warned. Explorer makes an effort to not allow terrible things to happen, but be careful! It's recommended you use the EXPLORER_CONNECTION_NAME setting to connect SQL Explorer to a read-only database role.
    - Explorer supports two different permission checks for users of the tool. Users passing the EXPLORER_PERMISSION_CHANGE test can create, edit, delete, and execute queries. Users who do not pass this test but pass the EXPLORER_PERMISSION_VIEW test can only execute queries. Other users cannot access any part of Explorer. Both permission groups are set to is_staff by default and can be overridden in your settings file.
    - Enforces a SQL blacklist so destructive queries don't get executed (delete, drop, alter, update etc). This is not bulletproof and it's recommended that you instead configure a read-only database role, but when not possible the blacklist provides reasonable protection.
- **Easy to get started**
    - Built on Django's ORM, so works with Postgresql, Mysql, and Sqlite.
    - Small number of dependencies.
    - Just want to get in and write some ad-hoc queries? Go nuts with the Playground area.
- *new* **Snapshots**
    - Tick the 'snapshot' box on a query, and Explorer will upload a .csv snapshot of the query results to S3. Configure the snapshot frequency via a celery cron task, e.g. for daily at 1am:

    .. code-block:: python

       'explorer.tasks.snapshot_queries': {
           'task': 'explorer.tasks.snapshot_queries',
           'schedule': crontab(hour=1, minute=0)
       }

    - Requires celery, obviously. Also uses djcelery and tinys3. All of these deps are optional and can be installed with `pip install -r optional-requirements.txt`
    - The checkbox for opting a query into a snapshot is ALL THE WAY on the bottom of the query view (underneath the restults table).
- **Email query results**
    - Click the email icon in the query listing view, enter an email address, and the query results (zipped .csv) will be sent to you.
- **Parameterized Queries**
    - Use $$foo$$ in your queries and Explorer will build a UI to fill out parameters. When viewing a query like 'SELECT * FROM table WHERE id=$$id$$', Explorer will generate UI for the 'id' parameter.
    - Parameters are stashed in the URL, so you can share links to parameterized queries with colleagues
    - Use $$paramName:defaultValue$$ to provide default values for the parameters.
- **Schema Helper**
    - /explorer/schema/ renders a list of your Django apps' table and column names + types that you can refer to while writing queries. Apps can be excluded from this list so users aren't bogged down with tons of irrelevant tables. See settings documentation below for details.
    - This is available quickly as a sidebar helper while composing queries (see screenshot)
    - Supports many_to_many relations as well.
    - Quick search for the tables/django models you are looking for. Just start typing!
- **Template Columns**
    - Let's say you have a query like 'select id, email from user' and you'd like to quickly drill through to the profile page for each user in the result. You can create a "template" column to do just that.
    - Just set up a template column in your settings file:

    ``EXPLORER_TRANSFORMS = [('user', '<a href="https://yoursite.com/profile/{0}/">{0}</a>')]``

    - And change your query to 'SELECT id AS "user", email FROM user'. Explorer will match the "user" column alias to the transform and merge each cell in that column into the template string. Cool!
- **Pivot Table**
    - Go to the Pivot tab on query results to use the in-browser pivot functionality (provided by Pivottable JS).
    - Hit the link icon on the top right to get a URL to recreate the exact pivot setup to share with colleagues.
- **Query Logs**
    - Explorer will save a snapshot of every query you execute so you can recover lost ad-hoc queries, and see what you've been querying.
    - This also serves as cheap-and-dirty versioning of Queries, and provides the 'run count' property and average duration in milliseconds, by aggregating the logs.
    - You can also quickly share playground queries by copying the link to the playground's query log record -- look on the top right of the sql editor for the link icon.
    - If Explorer gets a lot of use, the logs can get beefy. explorer.tasks contains the 'truncate_querylogs' task that will remove log entries older than <days> (30 days and older in the example below).

    .. code-block:: python

       'explorer.tasks.truncate_querylogs': {
           'task': 'explorer.tasks.truncate_querylogs',
           'schedule': crontab(hour=1, minute=0),
           'kwargs': {'days': 30}
       }
- **Stable**
    - 95% according to coverage...for what that's worth. Just install factory_boy and run `python manage.py test explorer.tests --settings=explorer.tests.settings`
    - Battle-tested in production every day by the ePantry team.
- **Power tips**
    - On the query listing page, focus gets set to a search box so you can just navigate to /explorer and start typing the name of your query to find it.
    - Quick search also works after hitting "Show Schema" on a query view.
    - Command+Enter and Ctrl+Enter will execute a query when typing in the SQL editor area.
    - Hit the "Format" button to format and clean up your SQL (this is non-validating -- just formatting).
    - Use the Query Logs feature to share one-time queries that aren't worth creating a persistent query for. Just run your SQL in the playground, then navigate to /logs and share the link (e.g. /explorer/play/?querylog_id=2428)
    - If you need to download a query as something other than csv but don't want to globally change delimiters via settings.EXPLORER_CSV_DELIMETER, you can use /query/download?delim=| to get a pipe (or whatever) delimited file. For a tab-delimited file, use delim=tab. Note that the file extension will remain .csv
    - If a query is taking a long time to run (perhaps timing out) and you want to get in there to optimize it, go to /query/123/?show=0. You'll see the normal query detail page, but the query won't execute.
    - Set env vars for EXPLORER_TOKEN_AUTH_ENABLED=TRUE and EXPLORER_TOKEN=<SOME TOKEN> and you have an instant data API. Just:

    ``curl --header "X-API-TOKEN: <TOKEN>" https://www.your-site.com/explorer/<QUERY_ID>/csv``

Install
=======

Requires Python 2.7, 3.4, or 3.5. Requires Django 1.7.1 or higher.

Install with pip from github:

``pip install django-sql-explorer``

Add to your installed_apps:

``INSTALLED_APPS = (
...,
'explorer',
...
)``

Add the following to your urls.py (all Explorer URLs are restricted to staff only per default):

``url(r'^explorer/', include('explorer.urls')),``

Run syncdb to create the tables:

``python manage.py syncdb``

You can now browse to https://yoursite/explorer/ and get exploring! However note it is highly recommended that you also configure Explorer to use a read-only database connection via the EXPLORER_CONNECTION_NAME setting.

Dependencies
============

An effort has been made to keep the number of dependencies to a minimum.

*Back End*

=========================================================== ======= ================
Name                                                        Version License
=========================================================== ======= ================
`sqlparse  <https://github.com/andialbrecht/sqlparse/>`_    0.1.18  BSD
`Factory Boy <https://github.com/rbarrois/factory_boy>`_    2.6.0   MIT
`unicodecsv <https://github.com/jdunck/python-unicodecsv>`_ 0.14.1  BSD
=========================================================== ======= ================

- sqlparse is Used for SQL formatting only
- Factory Boy is only required for tests
- unicodecsv is used for CSV generation

*Front End*

============================================================ ======== ================
Name                                                         Version  License
============================================================ ======== ================
`Twitter Boostrap <http://getbootstrap.com/>`_               3.3.6    MIT
`jQuery <http://jquery.com/>`_                               2.1.4    MIT
`jQuery Cookie <https://github.com/carhartl/jquery-cookie>`_ 1.4.1    MIT
`Underscore <http://underscorejs.org/>`_                     1.7.0    MIT
`Codemirror <http://codemirror.net/>`_                       5.11.0   MIT
`floatThead <http://mkoryak.github.io/floatThead/>`_         1.2.8    MIT
`list.js <http://listjs.com>`_                               1.1.1    MIT
`pivottable.js <http://nicolas.kruchten.com/pivottable/>`_   2.0.0    MIT
============================================================ ======== ================

Tests
=====

Factory Boy is needed if you'd like to run the tests, which can you do easily:

``python manage.py test --settings=explorer.tests.settings``

and with coverage:

``coverage run --source='.' manage.py test --settings=explorer.tests.settings``

then:

``coverage report``

...95%! Huzzah!

Settings
========

============================= =============================================================================================================== ================================================================================================================================================
Setting                       Description                                                                                                                                                  Default
============================= =============================================================================================================== ================================================================================================================================================
EXPLORER_SQL_BLACKLIST        Disallowed words in SQL queries to prevent destructive actions.                                                 ('ALTER', 'RENAME ', 'DROP', 'TRUNCATE', 'INSERT INTO', 'UPDATE', 'REPLACE', 'DELETE', 'ALTER', 'CREATE TABLE', 'SCHEMA', 'GRANT', 'OWNER TO')
EXPLORER_SQL_WHITELIST        These phrases are allowed, even though part of the phrase appears in the blacklist.                             ('CREATED', 'DELETED','REGEXP_REPLACE')
EXPLORER_DEFAULT_ROWS         The number of rows to show by default in the preview pane.                                                      1000
EXPLORER_SCHEMA_EXCLUDE_APPS  Don't show schema for these packages in the schema helper.                                                      ('django.contrib.auth', 'django.contrib.contenttypes', 'django.contrib.sessions', 'django.contrib.admin')
EXPLORER_CONNECTION_NAME      The name of the Django database connection to use. Ideally set this to a connection with read only permissions  None  # Which means use the 'default' connection
EXPLORER_PERMISSION_VIEW      Callback to check if the user is allowed to view and execute stored queries                                     lambda u: u.is_staff
EXPLORER_PERMISSION_CHANGE    Callback to check if the user is allowed to add/change/delete queries                                           lambda u: u.is_staff
EXPLORER_TRANSFORMS           List of tuples like [('alias', 'Template for {0}')]. See features section of this doc for more info.            []
EXPLORER_RECENT_QUERY_COUNT   The number of recent queries to show at the top of the query listing.                                           10
EXPLORER_GET_USER_QUERY_VIEWS A dict granting view permissions on specific queries of the form {userId:[queryId, ...], ...}                   {}
EXPLORER_TOKEN_AUTH_ENABLED   Bool indicating whether token-authenticated requests should be enabled. See "Power Tips", above.                False
EXPLORER_TOKEN                Access token for query results.                                                                                 "CHANGEME"
EXPLORER_TASKS_ENABLED        Turn on if you want to use the snapshot_queries celery task, or email report functionality in tasks.py          False
EXPLORER_S3_ACCESS_KEY        S3 Access Key for snapshot upload                                                                               None
EXPLORER_S3_SECRET_KEY        S3 Secret Key for snapshot upload                                                                               None
EXPLORER_S3_BUCKET            S3 Bucket for snapshot upload                                                                                   None
EXPLORER_FROM_EMAIL           The default 'from' address when using async report email functionality                                          "django-sql-explorer@example.com"
============================= =============================================================================================================== ================================================================================================================================================

Release Process
===============

Release process is documented `here <https://gist.github.com/chrisclark/07a6b4ef0114fdfa2ee0>`_. If there are problems with the release, please help me improve the process so it doesn't happen again!