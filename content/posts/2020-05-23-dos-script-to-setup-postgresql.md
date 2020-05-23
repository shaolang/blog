---
title: "MSDOS Script to Set Up PostgreSQL"
date: 2020-05-23T20:14:32+08:00
allowComments: true
---

Don't ask "Why Windows?"

Not being a database expert and needing to set up PostgreSQL database
in a Windows machine to do simple development with minimal fuss, the
following is a simplistic MSDOS script to automate that:

```batchfile {linenos=table, hl_lines=[16,17]}
@ECHO OFF
@BREAK ON
@SETLOCAL

SET PG_HOME=path_to_pg_home
SET superuser=postgres

IF NOT DEFINED PGDATA GOTO :MISSINGPGDATA
IF EXIST %PGDATA% GOTO :DBEXISTS

SET PGBIN=%PG_HOME%\bin
ECHO Initialzing database with superuser name: %superuser%

%PGBIN%\initdb -D %PGDATA% --pwprompt --username=%superuser% ^
  --no-locale --encoding=UTF8 ^
  --auth-local=trust
  --auth-host=trust

GOTO :EOF

:MISSINGPGDATA
ECHO Please set the environment variable PGDATA before running again.
GOTO :EOF

:DBEXISTS
ECHO Database already initialized at %PGDATA%. Please specify a different %%PGDATA%%.
```

A few things to note:

* Set the PostgreSQL home directory in line 5
* Change the default superuser name in line 6 if necessary
* Change [authentication methods][auth] to whatever that's appropriate:
  in this case, it just trusts all connections, so don't use `trust`
  in production!

Once the initializing is completed, run `pg_ctl start -D %PGDATA% -l logfile`
to start the database daemon. To stop it, run `pg_ctl stop`.

[auth]: https://www.postgresql.org/docs/12/auth-methods.html
