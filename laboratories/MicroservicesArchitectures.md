# Microservices Implementation: Case study on logistic transportation <img src="../statics/common/under_construction_sign.png" alt="sign_under_construction" width="35"/>

The implementation of microservices in Java is largely eased by the usage of 
Spring Boot. This framework is composed of a large set of libraries and builds 
upon the language, extends it with autowiring capabilities for selecting and 
connecting classes, links with components for data storage, launching servers, 
configuring applications, running microservices, accessing external queues in a 
reactive way, etc.

To start this session you must have got your development environment ready, and 
have been downloaded and charged in VSC the projects 
`TransportationOrderServer0` and `TraceServer0`.

Next, we explain the development of the `TransportationOrderServer0` project 
that you have downloaded from GitHub; it handles `TransportationOrder`. As we 
are going to develop two projects and two different SB applications, they will 
run on their own server, and therefore they must be attached to different ports.

In each project, the file `application.properties` configures the h2 embedded 
database where the objects will be stored. Also, the port where this service 
runs can be set (`server.port`, that must be different from 8080); other 
application configuration properties can be added. These are the contents of 
the file in the project `TraceServer0`.

```
spring.application.name=traceconsumer-kafka
spring.profiles.active=@spring.profile.activated@

#H2 settings
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
spring.h2.console.settings.trace=true
spring.datasource.url=jdbc:h2:mem:apsv
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create

server.port=8083
orders.server=http://localhost:8080/transportationorders/
```

In the project, the code will be located in `src/main/java` and there will 
appear three Java packages under `es.upm.dit.apsv.transportationorderserver`: 
`model`, `repository`, `controller`. The same shall be done for `TraceServer`.

## Entities definition

Let us start with the persistence layer, which is composed of the entities that 
will be mirrored by tables in the database and the methods for backup and 
retrieving objects from/to the database.

Developing the code for each entity is easy: define the class, define the 
attributes and using the proper helpers in the environment (Source Action... in 
the contextual menu) add default constructor, constructor with all fields, 
setters and getters, `toString`, `hashCode` and `equals` methods (two 
`TransportationOrder` are the same if correspond to the same truck, two 
`Traces` are the same if they have the same id). The class itself must be 
annotated with `@Entity`, and the key attribute with `@Id`.

`TransportationOrders` represent live orders for transportation, containing 
info about origin location and date, destination location and date, last time 
and location seen, and status. The truck id is the key. This has been already 
defined in the architecture document. Next, part of the code of 
`TransportationOrder` is included. Please note that an additional `toid` 
attribute has been added for compatibility with the simulation file. The 
`Lombok` library contains annotations to automatically generate getters, 
setters, no-args constructor and `toString` methods. You must add the proper 
body to the constructor, add `hashCode` and `equals` methods by means of the 
contextual menu- `Source code actions`. The method `distanceToDestination` 
should also be included here for later usage.

```
package es.upm.dit.apsv.transportationorderserver.model;
import javax.persistence.Entity;
import javax.persistence.Id;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.ToString;

@Entity
@Getter @Setter @NoArgsConstructor @ToString
public class TransportationOrder {
    private String toid;
    @Id
    private String truck;
    private long originDate;
    private double originLat;
    private double originLong;
    private long dstDate;
    private double dstLat;
    private double dstLong;
    private long lastDate;
    private double lastLat;
    private double lastLong;
    private int st;

    public TransportationOrder(String toid, String truck, long originDate,
                               double originLat, double originLong,
                               long dstDate, double dstLat, double dstLong,
                               long lastDate, double lastLat, double lastLong,
                               int st) { ... }

    public double distanceToDestination() {
        return Math.sqrt(Math.pow(this.dstLat -this.lastLat, 2)
                    + Math.pow(this.dstLong - this.lastLong, 2));
    }
}
```


Traces arrive from trucks and contain the truck id, and the location and date 
this trace has been generated. The key traceId will be set as a concatenation 
of truck id and timestamp. The `Trace` entity is also very simple. Add its 
constructor body, `hashCode` and `equals` methods.

```
package es.upm.dit.apsv.traceserver.model;
import javax.persistence.Entity;
import javax.persistence.Id;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.ToString;

@Entity
@Getter @Setter @NoArgsConstructor @ToString
public class Trace {
    @Id
    private String traceId;
    private String truck;
    private long lastSeen;
    private double lat;
    private double lng;
    
    public Trace(String traceId, String truck, long lastSeen,
                 double lat, double lng) { 
    
    }
}
```


## Entities connection to the database

The persistence layer implements the Data Access Object pattern, which implies 
defining an entity -yet done-, and creating a class with the methods that 
actually perform operations with the database. This DAO class is composed of an 
interface with only method headers, and the implementation class with the 
methods bodies.

SB makes creating the DAO surprisingly easy, once an entity has been defined, 
requiring just an empty interface per entity in the right package (repository) 
with an indication of the entity and the key type. SB then reads the code and 
creates an implementation class automatically for the most usual methods (CRUD, 
readAll and others); even if needed, SB can create the implementation of 
methods once their headers are included in the interface and their names follow 
some rules (`findByTruckAndSt` in `TransportationOrderRepository` returns the 
order with that truck id and status).

