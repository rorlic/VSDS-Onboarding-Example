# Publishing a Protected LDES
> **UPDATED** this tutorial has been changed to use Postgres as a data store instead of a MongoDB (which is obsoleted as of [LDES Server version 3.x](https://github.com/Informatievlaanderen/VSDS-LDESServer4J/releases/tag/v3.0.0-alpha)). You can find the previous version [here](https://github.com/Informatievlaanderen/VSDS-Onboarding-Example/tree/v2.0.0/protected-setup).

This tutorial will show you how to protect a LDES in order to prevent unauthorized access to a proprietary (or a public) data collection. It will also show how to expose the available LDES Server API as well as add and expose some metadata (using DCAT). In addition it will show you how to access such a protected LDES.

Please see the [introduction](../README.md) for the example data set and pre-requisites, as well as an overview of all examples.

## I’ll Protect You From the Hooded Claw
The LDES Server allows you to ingest a data collection and offers one or more views which allows replicating the data collection in whole or a part of it. However, not all data collection can be made publicly available. You need to protect those data collection in some way to prevent unauthorized access. What you will typically do is configure some security system which expects a Data Client to identify itself (_authentication_) after which the security system verifies it the Data Client has access to the requested data (_authorization_).

The LDES Server does not include such a security system because there are various ways for doing authentication anad authorization and no one-size fits all. You may want to protect the data simply with an API key (secret key shared with a Data Client), using OAuth2 (authorization only) and/or OpenID (based on OAuth2 adding authentication), etc. There are many open-source and commercial implementations available for these and other protocols, so we decided to not include this aspect into the LDES Server. This keeps the server lean and simple for serving open data collections.

But, even if you have a open data collection you may want to add a security layer to it to keep usage statistics per Data Client, e.g. to enforce fair usage policies, etc. So, we will look into how you can protect a LDES using a simple API key per Data Client and how the Data Client can access such a LDES.

This technique may also apply to a commercial data collection: as a Data Publisher you may also want to offer a part of your data to the public for publicity and marketing reasons but of course you will want to track its usage.

We will show how you can protect a LDES and what changes are needed to retrieve such a LDES. We will use existing tutorials to kickstart our setup. To create and feed our LDES we can use the [advanced conversion setup](../advanced-conversion/README.md) and to retrieve it we can use the [minimal client](../minimal-client/README.md). Later we will add a reverse proxy to shield the LDES Server from the outside and configure things in such a way that we have controlled access to both LDES Server administration and LDES replication & synchronization. Obviously we will have to make changes on both the Data Publisher and the Data Client side.

But first, let us setup the system without access limitation first to ensure everything works fine. As usual, we start by creating a docker compose file containing the services that we need.

At the Data Publisher side we need a database for the LDES Server (`ldes-postgresdb`), the LDES Server itself (`ldes-server`) and a workbench to feed the LDES Server (`server-workbench`). We can basically copy/paste the services from the [advanced conversion docker compose](../advanced-conversion/docker-compose.yml) file:

```yaml
  ldes-postgresdb:
    image: postgres:latest
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PWD}
    ports:
      - 5432:5432
    networks:
      - protected-setup-server


  ldes-server:
    image: ldes/ldes-server:3.6.1
    ports:
      - 9003:80
    networks:
      - protected-setup-server
    depends_on:
      - ldes-postgresdb
    environment:
      - SERVER_PORT=80
      - LDESSERVER_HOSTNAME=http://host.docker.internal:9003
      - SIS_DATA=/tmp
      - SERVER_SERVLET_CONTEXTPATH=
      - SPRING_DATASOURCE_URL=jdbc:postgresql://ldes-postgresdb:5432/${POSTGRES_DB}
      - SPRING_DATASOURCE_USERNAME=${POSTGRES_USER}
      - SPRING_DATASOURCE_PASSWORD=${POSTGRES_PWD}
      - SPRING_BATCH_JDBC_INITIALIZESCHEMA=always
      - MANAGEMENT_TRACING_ENABLED=false
      - SPRING_TASK_SCHEDULING_POOL_SIZE=5
      - LDESSERVER_FRAGMENTATIONCRON=*/1 * * * * *
      - LDESSERVER_MAINTENANCECRON=-  # no retention / compaction / deletion
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://ldes-server/actuator/health"]
      interval: 12s
      timeout: 3s
      retries: 20


  server-workbench:
    image: ldes/ldi-orchestrator:2.12.0
    environment:
      - SERVER_PORT=80
    ports:
      - 9004:80
    networks:
      - protected-setup-server 
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://server-workbench/actuator/health"]
    depends_on:
      - ldes-server
```

> **Notes**:
> * for clarity we renamed the network as well as removed the container names
> * we also renamed the workbench in order to stress that this is the workbench which feeds the LDES Server
> * we moved the configuration files to organize the setup a bit
> * we changed the environment variable `LDESSERVER_HOSTNAME` to access the LDES Server from the client network

At the Data Client side we only need a workbench (`client-workbench`) which we can borrow from the [minimal client docker compose](../minimal-client/docker-compose.yml) file and a sink system (for which we can use a [Test Message Sink](https://github.com/Informatievlaanderen/VSDS-LDES-E2E-message-sink)):

```yaml
  client-workbench:
    image: ldes/ldi-orchestrator:2.12.0
    environment:
      - SERVER_PORT=80
    networks:
      - protected-setup-client
    ports:
      - 9006:80
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - sink-system
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://client-workbench/actuator/health"]


  sink-system:
    image: ghcr.io/informatievlaanderen/test-message-sink:latest
    ports:
      - 9007:80
    networks:
      - protected-setup-client
    environment:
      - MEMORY=true
      - MEMBER_TYPE=http://schema.mobivoc.org/#ParkingLot
```

> **Note** that the client workbench uses its own network which is completely disconnected from the network used by the LDES Server and its database and workbench. The LDES Client component therefore needs to use docker host name (`host.docker.internal`) and exposed server port (`9003`) to access the LDES.

What we have now is illustrated in the following system container diagram:
![container diagram](./uml/unprotected-container.png)

Fig. 1 - Unprotected setup

At this point we can run all the systems and verify that we receive the LDES members in the sink. Start all systems using (in a bash shell):
```bash
clear

# start and wait for all systems to be available
docker compose up -d --wait

# define the LDES and the view
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams" -d "@./definitions/occupancy.ttl"
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams/occupancy/views" -d "@./definitions/occupancy.by-page.ttl"

# start the publish and client pipelines
curl -X POST -H "content-type: application/yaml" "http://localhost:9004/admin/api/v1/pipeline" --data-binary "@./definitions/publish-pipeline.yml"
curl -X POST -H "content-type: application/yaml" "http://localhost:9006/admin/api/v1/pipeline" --data-binary "@./definitions/client-pipeline.yml"

# check the sink system
while true; do curl "http://localhost:9007"; echo ""; sleep 15; done
```
Once you have verified that the members appear in the sink (this may take a couple of minutes) you can press `CTRL-C` to stop following the member count in the sink system. After that you can shutdown the systems and remove the private network for now using:
```bash
docker compose down
```

## It’s a Well Kept Secret
Now that we have a unprotected but working setup we can make the necessary changes to enforce security. We will need to do a few things:
* add a reverse proxy that will accept the LDES requests on the server's behalf, check authentication & authorization and forward the request or return an access error
* not expose the LDES Server outside of the internal docker network so that the only way to access it is through the reverse proxy
* change the Data Client (and Data Publisher) configuration to retrieve the LDES through the reverse proxy

The following illustration shows the setup after adding such a reverse proxy:
![container diagram](./uml/protected-container.png)

Fig. 2 - Protected setup

The first thing we need to asks ourselves is which endpoints we need to protect but before we can answer that question we need to know what endpoints are available. By default, the LDES Server does not make that immediately apparant but by exposing the so called [Swagger UI](https://swagger.io) we can make the available API visible. If we add the following environment variables (in an additional [docker compose](./expose-swagger.yml) file and [merge](https://docs.docker.com/compose/compose-file/13-merge/) them), we should see it (after launching the LDES Server):
```yaml
services:
  ldes-server:
    environment:
    - SPRINGDOC_SWAGGERUI_PATH=/admin/doc/v1/swagger
    - SPRINGDOC_SWAGGERUI_URLSPRIMARYNAME=base
```

> **Note** that we expose our swagger UI on the 'admin' API so we can protect it together with the rest of the admin API later.

The `path` allows us to define the endpoint where the API information is visualized and the `urlsPrimaryName` allows us to choose which collection of APIs are displayed by default when we browse to the swagger UI endpoint. Now, in order to see it, we need to launch the server again:

```bash
clear
docker compose -f docker-compose.yml -f expose-swagger.yml up -d --wait
```

Once started point your browser to http://localhost:9003/admin/doc/v1/swagger. You will be redirected to http://localhost:9003/admin/doc/v1/swagger-ui/index.html and in the top right corner you should see the `base` API collection selected and the base API displayed in the main window.

When we look at this base API, we see that there is one endpoint for the ingest that expects a `POST` to an endpoint. Obviously we do not want anybody else than the server workbench to push members to our ingest endpoint so we need to disallow this through the reverse proxy. We can do that by disallowing `POST` requests through the reverse proxy but we need to ensure that we can still seed the LDES definitions, which also use `POST` requests towards the administrative API.

Next we see that the LDES Server exposes an endpoint to retrieve metadata at the root `/` as well as the various data collections at `/{collection-name}` and their respective views at  `/{collection-name}/{view-name}`, all using `GET` requests. Most likely we want the metadata to be publicly accessible so that our available data collections can be discovered. In other words, we want metadata crawlers to be able to retrieve the metadata (typically [DCAT](https://en.wikipedia.org/wiki/Data_Catalog_Vocabulary) information) in an unsecured way so that we get some exposure for our data collections. Now, as for the collections and views themselves, we can setup the accessibility as required by our use cases. For this tutorial we will assume that only a few clients can access our LDES and view so we will protect them with an API key. In fact, we will assign one API key per client so we can distinguish them for statistical reasons (e.g. to enforce a fair use policy).

Now, if we switch to the admin API (select `admin` in the top right dropdown) we see a lot more available endpoints for managing the LDES Server. Obviously, we want to protect the whole admin API (including the swagger UI) so that only we are allowed to manage it. We will do this by protecting it with yet another API key.

> **Note** that an API key is not a very secure way of protecting an API. If somebody gets a hold of it, it can be misused. Therefore you should keep it a secret and use HTTPS instead of HTTP communication to prevent somebody sniffing the network and gaining access to the API key.

Currently we do not expose any metadata for our LDES. Without going into details of DCAT, we will simply add the metadata ([catalog](./definitions/metadata/catalog.ttl), [LDES metadata](./definitions/metadata/occupancy.ttl) and [view metadata](./definitions/metadata/occupancy.by-page.ttl)) to our LDES Server by means of the admin API. We have kept the DCAT itself to the bare minimum as that is beyond the scope of the tutorial. It will be sufficient for our purpose. To try this please run:

```bash
# upload LDES & view definitions
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams" -d "@./definitions/occupancy.ttl"
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams/occupancy/views" -d "@./definitions/occupancy.by-page.ttl"

# upload metadata definitions
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/dcat" -d "@./definitions/metadata/catalog.ttl"
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams/occupancy/dcat" -d "@./definitions/metadata/occupancy.ttl"
curl -X POST -H "content-type: text/turtle" "http://localhost:9003/admin/api/v1/eventstreams/occupancy/views/by-page/dcat" -d "@./definitions/metadata/occupancy.by-page.ttl"
```

Now you can get the full DCAT if you request the root http://localhost:9003. It is a mix of the metadata definitions which we uploaded and server generated data, resulting in something like this:
```text
PREFIX dcat: <http://www.w3.org/ns/dcat#>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX geo:  <http://www.opengis.net/ont/geosparql#>
PREFIX ldes: <https://w3id.org/ldes#>
PREFIX m8g:  <http://data.europa.eu/m8g/>
PREFIX owl:  <http://www.w3.org/2002/07/owl#>
PREFIX prov: <http://www.w3.org/ns/prov#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX sh:   <http://www.w3.org/ns/shacl#>
PREFIX shsh: <http://www.w3.org/ns/shacl-shacl#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX tree: <https://w3id.org/tree#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>

<http://host.docker.internal:9003/occupancy>
        rdf:type         dcat:Dataset;
        dct:conformsTo   <https://w3id.org/tree/specification> , <https://w3id.org/ldes/specification>;
        dct:description  "LDES containing the occupancy of the various park+rides in Ghent in real time"@en;
        dct:identifier   "http://host.docker.internal:9003/occupancy"^^rdfs:Literal;
        dct:title        "Real time occupancy P+R (Gent) as LDES"@en .

<http://host.docker.internal:9003>
        rdf:type         dcat:Catalog;
        dct:description  "Offers an overview of the dataset(s) and data service(s) needed for the tutorial 'Publishing And Accessing a Protected LDES'."@en;
        dct:identifier   "e9dd428a-0a81-4cdb-ba2f-a2d114982a27"^^rdfs:Literal;
        dct:title        "Catalog for Publishing And Accessing a Protected LDES"@en;
        dcat:dataset     <http://host.docker.internal:9003/occupancy>;
        dcat:service     <http://host.docker.internal:9003/occupancy/by-page/description> .

<https://semiceu.github.io/LinkedDataEventStreams/>
        rdf:type  rdfs:Resource .

<https://w3id.org/tree/specification>
        rdf:type  dct:Standard .

<https://w3id.org/ldes/specification>
        rdf:type  dct:Standard .

<http://host.docker.internal:9003/occupancy/by-page>
        rdf:type  rdfs:Resource .

<http://host.docker.internal:9003/occupancy/by-page/description>
        rdf:type                  dcat:DataService;
        dct:description           "Paged view for the occupancy of the various park+rides in Ghent in real time"@en;
        dct:identifier            "http://host.docker.internal:9003/occupancy/by-page"^^rdfs:Literal;
        dct:title                 "Real time occupancy P+R (Gent) by page"@en;
        dcat:endpointDescription  <https://semiceu.github.io/LinkedDataEventStreams/>;
        dcat:endpointURL          <http://host.docker.internal:9003/occupancy/by-page>;
        dcat:servesDataset        <http://host.docker.internal:9003/occupancy> .
```

As said before, we want this metadata to be publicly available, while limiting access to the admin API only to ourselves and the LDES & the view to a couple of well-known clients, all by means of a unique API key. You can create these keys using one of the free online GUID generators (e.g. https://www.uuidgenerator.net/guid) or a password generator (e.g. https://www.avast.com/random-password-generator), etc.

### Add a Reverse Proxy
First we need to create the reverse proxy service in the additional [docker compose](./introduce-proxy.yml) file (we also included the environment variables to expose the swagger):
```yaml
services:
  reverse-proxy:
    image: nginx:stable
    ports:
      - 9005:8080
    volumes:
      - ./reverse-proxy/protect-ldes-server.conf:/etc/nginx/conf.d/protect-ldes-server.conf:ro
    depends_on:
      - ldes-server
    networks:
      - protected-setup-server 

```
Here we chose a well-known freely available component that we can setup as a reverse proxy. There are many options out there open-source and commercial. The configuration is highly dependent on the component that you use but the principles are mostly the same: you allow or disallow access to some URL for some HTTP verbs (GET, HEAD, POST, etc) based on some conditions.

We'll be using these three API keys for demonstration purposes:

|API key|Purpose|
|-|-|
|admin-secret|admin|
|client-one-secret|client 1|
|client-two-secret|client 2|

This translates to a mapping for our reverse proxy. Next we define an API key validation location, meaning that for the URLs we need to protect we use this internal virtual URL for verifying the presence and validity of the given API key. Finally we add our access rules. As previously said we need to:
* allow access to metadata to the public
* allow access to admin API (including the swagger API) only to administrators
* allow access to the LDES, the view and all view nodes for registered clients only
* do not allow access to the ingest endpoint (no POST to LDES endpoint allowed from outside)

For this particular reverse proxy we end up with this [configuration](./reverse-proxy/protect-ldes-server.conf) and can start the reverse proxy to test if the rules allow or disallow access correctly:
```bash
docker compose -f docker-compose.yml -f introduce-proxy.yml up -d --wait
```

We have setup the reverse proxy to remap the LDES Server endpoints (all based at `/ldes`) a bit. The reverse proxy serves:
* the metadata at `/ldes`
* the admin API at `/admin`
* the LDES, the view and its nodes at `/feed`

If we do not pass an API key we can retrieve only the metadata and not the LDES, the view, the admin API and the swagger UI:
```bash
clear
curl -I "http://localhost:9005/ldes"
curl -I "http://localhost:9005/feed/occupancy"
curl -I "http://localhost:9005/feed/occupancy/by-page"
curl -I "http://localhost:9005/admin/api/v1/eventstreams"
```
Public access is only allowed (HTTP 200) for the first call, all other calls are unauthenticated (HTTP 401).
> **Note** that we pass `-I` in order to only retrieve the headers, not the actual content. 

If we pass a client API key we can retrieve the metadata, the LDES and the view but we cannot use the admin API:
```bash
clear
curl -I -H "x-api-key: client-one-secret" "http://localhost:9005/ldes"
curl -I -H "x-api-key: client-one-secret" "http://localhost:9005/feed/occupancy"
curl -I -H "x-api-key: client-one-secret" "http://localhost:9005/feed/occupancy/by-page"
curl -I -H "x-api-key: client-one-secret" "http://localhost:9005/admin/api/v1/eventstreams"
```
All but the last call should succeed (HTTP 200) while the last one is forbidden (HTTP 403) because we are authenticated but not authorized to use the admin API.
> **Note** that we need to pass the API key using the header `x-api-key: <well-known-key>`.

Finally, if we pass the admin API key all calls should be possible:
```bash
clear
curl -I -H "x-api-key: admin-secret" "http://localhost:9005/ldes"
curl -I -H "x-api-key: admin-secret" "http://localhost:9005/feed/occupancy"
curl -I -H "x-api-key: admin-secret" "http://localhost:9005/feed/occupancy/by-page"
curl -I -H "x-api-key: admin-secret" "http://localhost:9005/admin/api/v1/eventstreams"
```
Now all calls succeed. Great!

We need to verify one more rule: nobody (not even an administrator!) should we able to send data to the ingest endpoint of the LDES server:
```bash
clear
curl -X POST -i -H "content-type: text/turtle" -d "@./data/member.ttl" "http://localhost:9005/feed/occupancy"
curl -X POST -i -H "content-type: text/turtle" -d "@./data/member.ttl" "http://localhost:9005/feed/occupancy" -H "x-api-key: client-one-secret"
curl -X POST -i -H "content-type: text/turtle" -d "@./data/member.ttl" "http://localhost:9005/feed/occupancy" -H "x-api-key: admin-secret"
```
All calls should fail with a forbidden (HTTP 403).

One final thing to test is if we can POST to the admin endpoint, e.g. to define a LDES or add metadata.
```bash
curl -X POST -d "@./definitions/metadata/catalog.ttl" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/dcat" -H "x-api-key: admin-secret"
```
Should return an error similar to `Resource of type: dcat-catalog with id: c403cbbd-9e4d-47a2-8bb5-41a7642701ba already exists.` because we have already defined the catalog. But, this is great news because this means the reverse proxy let the request go through and the LDES Server returns an error response.

### Do Not Expose the LDES Server
Now that everything is working great we can remove the port mapping of the server in the additional [docker compose](./protect-server.yml) file (also exposing the swagger and introducing the reverse proxy) as we do not need and do not want any direct access to it:
```yaml
services:
  ldes-server:
    ports: !reset [] # do not expose ports
```

### Access the LDES Server Through the Reverse Proxy
Once the LDES Server is not directly accessible anymore, we need to change the `LDESSERVER_HOSTNAME` environment variable to `http://host.docker.internal:9005/feed` to use the reverse proxy instead:
```yaml
services:
  ldes-server:
    environment:
    - LDESSERVER_HOSTNAME=http://host.docker.internal:9005/feed
```
> **Note** that our complete [override docker compose](./protect-server.yml) file now looks like:
```yaml
services:

  ldes-server:
    ports: !reset [] # do not expose ports
    environment:
    - LDESSERVER_HOSTNAME=http://host.docker.internal:9005/feed
    - SPRINGDOC_SWAGGERUI_PATH=/admin/doc/v1/swagger
    - SPRINGDOC_SWAGGERUI_URLSPRIMARYNAME=base

  reverse-proxy:
    image: nginx:stable
    ports:
      - 9005:8080
    volumes:
      - ./reverse-proxy/protect-ldes-server.conf:/etc/nginx/conf.d/protect-ldes-server.conf:ro
    depends_on:
      - ldes-server
    networks:
      - protected-setup-server 
```

Finally, we also need to change the LDES Client configuration to use a new URL (`http://host.docker.internal:9005/feed/occupancy`) and should not forget the most important part, configure the LDES Client to pass a API key when requesting the LDES nodes:
```yaml
  name: Ldio:LdesClient
  config:
    urls: 
      - http://host.docker.internal:9005/feed/occupancy
    auth:
      type: API_KEY
      api-key-header: x-api-key
      api-key: client-two-secret
    ...
```

Show time! But first bring down all systems so we can start with a clean slate:
```bash
docker compose -f docker-compose.yml -f introduce-proxy.yml down
```

## Putting It All Together
To launch all the systems and configure it all you can run the following:
```bash
clear

# start and wait for the server and database systems
docker compose -f docker-compose.yml -f protect-server.yml up -d --wait

# define the LDES and the view
curl -X POST -H "x-api-key: admin-secret" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/eventstreams" -d "@./definitions/occupancy.ttl"
curl -X POST -H "x-api-key: admin-secret" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/eventstreams/occupancy/views" -d "@./definitions/occupancy.by-page.ttl"

# upload metadata definitions
curl -X POST -H "x-api-key: admin-secret" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/dcat" -d "@./definitions/metadata/catalog.ttl"
curl -X POST -H "x-api-key: admin-secret" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/eventstreams/occupancy/dcat" -d "@./definitions/metadata/occupancy.ttl"
curl -X POST -H "x-api-key: admin-secret" -H "content-type: text/turtle" "http://localhost:9005/admin/api/v1/eventstreams/occupancy/views/by-page/dcat" -d "@./definitions/metadata/occupancy.by-page.ttl"

# start the publish and client pipelines
curl -X POST -H "content-type: application/yaml" "http://localhost:9004/admin/api/v1/pipeline" --data-binary "@./definitions/publish-pipeline.yml"
curl -X POST -H "content-type: application/yaml" "http://localhost:9006/admin/api/v1/pipeline" --data-binary "@./definitions/protected-client-pipeline.yml"

# check the sink system
while true; do curl "http://localhost:9007"; echo ""; sleep 15; done
```
> **Note** that we:
> * use the admin endpoint (`http://localhost:9005/admin`) for sending LDES and metadata definitions,
> * provide our admin x-api-key for these requests, and
> * use a different pipeline (`protected-client-pipeline`).

It all goes well (and it should!) you will see the LDES members appear in the sink.

Press `CTRL-C` to stop following the member count in the sink system.

## It's Been a Long Day
We have shown you how to enable the swagger UI, how to provide metadata for your LDES views and, of course, how to access a protected LDES. In addition we have shown you how you can protect a LDES using a API key but if you require a stronger way of securing access have a look at other authentication and authorization mechanisms. The [documentation](https://informatievlaanderen.github.io/VSDS-Linked-Data-Interactions/ldio/ldio-core/ldio-http-requester) explains how to configure the LDES client in case you need to access an OAuth2/OpenID protected LDES.

Now that you have verified that the members appear in the sink you can shutdown the systems and remove the private network using:
```bash
docker compose -f docker-compose.yml -f protect-server.yml down
```
