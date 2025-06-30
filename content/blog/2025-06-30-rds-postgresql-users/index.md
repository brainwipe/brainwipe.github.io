---
title: "Changing user names in AWS RDS PostgreSQL"
date: 2025-06-30
categories: postgresql aws rds
---

If you want to replace users in AWS RDS Postgres, you need to use a particular process because AWS RDS does not give you access to the Postgress `SUPERUSER` role type, so you cannot use `REASSIGN OWNED` without following a grant process.

The process has three users:

1. A `super_user` that you will be logged in as to do the change
2. A `from_user` is the current owner of the database
3. A `to_user` is the next owner of the database

```sql
CREATE USER to_user;

GRANT rdsadmin TO super_user;
GRANT from_user TO super_user;
GRANT to_user TO super_user;

REASSIGN OWNED BY from_user TO staging_teto_usernant_user;

DROP ROLE from_user;
```

## Investigating users and ownerships

I found the following scripts helpful when working out what might have gone wrong.


### List roles with memberships

Can be used against any database (including the `postgres` one).

```sql
WITH roleagg AS (
    SELECT 
        m.member, 
        ARRAY_AGG(mr.rolname) role_name
    FROM pg_catalog.pg_auth_members m
    JOIN pg_roles mr ON (m.roleid = mr.oid) 
    GROUP BY 1
)

SELECT 
    r.rolname as user_name, 
    roleagg.role_name
FROM pg_catalog.pg_roles r
LEFT JOIN roleagg on roleagg.member = r.oid
WHERE r.rolcanlogin
ORDER BY 1;
```

Example Output (given users from above)

| user_name             | role_name                             |
| --------------------- | ------------------------------------- |
| rdsadmin              | NULL                                  |
| rdstopmgr             | ["pg_monitor"]                        |
| rdswriteforwarduser   | NULL                                  |
| super_user            | ["rdsadmin", "from_user", "to_user"]  |
| to_user               | NULL                                  |


### List databases with owners

```sql
SELECT 
    datname AS db_name, 
    pg_catalog.pg_get_userbyid(datdba) AS "owner" 
FROM pg_catalog.pg_database
ORDER BY 1;
```

Example Output (given users from above)
| db_name             | owner     |
| ------------------- | --------- |
| my_database         | to_user   |
| my_other_database   | to_user   |


### List database tables (and other objects)

Use with a single database.

```sql
SELECT 
    n.nspname AS schema_name,
    c.relname AS rel_name,
    pg_get_userbyid(c.relowner) AS owner_name
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE pg_get_userbyid(c.relowner) <> 'rdsadmin';
ORDER BY 1;
```

Example Output, rdsadmin schemas such as `pg_catalog` and `pg_toast` not included

| schema_name   | rel_name          | owner_name    |
| ------------- | ----------------- |-------------- |
| dbo           | my_dbo_table      | to_user       |
| public        | my_public_table   | to_user       |
