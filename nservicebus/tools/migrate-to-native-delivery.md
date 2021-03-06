---
title: Migrating from timeout manager to native delivery
summary: An overview of the tool supporting migrating from timeout manager to native delivery
reviewed: 2020-06-22
---

The timeout migration tool is designed to help system administrators migrate existing timeouts from the legacy [Timeout Manager](/nservicebus/messaging/timeout-manager.md) storage to the [native delayed delivery](/nservicebus/messaging/delayed-delivery.md) infrastructure of the currently used transport.

[Native delayed delivery](/nservicebus/messaging/delayed-delivery.md) was introduced in NServiceBus version 7 across most supported transports and a hybrid mode was made available which is enabled by default. When running in hybrid mode, endpoints consume timeouts that were registered in the system using the legacy [Timeout Manager](/nservicebus/messaging/timeout-manager.md) while new delayed messages flow through the native implementation.

NOTE: Make sure to use the `preview` option of the tool to make sure that the transport supports native delayed delivery before performing any migration attempt.

Most of the timeouts that were registered through the legacy timeout manager might have been consumed by now. But there might be scenarios in which there are timeouts waiting to expire, and those are stored in the timeout storage.
For those use cases, the timeout migration tool enables migrating timeouts to the native delayed delivery infrastructure so that the storage can be decommissioned.

The tool supports live-migration in most cases, so there's no need to shut down the endpoints before running the tool. The tool will hide the timeouts it will migrate from the legacy Timeout Manager to eliminate duplicate deliveries in the system.

WARN: Migrating large amounts of timeouts on RavenDB might require endpoints to be shutdown to improve performance. Run the `preview` command to find out if this is the case.

It's important to note the definition of an endpoint to migrate in the context of the tool. The legacy [Timeout Manager](/nservicebus/messaging/timeout-manager.md) stored timeouts at the sending side and sent them out to the destination endpoint at delivery time. This means that the endpoint names listed by the tool is for the endpoints **sending** the delayed message and not the destination.

Example:

Let's say there is a Sales endpoint that requested a timeout to be delivered to the Billing endpoint. The Billing endpoint is not requesting any timeouts.
Using the legacy [Timeout Manager](/nservicebus/messaging/timeout-manager.md), this means that the timeouts will be sent out by the Sales endpoint to the Billing endpoint when the delivery time is reached.
The tool will list the Sales endpoint as one of the options to migrate. Given that the Billing endpoint does not send timeouts, it won't be listed.
The tool will check that the Billing endpoint has the necessary infrastructure in place to handle native delivery.

The tool optionally supports the use of the `--cutoffTime`. This is the starting point in time from which timeouts become eligible to migrate, based on the time to deliver set in the timeout.

There are two main reasons to make use of this:

- SLA compliance: In case there are many timeouts in the storage to migrate, it might take some time for the migration to complete. Since the timeouts are first hidden from the legacy [Timeout Manager](/nservicebus/messaging/timeout-manager.md) and then migrated, this might result in some timeouts being delivered later than their original delivery time in case of large migrations.
- Phasing the migration: In case of big loads of timeouts to migrate, it might be interesting to run a phased migration based on the original delivery time of the timeouts. This can be achieved by setting the `--cutoffTime` to a far point in the future, and decrease it each run.

## Supported persisters

The current version of the tool supports the following persisters:

- [SQL Persistence](/persistence/sql/) using the SQL Server implementation
- [RavenDB](/persistence/ravendb) versions 3.5.x and 4.x of the RavenDB database server

## Supported transports

The tool supports the following transports:

- [RabbitMQ](/transports/rabbitmq/)

## Before using the tool

Even though the tool doesn't delete any timeout information when doing the migration it is recomended to follow industry standards related to modifying the database. It is advisable to create a backup of the production database and first run the migration on a test environment.

## How to install

`dotnet tool install Particular.TimeoutMigration -g`

To verify if the tool was installed correctly, run:

`dotnet tool list -g`

Verify the tool is listed among the available installed tools.

## Using the tool

The migration tool provides a `preview`, `migrate` and `abort` command.

Depending on the persistence, there are additional parameters required in order to run the tool:

For RavenDB:

- `--serverUrl`: The RavenDB server URL
- `--databaseName`: The database name where timeouts to migrate are stored
- `--ravenVersion`: The allowed values are "3.5" and "4"
- `--prefix`(optional): The prefix used for storage of timeouts. The default value is "TimeoutDatas"

For SQL:

- `--source`: The connection string to the database
- `--dialect`: The SQL dialect used to access the database. Supported dialects: `MsSqlServer`

### Preview

To get a preview of endpoints and their status use the `preview` command with the following extra parameters.

- `--target`: The connection string of the target transport used to validate destinations

**RavenDB**

```
migrate-timeouts ravendb preview
                        -t|--target <targetConnectionString>
                        --serverUrl <serverUrl>
                        --databaseName <databaseName>
                        --ravenVersion <ravenVersion>
                        [--prefix] <prefix>
```

**Sql Persistence**

```
migrate-timeouts sqlp preview
                        -t|--target <targetConnectionString>
                        --source <source>
                        --dialect <sqlDialect>
```

NOTE: The listed endpoints will be the in the escaped form that is used to prefix the endpoints timeout table

### Running a migration

To run a migration for selected endpoint(s) use the `migrate` command with the following parameters.

