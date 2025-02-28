# sqlalchemy-celery-beat

A Scheduler Based Sqlalchemy for Celery.

> NOTE: This project was originally developed by [AngelLiang](https://github.com/AngelLiang/celery-sqlalchemy-scheduler) to use sqlalchemy as the database scheduler for Flask or FastAPI, like [django-celery-beat](https://github.com/celery/django-celery-beat) for django. I am trying to continue on his work and maintain a working solution.


### Prerequisites

- Python 3
- celery >= 5.0
- sqlalchemy >= 1.4

First you must install `celery` and `sqlalchemy`, and `celery` should be >=5.0

```
$ pip install sqlalchemy celery
```

### Installing

Install from PyPi:

```
$ pip install sqlalchemy-celery-beat
```

Install from source by cloning this repository:

```
$ git clone git@github.com:farahats9/sqlalchemy-celery-beat.git
$ cd sqlalchemy-celery-beat
$ python setup.py install
```

## Usage

After you have installed `sqlalchemy_celery_beat`, you can easily start with following steps:

This is a demo for exmaple, you can check the code in `examples` directory

1. start celery worker

   ```
   $ celery worker -A tasks -l info
   ```

2. start the celery beat with `DatabaseScheduler` as scheduler:

   ```
   $ celery beat -A tasks -S sqlalchemy_celery_beat.schedulers:DatabaseScheduler -l info
   ```
    you can also use the shorthand argument `-S sqlalchemy`
## Description

After the celery beat is started, by default it create a sqlite database(`schedule.db`) in current folder. You can use `SQLiteStudio.exe` to inspect it.

Sample from the `PeriodicTask` model's table

![sqlite](screenshot/sqlite.png)

When you want to update scheduler, you can update the data in `schedule.db`. But `sqlalchemy_celery_beat` don't update the scheduler immediately. Then you shoule be change the first column's `last_update` field in the `celery_periodic_task_changed` to now datetime. Finally the celery beat will update scheduler at next wake-up time.

### Database Configuration

You can configure sqlalchemy db uri when you configure the celery, example as:

```Python
from celery import Celery

celery = Celery('tasks')

beat_dburi = 'sqlite:///schedule.db'

celery.conf.update(
    {
        'beat_dburi': beat_dburi,
        'beat_schema': None  # you can make the scheduler tables under different schema (tested for postgresql, not available in sqlite)
    }
)
```

Also, you can use MySQL or PostgreSQL.

```Python
# MySQL: `pip install mysql-connector`
beat_dburi = 'mysql+mysqlconnector://root:root@127.0.0.1:3306/celery-schedule'

# PostgreSQL: `pip install psycopg2`
beat_dburi = 'postgresql+psycopg2://postgres:postgres@127.0.0.1:5432/celery-schedule'
```

## Example Code 1

View `examples/base/tasks.py` for details.


Run Worker in console 1

    $ cd examples/base

    # Celery < 5.0
    $ celery worker -A tasks:celery -l info

    # Celery >= 5.0
    $ celery -A tasks:celery worker -l info

Run Beat in console 2

    $ cd examples/base

    # Celery < 5.0
    $ celery beat -A tasks:celery -S tasks:DatabaseScheduler -l info

    # Celery >= 5.0
    $ celery -A tasks:celery beat -S tasks:DatabaseScheduler -l info

## Example Code 2

### Example creating interval-based periodic task

To create a periodic task executing at an interval you must first
create the interval object:

```python
>>> from sqlalchemy_celery_beat.models import PeriodicTask, IntervalSchedule
>>> from sqlalchemy_celery_beat.session import SessionManager
>>> from celeryconfig import beat_dburi
>>> session_manager = SessionManager()
>>> engine, Session = session_manager.create_session(beat_dburi)
>>> session = Session()

# executes every 10 seconds.
>>> schedule = session.query(IntervalSchedule).filter_by(every=10, period=IntervalSchedule.SECONDS).first()
>>> if not schedule:
...     schedule = IntervalSchedule(every=10, period=IntervalSchedule.SECONDS)
...     session.add(schedule)
...     session.commit()
```

That's all the fields you need: a period type and the frequency.

You can choose between a specific set of periods:

- `IntervalSchedule.DAYS`
- `IntervalSchedule.HOURS`
- `IntervalSchedule.MINUTES`
- `IntervalSchedule.SECONDS`
- `IntervalSchedule.MICROSECONDS`

_note_:

    If you have multiple periodic tasks executing every 10 seconds,
    then they should all point to the same schedule object.

Now that we have defined the schedule object, we can create the periodic task
entry:

```python
    >>> task = PeriodicTask(
    ...     schedule_model=schedule,            # we created this above.
    ...     name='Importing contacts',          # simply describes this periodic task.
    ...     task='proj.tasks.import_contacts',  # name of task.
    ... )
    >>> session.add(task)
    >>> session.commit()
```

Note that this is a very basic example, you can also specify the
arguments and keyword arguments used to execute the task, the `queue` to
send it to[\*], and set an expiry time.

Here\'s an example specifying the arguments, note how JSON serialization
is required:

    >>> import json
    >>> from datetime import datetime, timedelta

    >>> periodic_task = PeriodicTask(
    ...     schedule_model=schedule,                  # we created this above.
    ...     name='Importing contacts',          # simply describes this periodic task.
    ...     task='proj.tasks.import_contacts',  # name of task.
    ...     args=json.dumps(['arg1', 'arg2']),
    ...     kwargs=json.dumps({
    ...        'be_careful': True,
    ...     }),
    ...     expires=datetime.utcnow() + timedelta(seconds=30)
    ... )
    ... session.add(periodic_task)
    ... session.commit()

### Example creating crontab-based periodic task

A crontab schedule has the fields: `minute`, `hour`, `day_of_week`,
`day_of_month` and `month_of_year`, so if you want the equivalent of a
`30 * * * *` (execute every 30 minutes) crontab entry you specify:

    >>> from sqlalchemy_celery_beat.models import PeriodicTask, CrontabSchedule
    >>> schedule = CrontabSchedule(
    ...     minute='30',
    ...     hour='*',
    ...     day_of_week='*',
    ...     day_of_month='*',
    ...     month_of_year='*',
    ...     timezone='UTC',
    ... )

The crontab schedule is linked to a specific timezone using the
'timezone' input parameter.

Then to create a periodic task using this schedule, use the same
approach as the interval-based periodic task earlier in this document,
the `schedule_model` is a generic foreign-key implementation which makes things very easy and efficient:

    >>> periodic_task = PeriodicTask(
    ...     schedule_model=schedule,
    ...     name='Importing contacts',
    ...     task='proj.tasks.import_contacts',
    ... )

What the previous code actually do is this:

    >>> periodic_task = PeriodicTask(
    ...     schedule_id=schedule.id,
    ...     discriminator=schedule.discriminator,
    ...     name='Importing contacts',
    ...     task='proj.tasks.import_contacts',
    ... )
So when you can use `discriminator` + `schedule_id` or use the convenient property `schedule_model` and it will populate them for you behind the scenes.

### Temporarily disable a periodic task

You can use the `enabled` flag to temporarily disable a periodic task:

    >>> periodic_task.enabled = False
    >>> session.add(periodic_task)
    >>> session.commit()

If you are using a bulk operation to update or delete multiple tasks at the same time, the changes won't be noticed by the scheduler until you do `PeriodicTaskChanged.update_changed()` or `.update_from_session()`

example:
``` python
from sqlalchemy_celery_beat.models import PeriodicTaskChanged

session = get_beat_session()

stmt = update(PeriodicTask).where(PeriodicTask.name == 'task-123').values(enabled=False)

session.execute(stmt)  # using execute causes no orm event to fire, changes are in the database but the schduler has no idea
session.commit()

PeriodicTaskChanged.update_from_session(session)
# now scheduler reloads the tasks and all is good
```
This is not needed when you are updating a specific object using `session.add(task)` because it will trigger the `after_update`, `after_delete` or `after_insert` events.

### Example running periodic tasks

The periodic tasks still need 'workers' to execute them. So make sure
the default **Celery** package is installed. (If not installed, please
follow the installation instructions here:
<https://github.com/celery/celery>)

Both the worker and beat services need to be running at the same time.

1.  Start a Celery worker service (specify your project name):

        $ celery -A [project-name] worker --loglevel=info

2.  As a separate process, start the beat service (specify the
    scheduler):

        $ celery -A [project-name] beat -l info --scheduler sqlalchemy_celery_beat.schedulers:DatabaseScheduler

## Working on adding the following features

- ✅ Add `ClockedSchedule` model
- ✅ Implement a generic foreign key
- ✅ More robust attribute validation on models
- ✅ Add Tests
- Add more examples
- Support for Async drivers like asyncpg and psycopg3 async mode
- Use Alembic migrations

Any help with the tasks above or feedback is appreciated 🙂

## Acknowledgments

- [django-celery-beat](https://github.com/celery/django-celery-beat)
- [celerybeatredis](https://github.com/liuliqiang/celerybeatredis)
- [celery](https://github.com/celery/celery)
- [SQLAlchemy](https://www.sqlalchemy.org)
