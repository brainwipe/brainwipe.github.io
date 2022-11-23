---
title: "Find out what SQL queries are slow on PostgreSQL locally"
date: 2022-11-23
categories: postgresql statistics windows
---
Your cloud provider is showing you that some of your PostgreSQL queries are slow. Your cloud provider is showing you SQL, and you might have set [`track_activity_query_size`](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.UsingDashboard.SQLTextSize.html) to something larger than 4096 bytes, so you can see your larger SQL statements.

Now you want to reproduce locally but where can you get those statistics from?

The process is much the same for Windows and other OSes but I'll be doing it on Windows.

## Switch on `pg_stat_statements`

- Find Postgres configuration file

```powershell
psql -U postgres -c 'SHOW config_file'
# C:\Program Files\PostgreSQL\13\data\postgresql.conf # For Postgres 13 on Windows 10
```

- Open the Postgres configuration file.
- Find `shared_preload_libraries`, it's probably commented out
- Uncomment the line and change to:

```inf
shared_preload_libraries = 'pg_stat_statements'	# (change requires restart)
```

## Restart PostgreSQL

For the extension to be available, you need to restart the Postgres server. Use the data directory of your Postgres installation. For Postgres 13 on Windows 10:

```powershell
pg_ctl -D "C:\Program Files\PostgreSQL\13\data" restart
```

Restarting your computer will also work. You also have these commands for manual stop/start:

```powershell
pg_ctl -D "C:\Program Files\PostgreSQL\13\data" stop
pg_ctl -D "C:\Program Files\PostgreSQL\13\data" start
```

## Create extension in database

Open your favourite editor (I use [DataGrip by Jetbrains](https://www.jetbrains.com/datagrip/)), select your database and run the following query to switch on the stats creation:

```sql
CREATE EXTENSION pg_stat_statements;
```

## Get basic statistics

From this point on, Postgres will start logging statistics on queries. To get at those queries, use:

```sql
select * from pg_stat_statements
```

You can now google for uses of `pg_stat_statements` to drill down for your specific needs.

## Troubleshooting

If you see the error:

    database> CREATE EXTENSION pg_stat_statements
    [2022-11-23 11:21:01] completed in 78 ms
    database.public> select * from pg_stat_statements
    [2022-11-23 11:21:43] [55000] ERROR: pg_stat_statements must be loaded via shared_preload_libraries

Then it means you need to switch on `pg_stat_statements`, see above.