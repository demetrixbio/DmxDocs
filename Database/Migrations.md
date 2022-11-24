[[_TOC_]]


# Automatic creation

The `Demetrix.Lims.Migrations` console app project provides DbUp integration for automatic management of db migrations.

If the database does not exist, it will be created with the `read_write` owner from provided configuration, and initialized with a baseline script that can be found in the `Baseline` folder.  Configuration for db migrations is stored in `appsettings.json`.  Below you can find an example of a valid configuration setup with optional WarehouseConnection section:

```
{
  "DatabaseConfig": {
    "MasterConnectionString": "Host=localhost;Username=postgres;Password=postgres;Database=postgres",
    "LimsConnectionCredentials": {
      "LimsDbName": "lims",
      "ReadWriteUserPassword": "readwrite",
      "ReadOnlyUserPassword": "readonly"
    },
    "WarehouseConfig": {
      "WarehouseConnectionString": "Host=localhost;Username=postgres;Password=postgres;Database=postgres",
      "WarehouseDbName": "ceres",
      "ReadWriteUserPassword": "readwrite",
      "ReadOnlyUserPassword": "readonly",
      "ReplUserPassword": "logicalrepl",
      "SetupReplication": true
    },
    "MaskSecrets": false
  }
}
```


# Configuration

You can overwrite the database credentials locally by adding following environment variables:

* `demetrix_lims_migrations_DatabaseConfig__MasterConnectionString`: The master connection string that can connect to the postgres database  (default in `appsettings.json`)
* `demetrix_lims_migrations_DatabaseConfig__LimsConnectionCredentials__LimsDbName`: The name of your local database (default `lims`)
* `demetrix_lims_migrations_DatabaseConfig__LimsConnectionCredentials_ReadWriteUserPassword`: The password of your readwrite user (default in `appsettings.json`)

Examples:

* `export demetrix_lims_migrations_DatabaseConfig__MasterConnectionString="Host=localhost;Username="..."`
* `export demetrix_lims_migrations_DatabaseConfig__LimsConnectionCredentials__LimsDbName="testlims"`
* `export demetrix_lims_migrations_DatabaseConfig__LimsConnectionCredentials__ReadWriteUserPassword="..."`

It is not possible to put a colon (`:`) in an env variable (confirmed for Windows and Mac) and you 
should replace it with a double underscore (`__`) as above.  


# Creating migrations

The `src/Lims/Migrations/Migrations` folder contains folders with individual updates to the schema.  Prior to version 1.9 these were arranged in folders by version number, but now they are arranged by month created.

To create a new migration:

In git bash or wsl or terminal, run 

```shell
dotnet fsi add-migration.fsx --name=migration_name lims
```

These commands invoke FAKE `build.fsx` with `AddMigration` target, that will

* Create a pair of sql migration scripts (one for prod one for warehouse) prefixed with current datetime ticks and with given name.
* Include them as embedded resources to folder with name = current utc date in 'yyyy-MM' within Demetrix.Migrations project.
* The production migration will be `tick_name.sql` and the matching warehouse migration `tick_W_name.sql`

Upon running migrations, DbUp will ensure that migrations will not reapply if they have already been applied.

The migration script has several run options, to get more information, please refer to the output of running:

```shell
dotnet fsi add-migration.fsx --help
```

> When adding a migration to alter an ENUM type, you may get the following error `ALTER TYPE ... ADD cannot run inside a transaction block`. [A workaround can be found here](Migrations/v6/636948231994967950_add_stopped_to_harvest_experiment_stage.sql).


# Applying migrations

In order to apply migrations, you can either invoke FAKE `build.fsx` with `RunMigrations` target, or execute `dotnet run` against Demetrix.Migrations.fsproj. Connection string for database, against which the migrations will be executed is stored in `appsettings.json`. Order of scripts execution is alphanumeric. As new migrations are created with DateTime.UtcNow ticks prefix, - it will ensure that scripts are executed in a same order as they were created.
There are two optional arguments that can be passed:
* TransactionBehavior (--transaction | -t) - Single | PerScript. Scripts will be executed either within single transaction for entire migration or transaction per migration script.
* MaxScripts (--max | -m) - uint value, that specifies how many new scripts should be applied during migration. Defaults to execution of all new scripts.
Example invocation of RunMigrations target with optional parameters: `.\build runmigrations --max 10 --transaction perscript`


