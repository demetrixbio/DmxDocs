[[_TOC_]]

# Docker database setup for development

The basics for docker database setup are in the project README.  This document provides more options and details.

First, download and install [Docker](https://docs.docker.com/install/).

Create an empty folder (for example `/Users/username/Documents/Demetrix_SRC/postgres_docker_dev/`) and create a `docker-compose.yml` file in it.  The contents of this file tell docker how to set up Postgres in a container, along with the Pgadmin tool, and some folders mapped into your local filesystem for easy access.

It should look like this:

```yaml
version: '3.3'
services:
    postgres:
        image: postgres:12.3
        restart: always
        container_name: demetrix_postgres
        ports:
          - "5432:5432"
        environment:
          - POSTGRES_USER=postgres
          - POSTGRES_PASSWORD=password
          - POSTGRES_DB=postgres
        volumes:
          - ./data-12:/var/lib/postgresql/data
          - ./shared_access_folder:/tmp/shared_access_folder
    pgadmin:
        image: dpage/pgadmin4:6.12
        container_name: demetrix_pgadmin
        environment:
            PGADMIN_DEFAULT_EMAIL: 'postgres@postgres.net'
            PGADMIN_DEFAULT_PASSWORD: 'password'
            PGADMIN_DISABLE_POSTFIX: 1
            PGADMIN_CONFIG_LOGIN_BANNER: '"Your Local DMX Database In Docker"'
            PGADMIN_CONFIG_ENABLE_PSQL: 1
        ports:
          - 8081:80
        volumes:
          - ./pgadmin_servers.json:/pgadmin4/servers.json # preconfigured servers/connections
          - ./pgadmin_pgpass.txt:/pgpass # passwords for the connections in the above file
          - ./pgadmin_storage:/var/lib/pgadmin/storage
        depends_on:
          - "postgres"
```

Note that the volume links point to three folders inside the one you created.

In addition to the `docker-compose.yml` you just created, you need to create two more files in this directory:

A file `pgadmin_servers.json` with the following contents:

```json
{
  "Servers": {
    "1": {
      "Name": "Demetrix Local Docker",
      "Group": "Servers",
      "Host": "demetrix_postgres",
      "Port": 5432,
      "MaintenanceDB": "postgres",
      "Username": "postgres",
      "PassFile": "/pgpass",
      "Comment": "Connection to your local Demetrix test database in Docker, as user postgres",
      "SSLMode": "prefer"
    }
  }
}
```

And a file `pgadmin_pgpass.txt` with this content:

```
demetrix_postgres:5432:postgres:postgres:postgres
```

Once these files are saved, go into the folder containing them and run `docker-compose up -d` to read in and run the configuration.  Docker will fetch Postgres, PgAdmin, and supporting packages, install them into a virtual image, and launch the database.

<p>
<details>
<summary>Additional troubleshooting information</summary>

Be careful with byte order marks in the `docker-compose.yml` file as these can lead to confusing error messages from `docker-compose -d` like: `>service ‘image’ must be a mapping not a string.`

There's a small chance you'll get an error with a strange Python stack trace when you try to run `docker-compose up`:

![odd_python_error_from_docker_compose](uploads/71061b7993574a0211362c617c0293c6/odd_python_error_from_docker_compose.png)

If you see this, try opening the Docker Desktop application that was installed when you installed Docker, and see if there are any lingering containers, perhaps from old configuration files that have been deleted.  You may have to delete these containers by hand using `docker rm --force (container name)`.

</details>
</p>

## Useful commands for working with the Postgres instance in Docker

To connect to the database using the psql inside the docker container (recommended):<ul><li>`docker exec -it demetrix_postgres psql -h localhost -U username -d lims`</li></ul>

To connect to the database using your native install (outside docker) of psql:<ul><li>`docker run -it --rm --link demetrix_postgres:postgres postgres psql -h postgres -U username -d lims`</li></ul>

To run a database restore in two steps, by logging into the container and then running the restore (recommended):<ul><li>`docker exec -it demetrix_postgres /bin/bash`</li><li>`pg_restore -h localhost -d lims -U postgres -c -v -C < /tmp/shared_access_folder/Dmx_Production_Lims-2019-10-14.sql`</li></ul>

Same as above, but restoring from a folder instead of an `.sql` file:<ul><li>`docker exec -it demetrix_postgres /bin/bash`</li><li>`pg_restore -h localhost -U postgres -d lims -v -C -c -j 16 /tmp/shared_access_folder/lims_prod_20220809_reduced`</li></ul>

To run a database restore operation from outside the docker container:<ul><li>`docker run -i --rm --link demetrix_postgres:postgres postgres pg_restore -h postgres -U postgres -c -v -C < /Users/gbirkel/Documents/Dmx_Production_Lims-2019-10-14.sql`</li></ul>

<p>
<details>
<summary>Additional troubleshooting information</summary>

If you're on Windows using a shell like gitbash, you may get an error like `the input device is not a TTY.  If you are using mintty, try prefixing the command with 'winpty'`.

You have two workarounds for this.  Either run the command in a Powershell terminal window instead, or do what it says and prefix the command with `winpty`, e.g. `winpty docker exec -it demetrix_postgres psql -h localhost -U username -d lims`

You can also run `winpty powershell` and then run commands inside that.  This might help where the above combinations don't.

</details>
</p>

<p>
<details>
<summary>Additional troubleshooting information</summary>

If you're on Windows using a shell like gitbash, you may get an error like `the input device is not a TTY.  If you are using mintty, try prefixing the command with 'winpty'`.

You have two workarounds for this.  Either run the command in a Powershell terminal window instead, or do what it says and prefix the command with `winpty`, e.g. `winpty docker exec -it demetrix_postgres psql -h localhost -U postgres -d postgres`

</details>
</p>

Now your first restore will work.  To restore from the command line, copy your dump file into the `shared_access_folder` that was created by Docker, and run commands like so:

```bash
docker exec -it demetrix_postgres /bin/bash;
cd /tmp/shared_access_folder
pg_restore -d lims -U postgres -c -v -C < Dmx_Production_Lims-2019-10-14.sql
```

The first command launches `bash` inside the image, and the third launches the `pg_restore` command inside the image and sends the dump file to it as input.

If you don't have a dump to restore, the migrations process built into the codebase will create a basic database for you with all the essential records to operate, when you trigger the first build of the server.

Sometimes restore operations can take an extremely long time.  To speed up a restore, you may wish to add some custom configuration to the `postgres` service in your `docker-compose.yml` file:

```bash
command: -c 'maintenance_work_mem=1GB' -c 'work_mem=1GB' -c 'temp_buffers=2GB' -c 'shared_buffers=4GB' -c 'wal_level=minimal' -c 'fsync=off' -c 'full_page_writes=off' -c 'synchronous_commit=off' -c 'max_wal_senders=0'
```

Set the first four values to a reasonable amount for your computer.  The rest are for reducing the transaction log, which will likely make your database unable to recover in the event of a crash.  (These settings will also making DML significantly faster in everyday use.)  Considering you are probably using "disposable" databases in your local dev environment, the resilience issue is not a big deal.  However, you may feel safer setting these parameters just for the duration of your `pg_restore` operation, then removing them.

# Other useful docker/psql information:

## Making a disposable docker database

As an alternative to setting up a full docker-compose config with permanent storage, you can get a lightweight (throw away) postgres instance that has no permanent storage (when the docker image ends the database contents are all gone).  This can be helpful for quick testing of a new branch where you want to build the db from scratch, or troubleshooting migrations etc.  It's low commitment. 

If you add this to your `~/.bash_profile`  (first time you will need to also `source ~/.bash_profile`), you can get a db with the command `docker_run_postgres`.   You can kill the session with `docker ps` and then `docker kill <id#>`.  Zero commitment.

```
# Bash function for standing up a temporary (no permanent storage backing) postgres instance in docker with wal_level logical
docker_run_postgres() {
     docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=password -d -p 127.0.0.1:5432:5432 postgres:12 -c wal_level=logical
}

```

If editing your bash file is too much, work, all you really need to type is

`docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=password -d -p 127.0.0.1:5432:5432 postgres:12 -c wal_level=logical `
Note - this config also enables logical replication which is helpful / necessary if you are using the warehouse features.