```
package es.upm.dit.apsv.transportationorderserver.repository;

import org.springframework.data.repository.CrudRepository;
import es.upm.dit.apsv.transportationorderserver.model.TransportationOrder;

public interface TransportationOrderRepository extends
CrudRepository<TransportationOrder,String> {
    TransportationOrder findByTruckAndSt(String truck, int st);
}
```

In case you only need default methods the interface will be empty.

```
package es.upm.dit.apsv.traceserver.repository;

import org.springframework.data.repository.CrudRepository;
import es.upm.dit.apsv.traceserver.model.Trace;

public interface TraceRepository extends CrudRepository<Trace,String> {

}
```

## REST services implementations

The definition and implementation of REST APIs require some coding, but once 
the JPA persistence layer is done, it is reduced to defining a consistent API 
over HTTP primitives and creating very simple implementations. 
`TransportationOrders` API is shown below; the path indicates the URL fragment 
after the server and port to access the service. Semantics refer to the 
persistence order to invoke.

| HTTP primitive | Path                          | Semantics-service implementation      |
|----------------|-------------------------------|---------------------------------------|
| GET            | /transportationorders         | findAll                               |
| GET            | /transportationorders/{truck} | findById (truck)                      |
| POST           | /transportationorders         | save (neworder, in message body)      |
| PUT            | /transportationorders         | save (updated order, in message body) |
| DELETE         | /transportationorders/{truck} | deleteById (truck)                    |


Most implementations are direct calls to the persistence layer. Those methods 
that may render a null object must return instead a 
`ResponseEntity<TransportationOrder>`, where the object is included, or a 
`NOT_FOUND` message elsewhere.

This makes usage of SB annotations again, such as:
- `@RestController` to identify the class is a REST API implementation
- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` are located 
before methods to assign the path to that method; the path may contain variable 
parts
- `@PathVariable` takes the path variable value and inserts as a method 
parameter
- `@RequestBody` for a method parameter indicates that object will arrive as 
part of the message body

SB analyzes the class code and adds the necessary code to perform conversions 
from/to HTTP messages, JSON objects, Java objects, and returning values. 
Perhaps the largest part of the file is for imports (in VCS they are set by 
clicking a name and cmd key). Please note the attribute called repository and 
how it is set in the constructor; SB performs beans injection at this point so 
if SB discovers there is an implementation of the `TraceRepository` interface 
(the one SB has created automatically), creates an object and puts its 
reference at the start.

```
package es.upm.dit.apsv.transportationorderserver.controller;

import java.util.List;
import java.util.Optional;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import es.upm.dit.apsv.transportationorderserver.repository.TransportationOrderRepository;
import es.upm.dit.apsv.transportationorderserver.model.TransportationOrder;

@RestController
public class TransportationOrderController {
    private final TransportationOrderRepository repository;
    public static final Logger log = LoggerFactory.getLogger(
        TransportationOrderController.class
    );

    public TransportationOrderController(TransportationOrderRepository repository) {
      this.repository = repository;
    }

    @GetMapping("/transportationorders")
    List<TransportationOrder> all() {
      return (List<TransportationOrder>) repository.findAll();
    }

    @PostMapping("/transportationorders")
    TransportationOrder newOrder(@RequestBody TransportationOrder newOrder) {
      return repository.save(newOrder);
    }

    @GetMapping("/transportationorders/{truck}")
    ResponseEntity<TransportationOrder> getByTruck(@PathVariable String truck) {
      Optional<TransportationOrder> ot = repository.findById(truck);
      if (ot.isPresent())
        return new ResponseEntity<>(ot.get(), HttpStatus.OK);
      return new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }
    
    @PutMapping("/transportationorders")
    ResponseEntity<TransportationOrder> update(@RequestBody TransportationOrder updatedOrder) {
      TransportationOrder to = repository.save(updatedOrder);
      if (to == null)
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);
      return new ResponseEntity<>(to, HttpStatus.OK);
    }

    @DeleteMapping("/transportationorders/{truck}")
    void deleteOrder(@PathVariable String truck) {
      repository.deleteById(truck);
    }    
}
```

Instead of returning `TransportationOrders` in `findById`, an 
`Optional<TransportationOrder>` is returned and `isPresent()` method must be 
called to know if it is null.

`TraceController` is very similar; some methods have been implemented in a 
slightly different way, just to show a different style. See below:

```
package es.upm.dit.apsv.traceserver.controller;

import java.util.List;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import es.upm.dit.apsv.traceserver.model.Trace;
import es.upm.dit.apsv.traceserver.repository.TraceRepository;

@RestController
public class TraceController {

    private final TraceRepository tr;

    public TraceController(TraceRepository tr) {
        this.tr = tr;
    }

    @GetMapping("/traces")
    List<Trace> all() {
        return (List<Trace>) tr.findAll();
    }