# Warehouse migrations

Migrations come in pairs (like Sith) - one to update production and one (possibly null or the same) to change the warehouse equivalent structures.

Any changes to the database definition itself [DDL](https://en.wikipedia.org/wiki/Data_definition_language) 
in a migration must be repeated in a separate warehouse migration. The table definitions must match in
the production and warehouse versions.  For example a column,schema,enum or table added or removed to
production should also be removed from the warehouse mirror versions.  This may be a subset of
your main production migration, so it is repeated in the separate `_W_` version

By implication, any dependent structures in the warehouse should also be considered during updates to the 
production tables, since adding or removing, renaming a table for example could break a dependent structure
in the warehouse.  Those changes would also be included in the warehouse side of the migration pair.

Conversely, any data manipulation statements [DML](https://en.wikipedia.org/wiki/Data_manipulation_language)
should occur only in the primary production migration and would not be repeated in the warehouse `_W_` version.
These include ontology entries, equipment or regular transformations of existing rows in the database.
The warehouse will receive those updates through the regular [logical replication mechanism.](https://www.postgresql.org/docs/current/logical-replication.html)


# Q&A

Q: If we make a migration that renames a table, do we have to re-publish it under the new name?

A: No. The publication is stored as a reference to the underlying table Id.


# Troubleshooting information

You will need to activate wal_level logical on the databases you are targeting.

Othewise migration setup of the warehouse will fail with this message:

`Migrations failed. 55000: logical decoding requires wal_level >= logical`

If you are running conventional postgres, change the `wal_level` config value to `logical` in the `postgreslql.conf` config file.   For AWS instances, there is a process for switching it via the web UI and it requires a reboot to activate.

```
# Bash function for standing up a temporary (no permanent storage backing) postgres instance in docker with wal_level logical
docker_run_postgres() {
     docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=postgres -d -p 127.0.0.1:5432:5432 postgres:12 -c wal_level=logical
}
```

1. `"MasterConnectionString" : "Host=localhost;Username=user;Password=password;Database=postgres"` - This connection must have permissions to create pg extensions
2. `"LimsDbName": "lims"` - Database you are running migrations against (db may not exist)
3. `"ReadWriteUserPassword": "readwrite"` - read_write user password (owner name is hardcoded in baseline dumps, so no point making it configurable. if user does not exist - he is also created)
4. `"MaskSecrets"` - should migration app report full connection credentials to logs
5. `WarehouseConnection`: connection string, database name and password for schema maintenance user.  Connection string is for super user (can create extensions and do db maintenance).  This whole section is optional

* The Db migration process is transactional per script file, so you do not need to begin and rollback transaction inside the script neither insert record into `migrations_applied` table. It's handled automatically.
* All migration scripts that are executed against target db are packaged in a `src/Lims/Migrations/Demetrix.Lims.Migrations.fsproj` as embedded resources. If you need to exclude certain migration script from execution (e.q. for debugging purposes), - you can just comment that `<EmbeddedResource Include="script_name" />` line in fsproj. Once uncommented, it will be automatically picked up by migrations. One thing to consider - excluding certain migrations might have bad outcomes, if migrations below given one do depend on changes that took place in commented migration.
* If you are targeting a db, protected by SSL certificate, - you need to specify path and name of the certificate in `appsettings.json`. (same as in main `Demetrix.Lims.Web` project)
* It is important to create new migrations using the commands in the root of the project. More detailed info can be found in `Creating new migrations` section.
* Upon merging master to your branch (or vise versa), - it is helpful to run migrations before pushing to ensure that there are now collisions between your new migration and someones merged one, that was not yet applied to your db.







