# Arquitecture <img src="./statics/common/under_construction_sign.png" alt="sign_under_construction" width="35"/>

Once a review of the Logistics case study domain model has been done, it is 
time to think about architectural and detailed design: which pieces and 
components will be developed with the current technology. We will rely on the 
application of Domain Driven Design and use C4 diagrams for documentation.

## Tracking system context diagram

The C4.1 context diagram is as follows: the tracking system gets 
Transportation Orders -these are received by each truck, and traces are sent 
by GPS devices onboard. The tracking system main goal is keeping the status of 
each TransportationOrder updated so users get to know the actual position of 
each truck, its status (onRoad, arrived), and the expected time of arrival 
(ETA).

![tracking_system_context_diagram](
./statics/arquitecture/tracking_system_context_diagram.png)

## The core tracking domain model

Getting back to the general domain model already defined, and as DDD advises, 
we will strive for simplicity. As regards the Tracking system, this implies 
getting only these two concepts as relevant and trying to simplify the 
relationship between them.

![tracking_core_domain_model](
./statics/arquitecture/tracking_core_domain_model.png)

The relationship between TransportationOrder and Trace will be substituted by 
the inclusion of the truck id as a field that both have in common, as the 
traces come from the truck and the TransportationOrders are given to a single 
truck. Of course this means we will add some more code than if we directly 
support the relationship but this code is quite simple.

The truck id may be the key for TransportationOrders provided we only store 
live orders -historic records must be stored elsewhere. It can not be the key 
for Traces as a truck will generate many of them. In any case, having the same 
truck id is a good way to correlate the Trace with the TransportationOrder.

Coordinates will be stored as latitude and longitude double values, and dates 
and timestamps will be handled with long values (which is very common and handy 
if the time is referred to the same origin). No helper classes will be defined 
to keep it simple, and status will be stored as an int.

The minimum set of attributes for TransportationOrders is:
- String truck as key
- long originDate
- double originLat
- double originLong
- long dstDate
- double dstLat
- double dstLong
- long lastDate
- double lastLat
- double lastLong
- int st for the status.

And the set of attributes for a Trace:
- String traceId as the key, obtained as a concatenation of truck id and timestamp (lastSeen)
- String truck
- long lastSeen
- double lat
- double lng

## Tracking system containers diagram

As two different pieces of information are handled by the Tracking system, and 
according to DDD which advises setting each as a separate entity, we will 
define a service (container) for each one. So we will have two microservices.

Each service will be providing functionality to different actors/external 
systems. The Customer who needs to know the actual position of a Truck and its 
ETA, will interact with the TransportationOrder server. On the other hand, the 
Trace server is specialized in receiving Traces from the GPS-enabled trucks; we 
can not foresee the amount and frequency of arrival of new Traces but the Trace 
server needs to cope with higher traffic levels than the TransportationOrder 
server. Besides, the Trace server needs to react whenever an event -a Trace- 
arrives from the Kafka queue, in an asynchronous manner with respect to the GPS 
application on board of the trucks. The TransportationOrder server gives 
responses in a synchronous way to its clients.

Please notice each server will run in a different port, 8080 for 
TransportationOrder server and 8083 for Trace server so no collisions appear. 
Also interesting, the Trace server will perform requests to the 
TransportationOrder server for each arriving Trace through the REST-API. These 
requests will be sent using the HTTP protocol and will arrive at port 8080 of 
the machine. If we intend to use this architecture for a very large amount of 
Traces, the delays imposed by doing these requests by means of the API-REST 
over HTTP would be significant; in some cases, we should move both services to 
the same Spring server on the same machine so both servers share the same 
memory which is several orders of magnitude faster.

In summary, we will keep the TransportationOrder server application and the 
Trace server application as separate containers. Both have their own database. 
Traces will asynchronously arrive in the system through a kafka queue. The 
Trace server application will interact with the TransportationOrder server 
application in a synchronous way (accessing the API-REST). There will be client 
applications for each server application.

![tracking_system_containers_diagram](
./statics/arquitecture/tracking_system_containers_diagram.png)

