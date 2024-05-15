---
title: Create and Host a Local "Snapshot" of the Sierra ILS Database Using Docker
date: 2024-05-15
categories: [Library Data, Docker]
tags: [data, docker, sierra]     # TAG names should always be lowercase
image:
  path: /vibrant and colorful image illustrating Docker and PostgreSQL integration. The image features a Docker whale logo and a PostgreSQL elephant logo int.webp
  alt: A Elephant and a Whale Walk into a Cloud ...
math: false
author: <ray_voelker>
pin: false
---

Sometimes you just want to be able to pluck a bit of data from a "previous version" of your data without having to restore the entire system. Below are the steps for creating a local version of the previous state of a Sierra PostgreSQL Database:


## Create a PostgreSQL Docker container and Restore the Database Within It 

1. We'll need a **copy of the database dump** that Sierra creates in the nightly backup process
   
   If you have access to the server, the backups are contained on the host where your database runs.
   
   In this example, I'll be using the hostname `sierra-db` to represent the database server from which we'll be getting our backup data. In an improved version, you'll likely want to use something like object storage to keep backups for more archival purposes.

   **Copy the data to your target machine / host where you want to run this Docker container:**

   ```bash
   $ cd ~/
   $ mkdir -p sierra-prod-db
   $ cd sierra-prod-db
   $ rsync -Pavz -e "ssh" sierra-db-rh8-bastion:/iiidb/sqlbackup .
   ```

1. You'll need Docker installed locally, along with the plugin `docker-compose`. Below is the `docker-compose.yml` file that I'm using for the version of PostgreSQL that Sierra is using for this particular dump (NOTE: to improve this, you may want to note the version of PostgreSQL as part of your backup process)

    `docker-compose.yml`:
    ```yaml
    version: '3.3'

    services:
      postgres:
        image: postgres:11.6
        container_name: postgres-11.6
        environment:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: iii
        # optimized for restore -- don't use these settings for any sort of production use
        command: >
          -c work_mem=256MB
          -c maintenance_work_mem=256MB
          -c max_wal_size=2GB
          -c min_wal_size=1GB
          -c shared_buffers=512MB
          -c effective_cache_size=2GB
          -c synchronous_commit=off
          -c fsync=off
          -c wal_compression=on
        volumes:
          # NOTE: adjust this to your own paths for where the dumps are located ...
          - /home/ray/sierra-prod-db:/var/lib/postgresql/dumps
        ports:
          - "127.0.0.1:5432:5432"
    ```

1. Bring the container up:

    ```bash
    $ docker-compose up -d
    ```

    The above command should fetch the necessary images, and bring the container up.

1. Start restoring the database within the container image:

    **NOTE**: This configuration does not "preserve" the database in the image -- you may want to do that by using another volume to preserve the database so that it's not purged when cleaning up if that's something that you need. 

    ```bash
    $ docker-compose exec postgres psql -U postgres -f /var/lib/postgresql/dumps/sqlbackup/roles.sql
    $ docker-compose exec postgres bash -c "pg_restore -U postgres -d iii /var/lib/postgresql/dumps/sqlbackup/iii.dump"
    ```

    This will take a ~~few minutes~~ fairly long while, depending on the size of you database, the speed of your disks, etc -- on my system, it took just under an hour to perform the full restore (`11 GB` .sql file, resulted in a database size of just over `100 GB`) 

    **NOTE**: You may notice a few errors -- it seems likely this is due to other extensions not being loaded -- a further improvement may be to find out what extensions are loaded on the running database, and load those before running the restore in the local container.

1. You may need to "reset" the default postgres user password after the database has been restored (**NOTE**: again, just setting the password to `postgres` should be fine for localhost use): 

    ```bash
    $ docker-compose exec postgres psql -U postgres -c "ALTER USER postgres WITH PASSWORD 'postgres';"
    ```

## Connecting / Using the Database

The database should now running on localhost within the Docker container.

You can now point whichever client you wish at the container which is bound to the localhost IP of `127.0.0.1` on port `5432`:

For example, if you have the PostgreSQL client installed locally, you can use the following command:

```bash
$ export PGPASSWORD=postgres
$ psql -h 127.0.0.1 -U postgres -d iii
```

Now, you can issue SQL commands:

```bash
iii=# select id, record_type_code, creation_date_gmt from sierra_view.record_metadata order by id desc limit 10;
        id         | record_type_code |   creation_date_gmt    
-------------------+------------------+------------------------
 49258601960828851 | p                | 2024-05-14 18:32:00+00
 49258601960828822 | p                | 2024-05-14 13:28:06+00
 49258601960828763 | p                | 2024-05-13 16:50:46+00
 49258601960828703 | p                | 2024-05-12 17:50:02+00
 49258601960828597 | p                | 2024-05-10 19:50:12+00
 49258601960828517 | p                | 2024-05-09 19:25:18+00
 49258601960828400 | p                | 2024-05-08 15:54:35+00
 49258601960828165 | p                | 2024-05-05 17:43:23+00
 49258601960828131 | p                | 2024-05-04 19:31:52+00
 49258601960828093 | p                | 2024-05-04 13:49:07+00
(10 rows)
```






   





