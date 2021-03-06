# Schema Exporter for Confluent Cloud

[![Build](https://travis-ci.com/abraham-leal/ccloud-schema-exporter.svg?branch=master)]() [![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=abraham-leal_ccloud-schema-exporter&metric=alert_status)](https://sonarcloud.io/dashboard?id=abraham-leal_ccloud-schema-exporter)

A tool to export schemas from a Confluent Cloud Schema Registry to another.
This app supports three modes: `batchExport`, `sync`, and `getLocalCopy`.

- `batchExport` will do a one time migration between schema registries, then it will reset the destination registry to `READWRTIE` mode.
- `sync` will continuously sync newly registered schemas into the destination registry.
- `getLocalCopy` will fetch and write local copies of Schema Registry's Schemas.

If you are looking to migrate schemas between On-Premise and Confluent Cloud, check out 
[Confluent Replicator](https://docs.confluent.io/current/connect/kafka-connect-replicator/index.html).

The exporter expects the following variables to be set in the environment to make the necessary calls:
(In the case of `-getLocalCopy` and `-customDestination` it does not need `DST_*` variables)

- `SRC_SR_URL` : The URL for the source Schema Registry
- `SRC_API_KEY` : The API KEY to be used to make calls to the source Schema Registry
- `SRC_API_SECRET` : The API SECRET to be used to make calls to the source Schema Registry
- `DST_SR_URL` : The URL for the destination Schema Registry
- `DST_API_KEY` : The API KEY to be used to make calls to the destination Schema Registry
- `DST_API_SECRET` : The API SECRET to be used to make calls to the destination Schema Registry

It is also possible to define the credentials through command flags. If both are defined, the flags take precedence.

## Build
````
git clone https://github.com/abraham-leal/ccloud-schema-exporter
cd ccloud-schema-exporter
go build ./cmd/ccloud-schema-exporter/ccloud-schema-exporter.go 
````

## Docker Run
````
docker run \
  -e SRC_SR_URL=$SRC_SR_URL \
  -e SRC_API_KEY=$SRC_API_KEY \
  -e SRC_API_SECRET=$SRC_API_SECRET \
  -e DST_SR_URL=$DST_SR_URL \
  -e DST_API_KEY=$DST_API_KEY \
  -e DST_API_SECRET=$DST_API_SECRET \
  abrahamleal/ccloud-schema-exporter:latest

````

A sample docker-compose is also provided at the root of this directory.

The docker image handles `-sync -syncDeletes -syncHardDeletes -noPrompt` continuous sync. For a one time export, it is recommended to use a release binary.

If you'd like to pass custom flags, it is recommended to override the entry-point such as with `--entrypoint` with `/ccloud-schema-exporter` at the beginning of the override.

For Docker, the `latest` tag will build directly from master. The master branch of this project is kept non-breaking;
However, for stable images tag a release.

## Run
- `./ccloud-schema-exporter -batchExport` : Running the app with this flag will perform a batch export.
- `./ccloud-schema-exporter -sync` : Running the app with this flag will start a continuous sync 
between source and destination schema registries.
- `./ccloud-schema-exporter -getLocalCopy` : Running the app with this flag will get a snapshot of your Schema Registry
into local files with naming structure subjectName-version-id per schema. The default directory is 
{currentPath}/SchemaRegistryBackup/.

When multiple flags are applied, prevalence is `sync` -> `batchExport` -> `getLocalCopy`

NOTE: Given that the exporter cannot determine a per-subject compatibility rule, it is recommended to set the destination schema registry compatibility level to `NONE` on first sync and restore it to the source's level afterwards.

### Options

````
Usage of ./ccloud-schema-exporter:

  -allowList value
    	A comma delimited list of schema subjects to allow. It also accepts paths to a file containing a list of subjects.
  -batchExport
    	Perform a one-time export of all schemas
  -customDestination string
    	Name of the implementation to be used as a destination (same as mapping)
  -deleteAllFromDestination
    	Setting this will run a delete on all schemas written to the destination registry. No respect for allow/disallow lists.
  -dest-sr-key string
    	API KEY for the Destination Schema Registry Cluster
  -dest-sr-secret string
    	API SECRET for the Destination Schema Registry Cluster
  -dest-sr-url string
    	Url to the Destination Schema Registry Cluster
  -disallowList value
    	A comma delimited list of schema subjects to disallow. It also accepts paths to a file containing a list of subjects.
  -getLocalCopy
    	Perform a local back-up of all schemas in the source registry. Defaults to a folder (SchemaRegistryBackup) in the current path of the binaries.
  -getLocalCopyPath string
    	Optional custom path for local copy. This must be an existing directory structure.
  -lowerBound int
    	Lower SR ID space bound (default 100000)
  -noPrompt
    	Set this flag to avoid checks while running. Assure you have the destination SR to correct Mode and Compatibility.
  -scrapeInterval int
    	Amount of time ccloud-schema-exporter will delay between schema sync checks in seconds (default 60)
  -src-sr-key string
    	API KEY for the Source Schema Registry Cluster
  -src-sr-secret string
    	API SECRET for the Source Schema Registry Cluster
  -src-sr-url string
    	Url to the Source Schema Registry Cluster
  -sync
    	Sync schemas continuously
  -syncDeletes
    	Setting this will sync soft deletes from the source cluster to the destination
  -syncHardDeletes
    	Setting this will sync hard deletes from the source cluster to the destination
  -timeout int
    	Timeout, in seconds, to use for all REST calls with the Schema Registries (default 60)
  -upperBound int
    	Upper SR ID space bound (default 101000)
  -usage
    	Print the usage of this tool
  -version
    	Print the current version and exit

````

#### Example Usage 
````
export SRC_SR_URL=XXXX
export SRC_API_KEY=XXXX
export SRC_API_SECRET=XXXX
export DST_SR_URL=XXXX
export DST_API_KEY=XXXX
export DST_API_SECRET=XXXX
./ccloud-schema-exporter <-sync | -batchExport | -getLocalCopy>
````

#### Filtering the export

It is now possible to filter the subjects which are sync-ed in all modes (`<-sync | -batchExport | -getLocalCopy>`).
Setting `-allowList` or/and `-disallowList` flags will accept either a comma delimited string, or a file containing
comma delimited entries for subject names (keep in mind these subjects must have their postfixes such as `-value` or 
`-key` to match the topic schema).
These lists will be respected with all run modes.
If specifying a file, make sure it has an extension (such as `.txt`).
A subject specified in `-disallowList` and `-allowList` will be disallowed by default.

NOTE: Lists aren't respected with the utility `-deleteAllFromDestination`

#### A note on syncing hard deletions

Confluent Schema Registry does not provide a good way to discover the full space of IDs registered.
Due to this, we do inefficient discovery of schema IDs. 
This means we have to assume Schema Registry IDs in the source and destiny SRs are within a range. 
By default, this range is 100,000 to 101,000. (The default start and limit in Confluent Cloud)
This range is tunable by setting `-lowerBound` and `-upperBound` together with enabling hard deletion sync with `-syncHardDeletes`.

Keep in mind syncing hard deletes does have a performance penalty on the sync. 
Getting the latest snapshot of the state is an expensive operation.
You can expect about a 2-second increase in sync time.

#### Non-Interactive Run

`ccloud-schema-exporter` is meant to be ran in a non-interactive way. 
However, it does include some checks to assure things go smoothly in the replication flow.
You can disable these checks by setting the configuration `-noPrompt`.
By default, the docker image has this in its entry point.

There are two checks made:
- The destination schema registry is in `IMPORT` mode. This is a requirement, otherwise the replication won't work.
- The destination schema registry is in `NONE` global compatibility mode.
This is not a requirement, but suggested since per-subject compatibility rules cannot be determined per version.
Not setting this may result in some versions not being able to be registered since they do not adhere to the global compatibility mode.
(The default compatibility in Confluent Cloud is `BACKWARD`).

If you'd like more info on how to change the Schema Registry mode to enable non-interactive runs, see the [Schema Registry API Documentation](https://docs.confluent.io/current/schema-registry/develop/api.html#mode)

#### Custom Destinations / Extendability

`ccloud-schema-exporter` supports custom implementations of destinations.
If you'd like to leverage the already built back-end, all you have to do is an implementation the `CustomDestination` interface.
A copy of the interface definition is below for convenience:

````
type CustomDestination interface {
	// Perform any set-up behavior before start of sync/batch export
	SetUp() error
	// An implementation should handle the registration of a schema in the destination.
	// The SchemaRecord struct provides all details needed for registration.
	RegisterSchema(record SchemaRecord) error
	// An implementation should handle the deletion of a schema in the destination.
	// The SchemaRecord struct provides all details needed for deletion.
	DeleteSchema(record SchemaRecord) error
	// An implementation should be able to send exactly one map describing the state of the destination
	// This map should be minimal. Describing only the Subject and Versions that already exist.
	// We assume this operation to be best done asynchronously, hence the channel.
	GetDestinationState(channel chan <- map[string][]int64) error
	// Perform any tear-down behavior before stop of sync/batch export
	TearDown() error
}
````

Golang isn't candid on runtime lookup of implementations of interfaces, so in order to make this implementation to the tool you must register it.
To register your implementation, go into `cmd/ccloud-schema-exporter/ccloud-schema-exporter.go` and modify the following map:

````
var factory = map[string]client.CustomDestination{
	"sampleCustomDestination": client.NewSampleCustomDestination(),
	// Add here a mapping of name -> factory/empty struct for reference at runtime
	// See sample above for the built-in sample custom destination that is within the client package
}
````

You will see this map already has one entry, that is because `ccloud-schema-exporter` comes with a sample implementation of the interface under `cmd/internals/customDestination.go`, check it out!
Make sure to add your implementation to this map.

Once added, all you have to do is indicate you will want to run with a custom destination with the `-customDestination` flag.
The value of this flag must be the name you gave it in the factory mapping.

The following options are respected for custom destinations as well:

````
  -allowList value
    	A comma delimited list of schema subjects to allow. It also accepts paths to a file containing a list of subjects.
  -batchExport
    	Perform a one-time export of all schemas
  -disallowList value
    	A comma delimited list of schema subjects to disallow. It also accepts paths to a file containing a list of subjects.
  -scrapeInterval int
    	Amount of time ccloud-schema-exporter will delay between schema sync checks in seconds (default 60)
  -sync
    	Sync schemas continuously
  -syncDeletes
    	Setting this will sync soft deletes from the source cluster to the destination
````

#### Feature Requests / Issue Reporting

This repo tracks feature requests and issues through Github Issues.
If you'd like to see something fixed that wasn't caught by testing, or you'd like to see a new feature, please feel free
to file a Github issue in this repo, I'll review and answer at best effort.

Additionally, if you'd like to contribute a fix/feature, please feel free to open a PR for review.