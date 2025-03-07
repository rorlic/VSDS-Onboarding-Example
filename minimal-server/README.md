# Setting Up a Minimal LDES Server
> **UPDATED** this tutorial has been changed to use Postgres as a data store instead of a MongoDB (which is obsoleted as of [LDES Server version 3.x](https://github.com/Informatievlaanderen/VSDS-LDESServer4J/releases/tag/v3.0.0-alpha)). You can find the previous version [here](https://github.com/Informatievlaanderen/VSDS-Onboarding-Example/tree/v2.0.0/minimal-server).

This quick start guide will show you how to setup a minimal LDES server to accept linked data members.

Please see the [introduction](../README.md) for the example data set and pre-requisites, as well as an overview of all examples.

## Enter the LDES Server
The [LDES Server](https://informatievlaanderen.github.io/VSDS-LDESServer4J/) is one of the components built as part of a large project on creating a [data space for Flanders](https://www.vlaanderen.be/vlaamse-smart-data-space-portaal) (sorry, Dutch only) for Data Owners, Data Publishers and Data Clients. Its main purpose is to accept data set members, store them in external storage and allow to retrieve the data set as a LDES for replication as a whole or just a subset of the data set. However, the real benefit of LDES lies in its ability to allow Data Clients to keep in sync with changes that occur on the data set.

Data Clients are typically not only interested in the current state of a data set but also in the historical changes that occurred, e.g. in order to understand the evolution of air quality in a city you need the analyze the measurements of some sensors over time. The LDES Server expects the data that it receives to be a so called version object, that is, the state of something at some point in time. This allows you to retrieve the changes of the object state from initial state (first version) to the current state (latest version) and due to the ability to keep in sync, even the future state (future versions). Pretty cool! Don't you think?

## Where Do We Keep This Gem?
As each Data Publisher needs to publish its own data set, the LDES Server is offered as a Docker image that can be configured to get the job done in a custom way and according to the needs of the Data Publisher.

The Docker images are available on [Docker Hub](https://hub.docker.com/r/ldes/ldes-server). [Here](https://hub.docker.com/r/ldes/ldes-server/tags) you can find the stable releases. Notice that some tags end with `-SNAPSHOT`. These images can safely be used. The only difference with the ones without this label is that validity with the official specification did not yet occur. Functionality, they are exactly the same. Every two weeks (our sprint duration) a new image is created and labeled with [semantic versioning](https://semver.org/). You can expect backwards compatible images to keep the same major version number and have an increased minor version number. Very occasionally we create a patch release with a non-zero patch number.

> **Note** that our [Github repository](https://github.com/Informatievlaanderen/VSDS-LDESServer4J) contains many [more images](https://github.com/Informatievlaanderen/VSDS-LDESServer4J/pkgs/container/ldes-server) which are created of our development branch. Please use these only if really needed as they can and will contain some issues.

## Setup Up the Basics
As mentioned in the pre-requisites, you need some minimal knowledge on Docker Compose as we use it to run both the LDES Server and the required data storage system as a container. Currently we only support [Postgres](https://www.postgresql.org/) for storing the LDES members and other data. On [Docker Hub](https://hub.docker.com/_/postgres) you can found the [container images](https://hub.docker.com/_/postgres/tags). In the future we may add other databases.

If you look into the [Docker Compose file](./docker-compose.yml), you will see that we define one (private) network for our two services: the LDES Server and the Postgres database. This allows the LDES Server to refer to the Postgres database by its service name `ldes-postgresdb` but also requires us to map the internal port numbers (respectively 8080 and 5432) to external port numbers (respectively 9003 and 5432) in order to be able to reach the services (respectively `http://localhost:9003` and `postgresql://ldes-postgresdb:5432/minimal-server`). We'll see in a minute where the internal ports come from. Furthermore the compose file contains names for our containers, the images and tags to use as a definition (i.e. a cookie-cutter) for our containers, a dependency on the Postgres so the LDES Server is started after the database container, and a (read-only) volume mapping of our LDES Server configuration file into the container file-system to allow our LDES Server to use it.

The Docker Compose file also contains some environment variables. The `SPRING_DATASOURCE_URL` is a [JDBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity) connection string that ties the LDES Server to our Postgres database. Note that the format is `jdbc:<scheme>://<server>:<port>/<database>` where the `<scheme>` is always `postgres`, the `<server>` is the service name of our datbase system (i.e. `ldes-postgresdb`), the `<port>` is the default port number `5432`, and finally, `<database>` is the name of the database that will hold our LDES data, for which we chose `minimal-server`. We also specify the external base path of our LDES Server (`LDESSERVER_HOSTNAME`) so that the links within the LDES can be followed. Because of the port mapping we set our LDES Server to be available at http://localhost:9003.

> **Note** that we also used environment variables to pass in the database credentials (`POSTGRES_USER` and `POSTGRES_PWD`). For your convenience we added some 'default' values in a file named [.env](./.env) which is automatically used by the Docker command line tools. Of course, you can pass in these environment variables in other ways too. Please see the Docker documentation [here](https://docs.docker.com/compose/environment-variables/).

The LDES Server [configuration file](./config/application.yml) is very basic: we only need to set some logging options and disable (push-based) tracing as we do not have a trace receiver.

Ready for some action? Let's get our hands dirty!

## Systems Ready? 3, 2, 1 .. Ignition!
Well, enough theory! Let's get this thing going and see how we can actually feed the LDES Server with some data.

To run the commands below you need to use a [bash command shell](https://en.wikipedia.org/wiki/Bash_(Unix_shell)). This allows us to keep the commands the same across the main three operating systems: Linux, Windows and MacOS. But, if you cloned this repository locally you already have a bash shell! It comes with your git installation.

To launch the LDES Server and the Postgres containers:
```bash
clear
docker compose up -d --wait
```

> **Note** that we start the containers as deamons and then wait for the LDES server to be available by using the Docker Compose [health check attribute](https://docs.docker.com/compose/compose-file/05-services/#healthcheck).

## Defining Our First LDES
As soon as the systems are started and ready we need to tell our LDES Server what data set we want to store because the LDES Server can host more than one data set. To manage this the LDES Server offers an administration API. We will explore this API later but for now we simply send the [LDES definition](./definitions/occupancy.ttl) to this API so that we can store [some data](./data/member.ttl) in our data set.

To define the LDES:
```bash
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams" -d "@./definitions/occupancy.ttl"
```

This may look like a bit of hocus-pocus but it is truly very simple. The [LDES definition](./definitions/occupancy.ttl) is a [turtle file](https://www.w3.org/TR/turtle/) which is a way of serializing [RDF](https://www.w3.org/RDF/). The definition file starts with a few declarations allowing for some short-hand notation so we do not get lost in translation. The other lines are the interesting part. First we start by saying that we define a LDES (`</occupancy> a ldes:EventStream`) and that we want the server to accept members and fetching the LDES on a sub-path (`/occupancy`). Then, we tell the server to automatically create version objects of our state object identified by the member property `dcterms:isVersionOf` and can be distinguished by the member property `prov:generatedAtTime`. Essentially, these properties allow us to group members to find all versions of something respectively to order them to know which member precedes another member.

We can verify that the LDES is actually known to the server by requesting it by its endpoint. This endpoint depends on several things: the port mapping, the base path of the server and the data set sub-path we defined in the LDES definition. In this example it is `http://localhost:9003/occupancy`.

To check out our LDES:
```bash
curl "http://localhost:9003/occupancy"
```

## Storing Our First Member
Once we have defined our LDES we can finally send our first member to the endpoint we defined and the LDES Server can store it.

To ingest a member:
```bash
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/occupancy" -d "@./data/member.ttl"
```

> **Note** that the member is some linked data serialized as a turtle file for which the default file extension is `.ttl` and the mime type is `text/turtle`. The LDES server can accept a few more RDF serialization formats, each identified by their own mime type.

## Creating Our First View
Now that we have ingested a member, you may wonder how we can check that it is actually in there? And how will Data Clients retrieve our small data set. The LDES Server can offer various views on each data set so we need to tell it how we want the LDES to be available for consumption. We do this again using the administration API by sending a [view definition](./definitions/occupancy.by-page.ttl) to the LDES Server and attaching it to our LDES. The LDES Server will start a background process to create this LDES view and offer the data set in fragments. Such a fragment is a number of members as well as information about the LDES and ways to navigate to the other members from the data set. See the [Tree specification](https://treecg.github.io/specification/) on which the LDES specification is built for details. Essentially, the fragments create a (typically hierarchical but possibly a graph) structure for navigating between them by defining the relation between themselves and their related fragments. Data Clients can then navigate to all or a part of these relations and therefore reach all or a subset of the data set. But enough theory! Let's define the view and request the member.

To define the LDES view:
```bash
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams/occupancy/views" -d "@./definitions/occupancy.by-page.ttl"
```

The view definition is a turtle file very similar to the LDES definition. It contains the view definition (it is a tree node) and how to get to this view (`</occupancy/by-page> a tree:Node`). It set the number of members per fragment (`tree:pageSize "50"^^xsd:integer`). That's all folks!

To check out our LDES:
```bash
curl "http://localhost:9003/occupancy/by-page"
```

> **Note** that you can create more than one view of a LDES, even for simple pagination by specifying a different view URI and page size. Later when we learn about retention policies and different fragmentation strategies, this may make more sense. For now remember that you can create a view before or after you ingest data, You can delete views and re-create them with different options. For this, you will need to use the administration API. Later, we will show you how to enable the [swagger](https://swagger.io/) to explore this API.

## Show Me the Data!
Depending on the size of the data set the LDES Server magic may take a while to make all members available, but you can already get the first members almost immediately.

To retrieve the data set:
```bash
curl "http://localhost:9003/occupancy/by-page?pageNumber=1"
```

## All Good Things Must Come To an End
That's it. Now you have an understanding what the LDES Server is about, where you can find it and how to set it up. You have learned how to create a LDES and a view for it as well as how to ingest and retrieve your data set. It's time now to stop the LDES Server and its storage.

To bring the containers down and remove the private network:
```bash
docker compose down
```
