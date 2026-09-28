# Meridian

Provincial agri-environmental monitoring and yield forecasting platform.

## Architecture

Meridian is a three-tier application:

1. **Collector** (`ingest/`) receives telemetry from the field station
   network and writes it to the reading store.
2. **Application server** (`services/`) is a Spring Boot application
   exposing the REST API and the scheduled batch jobs.
3. **Web client** (`dashboard/`) is the operator console.

The reading store is a PostgreSQL instance on `db01.meridian.internal`.

## Building

    ./gradlew build

Requires JDK 8 and PostgreSQL 9.4 client libraries.

## Running the nightly batch

The batch is scheduled by the application server's Quartz configuration.
See `services/src/main/resources/quartz.properties`.

## Contacts

- Platform: D. Ferreira (dferreira@)
- Application: A. Okonkwo (aokonkwo@)
- Agronomy model: R. Halvorsen (rhalvorsen@)


Syed Raamis
Unsur Mir 

---
*Last updated 2015-11-04.*
