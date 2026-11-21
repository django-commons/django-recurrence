.. _install:

Installation
============

.. contents::
   :local:


Download the library
--------------------

Firstly, you'll need to install ``django-recurrence`` from PyPI. The
easiest way to do this is with pip::

    pip install django-recurrence

Then, make sure ``recurrence`` is in your ``INSTALLED_APPS`` setting:

.. code-block:: python

    INSTALLED_APPS = (
      ...
      'recurrence',
    )

Supported Django and Python versions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Currently, django-recurrence supports Python 3.11, 3.12, 3.13, 3.14, and 3.15.

django-recurrence is currently tested with Django 5.2, 6.0, and 6.1.

Set up internationalization
---------------------------

The widget is shown in English by default. To show it in another
language, django-recurrence's JavaScript needs the translation
strings. They are served by Django's built in ``JavaScriptCatalog``
view, which you install by adding the following to your project
``urls.py`` file:

.. code-block:: python

    from django.urls import path
    from django.views.i18n import JavaScriptCatalog

    # Your normal URLs here...

    # If you already have a JavaScriptCatalog view that lists its
    # packages, just add 'recurrence' to them.
    urlpatterns += [
        # jsi18n/ can be anything you like here
        path(
            'jsi18n/',
            JavaScriptCatalog.as_view(packages=['recurrence']),
            name='javascript-catalog',
        ),
    ]

``{{ form.media }}`` finds this view and loads the catalog before the
widget scripts. If you include the scripts by hand, load the catalog
first:

.. code-block:: html+django

    <script src="{% url 'javascript-catalog' %}"></script>


Configure static files
----------------------

django-recurrence includes some static files (all to do with
rendering the JavaScript widget that makes handling recurring dates
easier). To ensure these are served correctly, you'll probably want
to ensure you also have ``django.contrib.staticfiles`` in your
``INSTALLED_APPS`` setting, and run::

    python manage.py collectstatic

.. note::
   After collecting static files, you can use ``{{ form.media }}`` to
   include recurrence's static files within your templates.
