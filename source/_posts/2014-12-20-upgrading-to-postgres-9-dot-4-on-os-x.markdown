---
layout: post
title: "Upgrading to Postgres 9.4.0 on OS X"
date: 2014-12-20 10:58:28 -0700
comments: true
categories: 
---
Postgres 9.4.0 has finally shipped. This long awaited release includes numerous upgrades, 
including native support for JSON, and improved scalability and index performance.

Upgrading to Postgres 9.4.0 is not as simple as merely replacing your current 9.3.x version with the newer
version. The release notes indicate that users who wish to upgrade must first migrate their existing data
from prior versions using one of two strategies: `pg_dumpall` or `pg_upgrade`.

In this post I'll provide the instructions to use `pg_dumpall` to upgrade on OS X.

There are typically two ways that OS X users install Postgres on their machine: via
the OS X native `postgres.app` (the one with the blue elephant icon) or using the package manager `homebrew`.
I'll provide instructions for both.

## 1: Export your existing databases to a SQL script file

_DO THIS FIRST FOR EITHER INSTALLATION METHOD_

Postgres stores databases in version specific directories.  So before we get started we need to know what 
the current Postgres data directory is for the installed version.  Fire up your postgres console
and find out with `SHOW data_directory;`.  Your output may look different, but what we want is the path that
is returned.  On my machine it looked like this:

``` sql Using psql to locate the data directory
$ psql
Null display is "[NULL]".
Expanded display is used automatically.
psql (9.3.5.2)
Type "help" for help.

localhost @David=# SHOW data_directory;

                      data_directory
-----------------------------------------------------------
 /Users/David/Library/Application Support/Postgres/var-9.3
(1 row)

localhost @David=# \q

$
```
Make a note of this directory.  Once the data has been migrated and you've confirmed everything is
working you'll probably want to delete the old copy to free up space.

Next export the existing 9.3 databases out of that directory prior to upgrading to 9.4:

``` bash Using pg_dumpall to create a single SQL script file for migration to 9.4
mkdir pg_migrate
cd pg_migrate
pg_dumpall > db.out
``` 
It's worth noting that like many unix commands you will get no feedback from this command while it is running.
Depending on how many and/or how large your pg databases are this may take a while.  When you are returned to 
your standard shell prompt run `ls` and if you see `db.out` you are ready to go.

_Go to step 2a for `homebrew` or step 2b for `postgres.app`_

## 2a: If you are using `homebrew` follow this path:

``` bash 
brew update && brew outdated
```
If you do not run this command on a regular basis you may be surprised by how many packages are 
outdated. In any event you should see `postgresql` listed among the packages.

Since you can't have two versions of postgres installed, you need to uninstall the old one first and then 
install the new version.  Do that like this:

``` bash
brew uninstall postgresql
brew install postgresql
```

Confirm that postgres is running and that you have the new version:
```bash
psql --version
psql (PostgreSQL) 9.4.0
```
If postgres will not respond you may need to restart your terminal to pick up the new changes, or follow
instructions issued after the homebrew install to manually start the postgres server.

If postgres is running but you cannot start the `psql` interface, try running `createdb` to fix this.

## 2b: If you are using the OS X native `postgres.app` 

Click the elephant icon in the OS X menubar and choose `quit` to stop the server.

Go to your Applications directory and find the Postgres app (the blue elephant icon). Rename it to
Postgres.old.

Go to http://postgresapp.com and download the latest version.  Double cick the downloaded zip file and drag
the new Postgres app into your Applications folder.

Double click the icon and wait for the application to start.  From either the elephant icon in the menu bar
or the `psql` command check the version number. It should be 9.4.0.

## 3: Import the SQL dump into the new server directory.

From the directory where you ran `pg_dumpall` in step 1 run the following command:

```bash
psql -f db.out postgres
```

Unlike the original `pg_dumpall` command you will see a lot of activity on the screen, including some warnings
that look like errors.  This is expected output, so be patient while the databases are recreated in the new 9.4
server directory.  

Once you get your command prompt back, go into the `psql` console and check you databases are all there the 
command `\l`. At a minimum you should have output like the example below, where one database typically has
your OSX or postgres admin username, two are templates, and one is postgres:

``` sql
$ psql
Null display is "[NULL]".
Expanded display is used automatically.
psql (9.4.0)
Type "help" for help.

localhost @David=# \l
                                       List of databases
            Name             | Owner | Encoding |   Collate   |    Ctype    | Access privileges
-----------------------------+-------+----------+-------------+-------------+-------------------
 David                       | David | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres                    | David | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0                   | David | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/David         +
                             |       |          |             |             | David=CTc/David
 template1                   | David | UTF8     | en_US.UTF-8 | en_US.UTF-8 | David=CTc/David  +
                             |       |          |             |             | =c/David
(4 rows)

localhost @David=# \q
$
```


##4: Clean up the export file and the old data directory

Verify that your applications can access your databases, and that everything is working as expected.
Once you are comfortable with that you have some cleaning up to do.

Backup or delete the `db.out` file.

If you are using `postgres.app` delete the `postgres.old` application from your Applications folder.

Finally using the `SHOW data_directory;` command from step 1, verify that the new directory is not the same
as the old one, and then you can safely delete the old database directory.  In my case the old directory was
`~/Library/Application Support/Postgres/var-9.3`, and the new one is now:

```sql
$ psql
Null display is "[NULL]".
Expanded display is used automatically.
psql (9.4.0)
Type "help" for help.

localhost @David=# SHOW data_directory;
                      data_directory
-----------------------------------------------------------
 /Users/David/Library/Application Support/Postgres/var-9.4
(1 row)
```

You can remove or backup the old directory at your discretion.

That completes your upgrade to 9.4.0.  For any patch revisions, i.e. `9.4.x`, that come later you can simply upgrade
the postgres application either via `homebrew upgrade postgresql` or by downloading the newer `postgres.app` and
installing it over the old version.