    @PostMapping("/traces")
    Trace newTraze(@RequestBody Trace newTrace) {
        return tr.save(newTrace);
    }

    @GetMapping("/traces/{id}")
    Trace one(@PathVariable String id) {
        return tr.findById(id).orElseThrow();
    }
    
    @PutMapping("/traces/{id}")
    Trace replaceTraze(@RequestBody Trace newTrace, @PathVariable String id) {
        return tr.findById(id).map(Trace -> {
          Trace.setTraceId(newTrace.getTraceId());
          Trace.setTruck(newTrace.getTruck());
          Trace.setLastSeen(newTrace.getLastSeen());
          Trace.setLat(newTrace.getLat());
          Trace.setLng(newTrace.getLng());
          return tr.save(Trace);
        }).orElseGet(() -> {
          newTrace.setTraceId(id);
          return tr.save(newTrace);
        });
    }

    @DeleteMapping("/traces/{id}")
    void deleteTraze(@PathVariable String id) {
        tr.deleteById(id);
    }
}
```

## Launching the web app that hosts REST services

The REST services implemented need to be launched by a web app. SB provides a 
simple one (see Configuration document). Please notice the usage of SB 
annotation `@SpringBootApplication`. Before actually launching the execution SB 
identifies, creates and adds as attributes the objects of classes (beans) 
needed. This is called “dependencies injection”.

```
@SpringBootApplication
public class TransportationOrderServerApplication {

        public static final Logger log = LoggerFactory.getLogger(
            TransportationOrderServerApplication.class
        );
        
        private TransportationOrderRepository torderRepository;
        
        public static void main(String[] args) {
                SpringApplication.run(
                    TransportationOrderServerApplication.class, args
                );
        }
}
```

## Testing REST services

Once developed the REST API implementations and the applications as described, 
each service is launched either using the icons in VSC or the manual command in 
the command terminal in its project folder:

```
./mvnw clean package spring-boot:run
```

Of course, if more than one server is launched they must run in different ports 
(`TransportationOrderServer` will run on 8080). For `TraceServer` this 
parameter is set in the file `application.properties` in `src/main/resources` 
with this configuration variable:

```
server.port=8083
```

And after launching, several actions can be done to check things are running:
- access http://localhost:8080/h2-console opens a console for direct 
interaction with the in-memory h2 server that runs in each service (use port 
8083 for `TraceServer`). Warning: on the first screen of h2 console, get sure the 
JDBC URL contains the same value given to `spring.datasource.url` in 
`application.properties`.
- the `springdoc-openapi-ui` dependency in `pom.xml` adds runtime support to 
generate automatic documentation about your API-REST in the URL
http://localhost:8080/swagger-ui.html
- using the Web browser perform GET operations to get all elements, or one 
given its key value
- for TransportationOrderServer, if the `orders.json` file is in place, run the 
following shell commands to populate orders 

    ```
    while IFS= read -r line; do
    curl -H "Content-Type: application/json" -X PUT --data-binary "$line" http://localhost:8080/orders;
    done < orders.json
    ```

- for `TraceServer` create a file traces.json with the contents below, and adapt 
the shell script to populate traces

Example of `orders.json` contents (a `TransportationOrder` at a row):

```
{"toid":"85","truck":"0218LFH","originDate":1573974000000,"originLat":40.4562191,"originLong":-3.8707211,"dstDate":1573999427000,"dstLat":42.5519258,"dstLong":-8.9935558,"lastDate":0,"lastLat":0.0,"lastLong":0.0,"st":0}
{"toid":"59","truck":"0865FTJ","originDate":1568959200000,"originLat":40.4562191,"originLong":-3.8707211,"dstDate":1568962469000,"dstLat":40.9308098,"dstLong":-4.110524799999999,"lastDate":0,"lastLat":0.0,"lastLong":0.0,"st":0}
{"toid":"78","truck":"2915YYL","originDate":1578898800000,"originLat":40.4562191,"originLong":-3.8707211,"dstDate":1578903092000,"dstLat":40.6491754,"dstLong":-4.6736772,"lastDate":0,"lastLat":0.0,"lastLong":0.0,"st":0}
```

Example of `traces.json` contents (a `Trace` at a row):

```
{"traceId":"7208RMB-1","truck":"7208RMB","lastSeen":1617084900000,"lat":40.28668150060635,"lng":-3.854276103837144}
{"traceId":"7208RMB-2","truck":"7208RMB","lastSeen":1617085800000,"lat":40.12609584312012,"lng":-3.838699438067334}
{"traceId":"7208RMB-3","truck":"7208RMB","lastSeen":1617086700000,"lat":39.96871167177502,"lng":-3.8234333136021945}
```

# Assignment
Develop the complete `TransportationOrderServer0` and `TraceServer0` projects 
in their own folders. Zip-compress both folders and contents in a file called 
`APSV-24-25-ASSIGN-19-SURNAME1-NAME1–SURNAME2-NAME2.zip`. Upload to the Moodle 
entry.

