# Cassandra Schema Migration for Java [![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.cognitor.cassandra/cassandra-migration/badge.svg)](https://maven-badges.herokuapp.com/maven-central//org.cognitor.cassandra/cassandra-migration)

## Purpose
This library can be used to implement migrations for the Cassandra database schema inside your Java application.
The usage is oriented on the popular tools for relational databases like flyway or liquibase.

If you want to use this library with Spring Boot you can also scroll down to the description
of how to use the Spring Boot Starter.

## Usage
Using this library is quite simple. Given that you have a configured instance of the
cluster object all you need to do is integrate the next lines in your projects startup code:

```
Database database = new Database(cluster, "nameOfMyKeyspace");
MigrationTask migration = new MigrationTask(database, new MigrationRepository());
migration.migrate();
```

This assumes that all migration scripts can be found on the classpath in a folder '/cassandra/migration'.
If you put your scripts on a different classpath location you just need to pass the path in the constructor
like this:

```java
new MigrationRepository("/my/path/here");
```

## Naming
Scripts should be named in the following schema:

```
<version>_<name>.cql
```

If the '.cql' extension is missing the file will be ignored. The 'version' is required to figure out the latest
version of the scripts and relates to the version that is stored in the database schema information table.
The version should start with one as an empty database is considered to have a version of
zero. Leading zeros for better sorting are accepted.
The name is something that is just for the developers purpose and should be something descriptive.

In case there are multiple scripts with the same version (duo to a merge of branches for example),
an exception is thrown immediately. This behavior can be changed by creating an instance of the
MigrationRepository providing a ScriptCollector implementation like this:
```java
new MigrationRepository(MigrationRepository.DEFAULT_SCRIPT_PATH, new IgnoreDuplicatesCollector());
```
Until the version 1.0.2 inclusive the default behavior was to ignore duplicates by considering
only the first script file for a particular version. As this can lead to unpredictable behavior, since
it is just a matter of which script is found first, this behavior is no longer the default.

## Script content
The script format is rather simple. It allows one statement per line and lines should be finished
with a ';' character. Every line that is not empty and is not a single line comment will be executed against the Cassandra instance.
Single line comments are indicated by either '//' or '--' characters.
Multi line comments are not supported.

## Migrations
Migrations are executed with the Quorum consistency level to make sure that always a majority of nodes share the same schema information.
Error handling is not really implemented (and as far as I know not really possible from a database point of view).
If one script fails the migration is stopped and an exception is thrown. The exception contains the name of
the failing script as well as the broken statement in case the error happened during the execution of a
statement.
Every script will result in an entry into the schema_migration table. If a script fails, an entry will be put
into the 'migration_schema' table stating that this script failed. You can then fix the script and retry the migration.
It should normally not be necessary to remove failed migrations from the 'migration_schema' table.

However, in case you have multiple statements in one script and one of them failed you need to make sure that
the statements before the failing one are safe to be executed again. You either need to manually revert
the actions or, the preferred approach, make use of Cassandras "IF EXISTS" or "IF NOT EXISTS" mechanism to
ensure that the same script can be run multiple times without failing.

## More details
The library checks if there is a table inside the given keyspace that is called "schema_migration". If it
is not existing it will be created and it contains the following columns:
* applied_successful (boolean)
* version (int)
* script_name varchar
* script (text)
* executed_at (timestamp)

"applied_successful" and "version" together make the primary key. The version of the database schema is equivalent
to the highest number returned by the version column where applied_successful is true.
This means, even if your counting does not start at one (because you removed some very old scripts)
the schema version is not affected by this.

All migrations that are marked as applied_successful = false do not affect the version number in any way. It is also
perfectly legal to have the same version number once with a successful execution and one with a failing execution,
for example, if the first try failed and the script was fixed afterwards. However, you will only see the last failing
execution. If the same script fails twice the first failure will be overwritten.

## Maven
If you are using maven you can add cassandra-migration as a dependency to your project like this:
```xml
  <dependency>
      <groupId>org.cognitor.cassandra</groupId>
      <artifactId>cassandra-migration</artifactId>
      <version>2.1.1</version>
  </dependency>
```

## Spring Boot
Cassandra Migration comes with a Spring Boot Starter module that can be used to autoconfigure
the migration. You have to include the following dependency to make it work:
```xml
  <dependency>
      <groupId>org.cognitor.cassandra</groupId>
      <artifactId>cassandra-migration-spring-boot-starter</artifactId>
      <version>2.1.1</version>
  </dependency>
```

In your properties file you will have three new properties that can be set:
* cassandra.migration.keyspace-name Specifies the keyspace that should be migrated
* cassandra.migration.script-location Overrides the default script location
* cassandra.migration.strategy Can either be IGNORE_DUPLICATES or FAIL_ON_DUPLICATES
