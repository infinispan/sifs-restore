# SoftIndexFileStore Restore Tool
SoftIndexFileStore restore tool used to transfer data files to a new server or rebuild an index offline.
This will load up a given data and optional index directory to then restore the contents into a remote server.

## How to build

A simple `mvn clean install` should build the tool and make it usable to run.

## Example usage

The simplest way to run is the following:

`mvn exec:java -Dexec.args="cache-name ${ISPN-dir}/server/data"`

This will load up any SIFS data/index files in the given data directory for that cache and
upload those to a client that is configured via the `hotrod-client.properties` file which
is found in the `src/main/resources` dirctory of the project.


It may however be required to pass additional arguments if your prior configuration was overriding
the data and/or index path.

For example this xml set the paths to equivalent name.
```xml
<persistence>
    <file-store>
        <data path="data"/>
        <index path="index"/>
    </file-store>
</persistence>
```

For this case you must also include `-d <data-path>` and `-i <index-path>` to the previous command so
that the tool can find the appropriate location of the data.

## SIFS data

Only the data directory is required to be present to find data to load. If there is an index present it will
be used if possible, if not a new index is created in the index directory.
Note that it is possible to provide `-r` to the command line to only rebuild the index and not perform remote loading.

## Marshalling Concerns

The tool is purposely implemented in a way so that no external user classes or marshalling is required.
All data is transferred as opaque binary data.
The only requirement is the cache serialization configuration that generated the original SIFS data must
also be present in the remote server cache configuration so it can properly deserialize the data.

## Remote server restore

Once the SIFS data is indentified and indexed tool will start inserting all values in the cache into the remote server.
By default data is inserted from 16 parallel requests. This can be changed by providing the `p` argument, for example 
`-p 32` will increase the paralle requests to 32.

Data is inserted using `putIfAbsent` and thus if data already exists for the same key it is not changed, only "new"
data is written to the remote server.

## Progress updates

The restore tool will provide updatese every five seconds for both reindexing and the remote upload procedure.
It is possible to increase or decrease the frequency of these udpates by providing the `f` argument such as `-f 1`
to receive the updates every seconds.

Example SIFS index being rebuilt. Note the number next to pending is the number of data files remaining to be
indexed.

```
Starting cache from data directory.. this may take some time especially if store index was not shut down cleanly
May 22, 2025 7:25:04 AM org.jboss.threads.Version <clinit>
INFO: JBoss Threads version 3.6.1.Final
May 22, 2025 7:25:09 AM org.infinispan.commons.util.ProgressTracker$State run
INFO: ISPN000972: Task 'sifs-task', pending (34), last check had (0) pending, status is PROGRESSING
May 22, 2025 7:25:14 AM org.infinispan.commons.util.ProgressTracker$State run
INFO: ISPN000972: Task 'sifs-task', pending (30), last check had (34) pending, status is PROGRESSING
```

Example remote cache upload. Note the number to the left is number of entries written
and in parenthesis is the percent completion.

```
Starting remote upload using 16 parallel inserts.. this will take some time (Updates every 5 seconds)
completed 393733 (3.93733%)
completed 924742 (9.24742%)
completed 1467822 (14.678221%)
completed 1992292 (19.92292%)
```

## Help usage

Here is an example output of the help output which can be retrieved by not passing any arguments or `-h`

```
Usage: <main class> [-r] [-d=<dataDirName>] [-f=<updateFrequency>]
                    [-i=<indexDirName>] [-p=<parallelInserts>] CACHE DIR
      CACHE               The name of the cache - should match the name in the
                            data directory and remote server (must be defined
                            on remote server)
      DIR                 The directory where cache persistence is - does not
                            include cache name
  -d=<dataDirName>        Name of the store data directory inside the cache
                            named directory. Use this if your previous
                            configuration defined an explicit data path
  -f=<updateFrequency>    How often message should be printed showing index or
                            insert progress in seconds. Default is 5 seconds
  -i=<indexDirName>       Name of the store index directory inside the cache
                            named directory. Use this if your previous
                            configuration defined an explicit index path
  -p=<parallelInserts>    How many parallel remote puts can be inflight. Only
                            used if rebuildIndexOnly is not set.
  -r                      Whether the index should only rebuilt and skip
                            writing the contents to a remote server.
                            
  -z                      Trusts all server certificates
```
