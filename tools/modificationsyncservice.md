# ModificationSyncService

This is a simple REST service that provides a way to synchronize data modifications between multiple applications. The SDACK business framework directly supports this service so that it can be used to prevent concurrent modifications by separate applications. It comes with a command line tool that can be used to administrate a running sync service.

* [ModificationSyncServiceEndpoint](modificationsyncservice.md#modificationsyncendpoint)
* [ModificationSyncServiceTool](modificationsyncservice.md#modificationsyncservicetool)

## ModificationSyncService

The `ModificationSyncService` is a REST service that can be used to synchronize the access to shared resources from multiple applications. Such resources may be a central database or interfaces to external systems, for example. Applications can request a lock to a certain resource and if the resource is not locked already the request will be granted and recorded. Otherwise the application will be notified that a lock exists so that it can react accordingly, e.g. by notifying the user or postpone the processing of the resource. The business framework of SDACK can use a sync service to synchronize requests to persistent entities in a database.

The service is part of the _esoco-lib_ library of SDACK. It is implemented in Java as a server application that listens for HTTP requests on a certain port \(default 7962 = SYNC\). This makes it simple to run it as a Microservice, e.g. in a Docker container. To connect to a sync service from an application _esoco-lib_ also contains the `ModificationSyncEndpoint` that provides a simple `Endpoint` interface to sync services. For the administration of running sync services the command line tool `ModificationSyncServiceTool` can be used.

### Service Structure

The data held by the sync service is a mapping from resources to lock holders. If a client application wants to register a lock for some target resource it must provide identifiers for the target and for itself. These identifiers are string values and their structure, content, and global uniqueness must be defined by the applications. For example, in the case of database records a typical target ID could be a combination of table name and unique record ID.

If no lock is already held by another client the mapping is entered and the lock is granted. Otherwise the HTTP error status code 423 \(= _Locked_\) is returned. If the client no longer needs access to the resource it must release the lock for the respective target ID. Releasing locks is crucial for the correct functioning of the service and must therefore be implemented thoroughly to prevent the lock service from holding stale locks.

To avoid the need to operate multiple sync services for different deployment stages of the applications the sync service expects an additional context parameter besides the target and client IDs. It defines the context for which a lock is managed \(e.g. develop, test, and production\), allowing to deploy a single service to handle all sync requests in an organization. But if desired it is still possible to deploy multiple sync services and just use a default context. If all contexts are handled in a single instance it must be ensured that especially the applications in development or under test don't compromise the sync service \(e.g. by flooding it with invalid locks\). In the case of erratic application behavior the [ModificationSyncServiceTool](modificationsyncservice.md#modificationsyncservicetool) can be used to remove stale locks.

### Service API

The API of the sync service can be accessed through several URLs relative to the service address. Requests to these URLs must be in JSON format which is also used for the responses. The main URL is `/api/sync/` under which the actual synchronization API is available to applications. There are also some additional URLs that can be accessed:

* `/info`: displays some basic informations about the service \(this is also the default if the service address is invoked without a URL
* `/ping` and `/healthcheck`: both return true if the service is up and running. Can be used for monitoring purposes
* `/api/status` provides serveral endpoints to query the service status:
  * `/api/status/uptime`: the uptime of the service in milliseconds
  * `/api/status/start_date`: the start date and time of the service in the standard ISO date-time format
  * `/api/status/current_locks`: the currently held locks, mirrored from the sync API
* `/api/control` allows to query **and modify** the service:
  * `/api/control/run` always returns true; settings this to false will terminate the service immediately
  * `/api/control/log_level` returns the log level of the service; it can set set to either `TRACE, DEBUG, INFO, WARN, ERROR,` or `FATAL` to set the logging level of the service

The main sync service API under `/api/sync` provides the following endpoints:

* `/api/sync/check_lock`: check whether a certain lock already exists
* `/api/sync/request_lock`: register a new lock for the current client if it not already exists
* `/api/sync/release_lock`: release a lock held by the current client
* `/api/sync/current_locks`: list the locks currently held by the service \(mirrored as `/api/status/current_locks`\)

In addition, the API is mirrored under the base URL `/webapi` which provides a simple HTML view of the service API. This provides a way to quickly check the service status from a web browser. Service modifications are not possible through the web service. For that purpose the [ModificationSyncServiceTool](modificationsyncservice.md#modificationsyncservicetool) \(or the regular JSON API\) should be used.

## ModificationSyncEndpoint

The library also contains an implementation of the `Endpoint` class that provides an easy way to communicate with a running ModificationSyncService. Examples for using this endpoint can be found in the ModificationSyncServiceTool.

## ModificationSyncServiceTool

This is a Java command line application that invokes the API of running sync services to perform maintenance operations on it. To use this tool the corresponding classes from _esoco-lib_ and it's dependencies need to be available. The simplest way to obtain a JAR file with this classes is by cloning the project and invoking the corresponding Gradle task:

```bash
> git clone https://github.com/esoco/esoco-lib
> cd esoco-lib
> ./gradlew build mss
```

Afterwards the file mss.jar will be available in the build/libs folder. With this the sync service tool can then be invoked like any java application:

`java -jar mss.jar`

Because the tool needs some parameters to run it will only display a text with usage information. To display additional help about the usage the parameter -h \(or --help\) can be used:

```bash
>java -jar mss.jar --help
Usage: mss [OPTION]...
Sends a command to a ModificationSyncService running at an URL that must be set with -url

        -h         Display this help or informations about a certain command
        --help     Display this help or informations about a certain command
        -url       The URL of the sync service (mandatory)
        -context   The context to which to apply a command
        -target    The target to which to apply a command (in a certain context)
        -list      Lists either all locks or only in a given context
        -lock      Locks a target in a context
        -unlock    Removes a target lock in a context
        -reset     Resets either all locks or only in the given context
        -loglevel  Queries or updates the log level of the target service
```

To invoke the tool at least the service URL and a command must be provided:

`java -jar mss.jar -url http://sync-service.example.com -list`

The command `-list` will list all locks in the sync service, grouped by contexts. This is the same data that will be displayed when navigating to the `/api/status/current_locks` URL of the service. If a context is provided with the `-context <context>` switch a command will only be applied to that context. The list command would then only list the locks in the given context.

The context and an additional -target parameter are required for the `-lock` and `-unlock` commands. These set or remove locks for the respective target IDs in a certain context. The `-reset` command will clear all locks in given context or, if omitted, in all contexts. **These lock-modifying operations should be handled with caution** as they can severely affect the lock synchronization in an infrastructure. **The ModificationSyncServiceTool is an administration tool** that is only intended to solve problems that occur because application perform erroneous lock handling.

The last command is `-loglevel` which sets the log-level of the sync service at runtime as described for the `/api/control/log_level` URL. Caution is advised for the TRACE and DEBUG log levels as they may cause the output of much data in short time, possibly filling available storage space of the sync service.