- `--target`: The connection string of the target transport
- `--endpoint`(Optional): The endpoint to migrate.
- `--allEndpoints`(Optional): Indicates to migrate all endpoints in one run
- `--cutoffTime`(Optional): The time from which to start migrating timeouts, it makes sense to start migrating timeouts that will expire at least one day in the future. The format in which to specify the `cutoffTime` is `yyyy-MM-dd HH:mm:ss`. The migration tool will convert the specified `cutoffTime` to UTC time.

NOTE: `--endpoint` and `--allEndpoints` arguments are mutually exclusive, either one is required.

**RavenDB**

- `forceUseIndex`(Optional): Required when migrating large amounts or timeouts. Requires all endpoints using the database to be turned off to not modify the timeout data.

```
migrate-timeouts ravendb migrate
                        -t|--target <targetConnectionString>
                        --serverUrl <serverUrl>
                        --databaseName <databaseName>
                        --ravenVersion <ravenVersion>
                        [--prefix] <prefix>
                        [-c|--cutoffTime <cutoffTime>]
                        [--endpoint] <endpointName>
                        [--allendpoints]
                        [--forceUseIndex]
```

**Sql Persistence**

```
migrate-timeouts sqlp migrate
                        -t|--target <targetConnectionString>
                        --source <source>
                        --dialect <sqlDialect>
                        [-c|--cutofTime <cutoffTime>]
                        [--endpoint] <endpointName>
                        [--allendpoints]
```

NOTE: The listed endpoints will be the in the escaped form that is used to prefix the endpoints timeout table

### Aborting a migration

To abort an ongoing migration use the `abort` command.

**RavenDB**

```
migrate-timeouts ravendb abort
                        --serverUrl <serverUrl>
                        --databaseName <databaseName>
                        --ravenVersion <ravenVersion>
                        [--prefix] <prefix>
```

**Sql Persistence**

```
migrate-timeouts sqlp abort
                        --source <source>
                        --dialect <sqlDialect>
```

## How the tool works

The migration tool will first perform a few health checks:

- verify it's able to connect to the storage
- verify it's able to connect to the target transport
- verify that the target transport supports native delayed delivery
- check that necessary infrastructure for native delays is setup for all delayed message destinations found
- list all the endpoints for which the tool can detect timeouts
- calculate the amount of timeouts to migrate per endpoint
- validate if there are timeouts the tool is [unable to migrate](migrate-to-native-delivery.md#limitations)

Even though the tool supports migrating all endpoints connected to the persistence at once, it is highly suggested to migrate endpoint by endpoint, especially for critical endpoints. Even when selecting the `--allEndpoints` option, the tool will execute an endpoint-by-endpoint migration behind the scenes.

## Cleanup

The tool will not delete any timeouts or storage artifacts to prevent data loss. This section describes how to cleanup archived timeouts and remove no longer used storage artifacts.

WARN: This is a destructive operation and should only be performed once it has been verified that the migration has been successful.

### RavenDB

- Make sure that [RabbitMQ compatibility mode](/transports/rabbitmq/delayed-delivery.md#backwards-compatibility) is turned off
- Delete all the documents in the `TimeoutDatas` documents that have an `OwningTimeoutManager` starting with `__migrated__`.

### Sql persistence

Use `SELECT * FROM TimeoutsMigration_State` to list all performed migrations. For all the successful ones do the following:

- Make sure that [RabbitMQ compatibility mode](/transports/rabbitmq/delayed-delivery.md#backwards-compatibility) is turned off
- Delete the empty `{EndpointName}_TimeoutData` table
- Delete the migration table named `TimeoutData_migration_{MigrationRunId}`, where `MigrationRunId` is taken from the output of the `TimeoutsMigration_State` query (this will free up the diskspace used by the timeouts).

## Limitations

As documented in the [RabbitMQ transport](/transports/rabbitmq/delayed-delivery.md), the maximum delay value of a timeout is 8 and a half years. If the migration tool encounters any timeouts that have delivery time set beyond that, it will not migrate that endpoint's timeouts.

If the tool presents endpoints that are not part of the system when running the `preview` command, it might be that an endpoint was renamed at some point.
Any timeouts that were stored for that endpoint, might already be late in delivery and should be handled separate from the migration tool since the tool has no way to detect where to migrate them to.

## Troubleshooting

If the migration started but stopped or failed along the way, the migration tool can recover and continue where it left off. To resume an interrupted migration the tool must be run with the same arguments.

To run the tool with different arguments, any in-progress migrations first need to be aborted using the `abort` command. Any timeouts that have been fully migrated at that point will not be restored since they have already been migrated to the native timeout infrastructure. Timeouts that were scheduled to migrate will be made available again to the legacy Timeout Manager.

### Logging

Turn on verbose logging using the `--verbose` option.

### RavenDB

Use the Raven Studio to check the state of ongoing and previous migrations by filtering documents using the prefix `TimeoutMigrationTool`. Completed migrations will be named `TimeoutMigrationTool/MigrationRun-{endpoint}-{completed-time}` while any ongoing migration will be present as `TimeoutMigrationTool/State`. The contents of the documents contains metadata about the migration such as the time started, time completed, used cutoff time etc.

### Sql Persistence

The history and migrated data is always kept in the database.

To list the history and status of migrations execute:

`SELECT * FROM TimeoutsMigration_State`

To list the status of timeouts for an a previous/in-progress run take the `MigrationRunId` from the query about and execute:

`SELECT * FROM TimeoutData_migration_{MigrationRunId}`

This will show all the timeouts and to which batch they belong and also that status of that batch, `0=Pending`, `1=Staged` and `2=Completed`.
