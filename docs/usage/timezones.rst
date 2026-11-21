.. _timezones:

Time zones and daylight saving time
-----------------------------------

.. contents::
   :local:

django-recurrence is at its simplest when a recurrence is a set of
*dates*. As soon as the occurrences have a time of day, time zones
and daylight saving time come into play. This page explains what is
stored, why a time can move by an hour, and how to keep an event at
the same local time all year.

What is stored
^^^^^^^^^^^^^^

A ``RecurrenceField`` stores text. Every date and time in it is
converted to UTC first:

.. code-block:: python

    from datetime import datetime
    from zoneinfo import ZoneInfo
    import recurrence

    berlin = ZoneInfo('Europe/Berlin')
    pattern = recurrence.Recurrence(
        dtstart=datetime(2022, 10, 28, 16, 0, tzinfo=berlin),
        rrules=[recurrence.Rule(recurrence.DAILY, count=4)],
    )

    str(pattern)
    # 'DTSTART:20221028T140000Z\nRRULE:FREQ=DAILY;COUNT=4'

The time zone itself is not stored. 16:00 in Berlin has become 14:00
UTC, and nothing in the stored text says that Berlin was meant. A
datetime without a time zone is taken to be in Django's current time
zone before it is converted.

What you get back
^^^^^^^^^^^^^^^^^

How the stored values are returned depends on your settings:

- With ``USE_TZ = True`` you get aware datetimes in UTC.
- With ``USE_TZ = False`` you get naive datetimes in Django's current
  time zone.

To choose for django-recurrence alone, set ``RECURRENCE_USE_TZ`` to
``True`` or ``False``. It takes the place of ``USE_TZ`` for the
values this package returns.

Why a time moves by an hour
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Occurrences are computed in the time zone of ``dtstart``. A
``dtstart`` that has been saved and loaded with ``USE_TZ = True`` is
in UTC, so the occurrences keep their UTC time. The local time then
changes when daylight saving time starts or ends. Clocks in Berlin
went back on 30 October 2022:

.. code-block:: python

    course.recurrences = pattern
    course.save()
    course.refresh_from_db()

    for occurrence in course.recurrences.occurrences():
        print(occurrence.astimezone(berlin))

    # 2022-10-28 16:00:00+02:00
    # 2022-10-29 16:00:00+02:00
    # 2022-10-30 15:00:00+01:00
    # 2022-10-31 15:00:00+01:00

The ``pattern`` object from before it was saved still has its Berlin
time zone and gives 16:00 on all four days. The difference only
appears after the round trip through the database.

.. _local-time-events:

Events at a fixed local time
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To keep an event at the same local time, let the recurrence decide
the *days* only, and keep the start date, the time of day and the
time zone in fields of their own:

.. code-block:: python

    from datetime import datetime, time
    from zoneinfo import ZoneInfo

    from django.db import models
    from django.utils import timezone
    from recurrence.fields import RecurrenceField


    class Course(models.Model):
        title = models.CharField(max_length=200)
        start_date = models.DateField()
        start_time = models.TimeField()
        time_zone = models.CharField(max_length=64, default='Europe/Berlin')
        recurrences = RecurrenceField(include_dtstart=False)

        def sessions_between(self, first, last):
            """Sessions from the date `first` to the date `last`."""
            zone = ZoneInfo(self.time_zone)
            with timezone.override(zone):
                days = self.recurrences.between(
                    datetime.combine(first, time.min),
                    datetime.combine(last, time.min),
                    dtstart=datetime.combine(self.start_date, time.min),
                    inc=True,
                )
            return [
                datetime.combine(day.date(), self.start_time, tzinfo=zone)
                for day in days
            ]

A course that starts on Monday 24 October 2022 at 16:00 and runs
every Monday, Wednesday and Friday, except on 31 October, then keeps
its time across the clock change:

.. code-block:: python

    from datetime import date

    sessions = course.sessions_between(date(2022, 10, 26), date(2022, 11, 4))
    for session in sessions:
        print(session)

    # 2022-10-26 16:00:00+02:00
    # 2022-10-28 16:00:00+02:00
    # 2022-11-02 16:00:00+01:00
    # 2022-11-04 16:00:00+01:00

Each part of ``sessions_between`` is there for a reason:

- ``dtstart`` is the course's own start date. The form widget has no
  input for a start, and without ``dtstart`` the pattern starts at
  the moment the code runs. Rules such as "every two weeks" or "ten
  times" count from ``dtstart``, so it should not change between
  calls.
- ``dtstart`` is at midnight and has no time zone. The widget stores
  the dates it adds, excludes or uses as "until" as midnight, and a
  date is only excluded when its time is the same as the time of the
  occurrence.
- ``timezone.override`` sets the time zone in which the stored
  midnights are read back.
- ``include_dtstart=False`` keeps the start date out of the result
  when no rule matches it.
- The time of day and the time zone are added last, to dates that are
  already final.

.. note::

    The widget stores a date as midnight in the time zone of the
    browser. If someone picks dates in a browser set to another time
    zone than the one of the event, those dates can end up a day off.

Everything in one time zone
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If every event is in the time zone of your ``TIME_ZONE`` setting,
and you do not activate other time zones, there is a shorter way.
Keep ``dtstart`` in the recurrence and set:

.. code-block:: python

    RECURRENCE_USE_TZ = False

The stored values then come back as naive datetimes in local time,
and naive datetimes keep the time of day across a clock change. Make
them aware when you need to:

.. code-block:: python

    from django.utils import timezone

    for occurrence in course.recurrences.occurrences():
        print(timezone.make_aware(occurrence))

    # 2022-10-28 16:00:00+02:00
    # 2022-10-29 16:00:00+02:00
    # 2022-10-30 16:00:00+01:00
    # 2022-10-31 16:00:00+01:00

The values are converted to Django's current time zone when they are
loaded. If a middleware activates the time zone of each user, the
same stored recurrence gives different local times to different
users, so use the approach in :ref:`local-time-events` instead.

Do not mix naive and aware datetimes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``dtstart`` and the datetimes you pass to ``between``, ``after`` and
``before`` have to be of the same kind: all with a time zone, or all
without. Otherwise the call fails:

.. code-block:: text

    TypeError: can't compare offset-naive and offset-aware datetimes

Two cases cause most of these errors:

- No ``dtstart`` is given, neither in the recurrence nor in the
  call. The pattern then starts at the current time, as a naive
  datetime, so the other arguments have to be naive too.
- A ``dtstart`` loaded with ``USE_TZ = True`` is aware, so the other
  arguments have to be aware too.
