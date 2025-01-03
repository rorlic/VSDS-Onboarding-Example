# Setting Up a Minimal LDIO Workbench
> **UPDATED** this tutorial has been changed to define the pipeline dynamically and to process regular (state) objects instead of creating (historical) version objects. You can find the previous version [here](https://github.com/Informatievlaanderen/VSDS-Onboarding-Example/tree/v1.0.0/minimal-workbench).

This quick start guide will show you how to setup a minimal workbench to create linked data state objects.

Please see the [introduction](../README.md) for the example data set and pre-requisites, as well as an overview of all examples.

## Show Me Your Workbench
The LDES Server allows to ingest linked data but typically the source data represents the state of an object in a non-linked data way. That is why a data transformation is needed.

Such a data transformation can be standalone or as part of a data transformation pipeline which can be build in various ways with many different data processing systems.

One such a workbench that allows to create data pipelines is [Apache NiFi](https://nifi.apache.org/). This is a mature and solid open-source solution that comes with many features, allows for horizontal scaling and comes with many standard processors for creating and monitoring complex data pipelines. However, it also comes with a steep learning curve and some other drawbacks. 

The Linked Data Interactions Orchestrator ([LDIO](https://informatievlaanderen.github.io/VSDS-Linked-Data-Interactions/)) is a simple and more light-weight solution that eases the process of creating more straightforward, linear data transformations while requiring minimal resources and attempting to keep the learning curve as low as possible. It is by no means a silver bullet but experience has learned us that most data publishing use cases can easy be covered with a simple linear pipeline and as such LDIO usually suffies.

LDIO allows to create one or more synchronous linear pipelines that convert non-linked data to linked data (state or version objects) that can be ingested by an LDES Server. It is centered around the concept of one input source with an adaptor to convert to linked data, one or more in-memory transformation steps and sending the result to one or more output sinks.

Various input components are available for starting a pipeline such as: accepting HTTP messages both using a push model (HTTP listener) and a pull model (HTTP poller), reading from Kafka, etc.

If the source data is already linked data you can use a simple RDF adaptor which allows to parse various RDF serializations. If the source data is not yet linked data you can use either a JSON-LD adaptor to attach a JSON-LD context to a JSON message or alternatively a RML adaptor, which allows to create linked data from various other message formats, such as JSON, XML, CSV, etc.

On the output side we also provide several possibilities such as POST-ing using HTTP, writing to Kafka, etc.

We provide several transformation components for manipulating linked data but most data transformations can be done using a SPARQL construct transformation step. In addition, we also provide some components for more specific tasks such as enriching the data from some external HTTP source, converting GeoJson to Well Known Text (WKT), etc.

All these components are provided as part of the LDIO workbench which is packaged as a Docker image. The Docker images are available on [Docker Hub](https://hub.docker.com/r/ldes/ldi-orchestrator). The stable releases can be found [here](https://hub.docker.com/r/ldes/ldi-orchestrator/tags).

## Configure Your First Pipeline
The example [docker compose file](./docker-compose.yml) only contains a LDIO service which runs in a private network and uses volume mapping to have its configuration file available in the container. As we will see in a minute, the pipeline starts with a HTTP listener and therefore we need a port mapping to allow the workbench to receive HTTP messages.

> **Note** that the workbench can contain more than one pipeline if needed. We simply need to define our pipelines with a different name using lowercase or uppercase letters, digits, blanks and the special characters `_`, `-`  & `.`.

The [pipeline definition](./definitions/pipeline.yml) starts with a name and a description. The latter is purely for documentation purposes, but the former is used as the base path on which the HTTP listener receives requests. In our case this is (based on the docker compose port mapping): http://localhost:9004/park-n-ride-pipeline. After that the definition continues with the input component and associated adapter, the (optional) transformation steps and the output(s). Let's look at these in more detail.

The input component simply states that it is a HTTP listener which uses a RDF adaptor and as such is expecting Linked Data:
```yaml
input:
  name: Ldio:HttpIn
  adapter:
    name: Ldio:RdfAdapter
```

We output the linked data (state) objects to the specified sink(s). For demo purposes we use a component that simply logs the member to the console, which for a Docker container results in its logs.
```yaml
outputs:
  - name: Ldio:ConsoleOut 
```

## Launch the Magic
After this long introduction let's get our hands dirty and see the magic in action.

To start the workbench and wait until it is available:
```bash
clear
docker compose up -d --wait
```

There is no visual component yet for the LDIO workbench, but you can check its status at http://localhost:9004/actuator/health.

The workbench is now ready to receive our pipeline. We can use the LDIO Workbench admin API to tell it to use our pipeline:
```bash
curl -X POST -H "content-type: application/yaml" http://localhost:9004/admin/api/v1/pipeline --data-binary @./definitions/park-n-ride-pipeline.yml
```

You can check that the pipeline is actually running:
```bash
curl http://localhost:9004/admin/api/v1/pipeline/status
```
which returns:
```json
{"park-n-ride-pipeline":"RUNNING"}
```

## You've Got Mail
Now that the workbench and the pipeline is up and running we can send a [message](./data/message.jsonld) through the pipeline and see its version object outputted to the workbench logs. We use the following simple JSON-LD message (clipped to the relevant parts):
```json
{
    "@context": {
        "@vocab": "https://example.org/ns/mobility#",
        "urllinkaddress": "@id",
        "type": "@type",
        "lastupdate": {
            "@type": "http://www.w3.org/2001/XMLSchema#dateTime"
        }
    },
    "lastupdate": "2023-11-30T21:45:15+01:00",
    "type": "offStreetParkingGround",
    "urllinkaddress": "https://stad.gent/nl/mobiliteit-openbare-werken/parkeren/park-and-ride-pr/pr-gentbrugge-arsenaal",
    ...
}
``` 

> **Note**: do not worry if you do not understand every detail in the context part. Its main purpose is to add meaning to the state object properties.

To send the message into the pipeline:
```bash
curl -X POST -H "Content-Type: application/ld+json" "http://localhost:9004/park-n-ride-pipeline" -d "@./data/message.jsonld"
```

Since it is a small and straight forward message the workbench log will almost immediately contain the version object. 

To watch the version object appear in the workbench log
```bash
docker logs -n 24 $(docker compose ps -q ldio-workbench)
```

You should see the following:
```text
2024-04-10T10:03:17.426Z  INFO 1 --- [nio-8080-exec-6] b.v.i.ldes.ldio.LdiConsoleOut            : @prefix :         <https://example.org/ns/mobility#> .
@prefix mobility: <https://example.org/ns/mobility#> .
@prefix rdf:      <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

<https://stad.gent/nl/mobiliteit-openbare-werken/parkeren/park-and-ride-pr/pr-gentbrugge-arsenaal>
        rdf:type                      mobility:offStreetParkingGround;
        mobility:availablespaces      0;
        mobility:freeparking          1;
        mobility:gentse_feesten       "True";
        mobility:isopennow            1;
        mobility:lastupdate           "2023-11-30T21:45:15+01:00"^^<http://www.w3.org/2001/XMLSchema#dateTime>;
        mobility:latitude             "51.0325480691";
        mobility:location             [ mobility:lat  5.10325480691E1;
                                        mobility:lon  3.7583663653E0
                                      ];
        mobility:longitude            "3.7583663653";
        mobility:name                 "P+R Gentbrugge Arsenaal";
        mobility:numberofspaces       0;
        mobility:occupancytrend       "unknown";
        mobility:openingtimesdescription
                "24/7";
        mobility:operatorinformation  "Mobiliteitsbedrijf Gent";
        mobility:temporaryclosed      0 .
```

## That's All Folks
You now know how to configure a basic LDIO workbench which takes in RDF messages containing a single state object and turn it into a version object that can be ingested as a LDES member.

To bring the containers down and remove the private network:
```bash
docker compose down
```