## Tracking system components diagram

The C4 components diagram adds information about the internals of services. Our 
TransportationOrder server is a REST-API server that supports a limited set of 
operations (basically Create, Read, Update, Delete -CRUD) performed on the 
TransportationOrder entity, as advised by DDD. The operations will be called 
from mobile or desktop Javascript client applications or any application able 
to generate HTTP requests and decode HTTP responses carrying JSON data.

The TransportationOrder controller gets REST-API requests, sends the actions to 
the database and packages responses. This controller will have a method per 
path and request as shown in the table below. The TransportationOrder 
repository holds all the code needed to interact with the database.

| HTTP method | Domain                        | Semantics                             |
|-------------|-------------------------------|---------------------------------------|
| GET         | /transportationorders         | findAll                               |
| GET         | /transportationorders/{truck} | findById (truck)                      |
| POST        | /transportationorders         | save (neworder, in message body)      |
| PUT         | /transportationorders         | save (updated order, in message body) |
| DELETE      | /transportationorders/{truck} | deleteById (truck)                    |

As regards the Trace server, the Trace consumer component does not necessarily 
support REST-API requests; instead, it is attached as a consumer to the Kafka 
queue where the trucks are putting Traces containing the relevant information. 
Whenever a new event arrives from the Kafka queue, the consumer checks it is 
correct, generates a Trace, uses this information to update the proper 
TransportationOrder, and stores the Trace using its repository. This component 
follows the Events-Driven-Architecture principles.

![tracking_system_components_diagram](
./statics/arquitecture/tracking_system_components_diagram.png)

In summary, the TransportationOrder server and the Trace server are separate 
containers, each connecting to its own database. The TransportationOrder server 
offers an API-REST -synchronous connection, and it holds the API-REST 
controller and a repository component for connecting to the database. The Trace 
server application is composed of a Trace consumer that triggers whenever a new 
event (Trace) gets in the Kafka queue, and a repository component for 
connecting to the database.

## Tracking system classes diagram

In each server we find a software stack, bottom-up composed of the controller 
and repository components. The controller component is composed by the 
controller class (containing the REST-API methods) and the entity defined 
previously from the domain model. The repository component also uses the 
entity, and holds the repository interface with the headers of methods for 
accessing the database programmatically, the implementation of these methods 
-that will be provided by Spring in an automatic way- and the DataSource which 
is a class for configuration properties of the database connection whose values 
will be taken by Spring from the application.properties file.

So taking into consideration only the Tracking bounded context, and applying 
DDD principles:
- we have simplified the domain model, removing Events as we collated with 
Traces
- removed all the other associations
- generated a service per domain concept
- generated an entity per domain concept with a lifecycle
- generated a repository per domain concept whose state will be stored

![tracking_system_classes_diagram_1](
./statics/arquitecture/tracking_system_classes_diagram_1.png)

For the implementation of microservices, we will use the Spring Boot Java 
framework. It eases the development by applying the “convention over 
configuration” approach which means that default values, classes and interfaces 
will be used unless configuration info is provided. Spring Boot also connects 
automatically classes (beans) if there is only one fit with respect to their 
type.

Each service will be defined with its API-REST and will implement the data 
persistence:
- each service gets its controller
- each one gets its entity
- each one gets its repository divided into interface and implementation
- each one gets its persistence provider, data source and database
- repository implementations, persistence providers and data sources are made 
automatically by SB
- TraceService needs a component to get Traces from the middleware queue in an 
asynchronous way
- TraceService interacts with TransportationOrderService to get/update the 
current status of TransportationOrders
- TraceProducer (GPSEnabledTrucksSimulator project) simulates the production of 
Traces from trucks

![tracking_system_classes_diagram_2](
./statics/arquitecture/tracking_system_classes_diagram_2.png)

To test the system, as no real GPS-enabled trucks will be used, we provide the 
GPSEnabledTrucksSimulator project. This project emulates the production of 
Traces in several trucks at various positions and times and injects them into 
the Kafka queue.
