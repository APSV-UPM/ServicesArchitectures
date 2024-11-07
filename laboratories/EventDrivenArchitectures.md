# Event Based Architecture: Case study on logistic transportation <img src="../statics/common/under_construction_sign.png" alt="sign_under_construction" width="35"/>

In our case study, traces are produced by the IoT subsystem in real-time, and 
they are injected in a kafka queue (by a simulator called 
`GPSEnabledTrucksSimulator`) to be consumed asynchronously by the 
`TraceServer`. This `TraceServer` is in charge of storing `Traces` and updating 
`TransportationOrders` if they match arriving `Traces`.

The whole operation of this component is launched by the arrival of a new 
`Trace` in the kafka queue. This is the base of Events Driven Architectures: 
services execution are launched by certain events -in this case the arrival of 
the `Trace`. The advantages of this approach are clear: the only dependency of 
services is to the source of events (the queue) and the event contents (the 
event type). This helps in keeping the complexity of service interaction under 
control, but the radical application of EDA impedes synchronous interactions 
(sending a direct request to a REST API). So it is a question of judgment about 
architectural design to decide the extent of application of EDA.

In former implementations of libraries, the services listened to the queues by 
polling (asking for data at certain moments in time), which made execution 
slower and wasted machine cycles. The usage of reactive patterns, implemented 
by means of functional programming in Java turns the control out so the 
component registers a function in the queue, and is the queue that activates 
this function. SB implements this very elegantly, although the code, that makes 
usage of these functional and generic programming elements and SB annotations, 
looks definitely different from usual Java pre-version 8.

The full scenario comprises the project `GPSEnabledTrucksSimulator` (which can 
be imported from GitHub 
https://github.com/juancarlosduenas/GPSEnabledTrucksSimulator.git) that behaves 
as a simulator of GPS devices mounted on trucks -producing `Traces`. There is 
also a kafka queue whose configuration files can be found in that same project; 
this will not work unless docker is installed in your machine. The 
`TransportationOrderServer0` project must be in place so `TransportationOrders` 
are stored utilizing its microservice. In its first version, the project 
`TraceServer0` must hold the operations required to get Traces from the kafka 
queue, store them and update `TransportationOrders` by a synchronous-API call 
to the `ordermanager` service.

If you need different versions of the same project, you can clone VSC maven 
projects: copy and paste the complete folder, rename it, and write the new 
project name in the proper place in `pom.xml`.

## Kafka queue definition

You must ensure docker has been installed on your machine; then check the 
following files appear in `GPSEnabledTrucksSimulator/src/main/resources`.

`docker-compose.yml`: it is in charge of configuring kafka and exposing ports.
```
---
version: '2'
services:
    kafdrop:
        image: obsidiandynamics/kafdrop:3.8.1
        container_name: kafdrop
        depends_on:
          - zookeeper
          - kafka
        expose:
          - 9000
        ports:
          - 9000:9000
        environment:
          ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_BROKERCONNECT: kafka:29092
    
    zookeeper:
        image: confluentinc/cp-zookeeper:latest
        container_name: zookeeper
        ports:
          - 2181:2181
        environment:
          ZOOKEEPER_CLIENT_PORT: 2181
          ZOOKEEPER_TICK_TIME: 2000
    
    kafka:
        image: confluentinc/cp-kafka:latest
        container_name: kafka
        depends_on:
          - zookeeper
        ports:
          - 9092:9092
          - 29092:29092
        environment:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
          KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
          KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
          KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

First, run Docker desktop on your machine. Then run `start-servers.sh`: a 
(Linux-Unix) shell script for starting the execution of kafka-zookeeper 
containers; you can execute these orders by hand if running on a different 
operating system.

```
docker-compose up -d
docker-compose logs -f
```

`stop-servers.sh`: the same to stop execution.

```
docker-compose rm -fsv
```

Both files must be given execution permissions:
```
chmod 700 start-servers.sh
chmod 700 stop-servers.sh
```

Take in mind that newer versions of Docker does not recognize `docker-compose` 
as a command and must be substituted with `docker compose`.

If you want to use DIT lab machines, you must start Docker this way:
- open a new terminal window and run `sudo /usr/sbin/usermod -a -G docker $USER`
- open a second new terminal window and there run `slogin localhost`
- you can check your user has been added to the docker group by using the 
command: `id`
- then you can run `start-servers.sh`
- Then change _docker-compose_ by _docker compose_ in those files before 
running `start-servers.sh`

## GPSEnabledTrucksSimulator

The tracking system would receive traces generated by GPS terminals, which 
would send (write) in the kafka queue. To test our implementation, we provide a 
separate project for a simulator of traces injection. It is simply a SB Java 
project that reads a file containing `Traces`, as JSON objects so they can be 
handled directly as `Strings` in a `List`; and writes these `Strings` into the 
queue. It uses Spring Cloud Stream as library support.

An example of these traces is:
```
{"traceId":"-","truck":"7208RMB","lastSeen":1617084900000,"lat":40.28668150060635,"lng":-3.854276103837144}
{"traceId":"-","truck":"7208RMB","lastSeen":1617085800000,"lat":40.12609584312012,"lng":-3.838699438067334}
{"traceId":"-","truck":"7208RMB","lastSeen":1617086700000,"lat":39.96871167177502,"lng":-3.8234333136021945}
```

The project `GPSEnabledTrucksSimulator` is a single class SB application:

```
package es.upm.dit.apsv.gpsenabledtruckssimulator;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Supplier;
import java.util.streamCollectors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframwork.util.ResourceUtils;

@SpringBootApplication
public class GPSEnabledTrucksSimulatorApplication {

        private static final Logger log = LoggerFactory.getLogger(
            GPSEnabledTrucksSimulatorApplication.class);
        private static int n = 0;
        private static List<String> messages =new ArrayList<String>();

        public static void main(String[] args) throws IOException {
                SpringApplication.run(GPSEnabledTrucksSimulatorApplication.class, args);
                log.info("Generación de trazas activa...");

                File file = ResourceUtils.getFile("classpath:tracesJSON.json");
                System.out.println("Reading from: tracesJSON.json");
                FileReader fr= new FileReader(file);
                messages = new BufferedReader(fr).lines()
                    .collect(Collectors.toList());
                fr.close();
        }

        @Bean("producer")
        public Supplier<String> checkTrace() {
                Supplier<String> traceSupplier = () -> {
                        if (messages.size() >0) {
                                return messages.get(n++ % messages.size());
                        }
                        else return null;
                };
                return traceSupplier;
        }
}
```

Reading the file contents as Strings, splitting into rows, and populating a 
`List` is made in a single line thanks to the `java.util.Stream` package. 
Please note the "producer" bean which returns a (lambda) function without 
parameters called `traceSupplier` fitting the `Supplier<String>` type (returns 
`String`, no params). The bean registers this function in the SB engine, and 
this activates it periodically. Each time it is called, it returns a message 
from the `List`, which is written to the kafka queue. As this producer is 
simply handling `Strings`, the `Trace` entity is not required, but we must 
ensure the textual JSON format we are using here fits the consumer expectation 
-this is, decoding the JSON text renders a `Trace` object.

Of course, the code is so simple because default configuration settings are 
set. For non-default ones we can give values in either of two files: 
`application.properties`.

```
# This setting controls the switch between modes
spring.profiles.active=@spring.profile.activated@

# Rate of message production (1000 = 1s)
# spring.cloud.stream.poller.fixed-delay=1000
server.port=8085
spring.application.name=GPSEnabledTrucksSimulator-kafka
```

And `application.yml`, which contains the configuration information SB needs to perform the connection between the producer bean and the actual queue. Please write this exactly as it is; any typo may lead to inconsistent execution without any clue of the mistake.

```
spring:
    cloud:
        stream:
          default-binder: kafka
          kafka:
            binder:
              brokers:
              - localhost:9092
          bindings:
            producer-out-0:
              destination: "test"
        function:
          definition: "producer"
```

## Trace consumer: asynchronous-event driven

Take as starting point your `TraceServer` project that provides entity, 
persistence, and REST service implementation for `Traces`. We will trigger this 
service on the arrival of a `Trace`, such as in EDA. Again we are going to take 
advantage of Spring Cloud Stream library to implement the reading of `Traces` 
in a reactive, asynchronous manner so instead of polling the queue, we register 
a `Consumer<Trace>` function that takes a `Trace` as a parameter (a `Consumer` 
takes a single parameter and produces no result-void). Whenever a new `Trace` 
is available, this function is called.

Of course, configuration files play their roles in the consumer. The same 
considerations made in `GPSEnabledTrucksSimulator` must be done again for 
`application.yml` (see the name of the bean is the same of 
`function:definition:`).

```
spring:
  cloud:
    stream:
      default-binder: kafka
      kafka:
        binder:
          brokers:
          - localhost:9092
      bindings:
        consumer-in-0:
          destination: "test"
          group: "${spring.application.name}"
    function:
      definition: "consumer"
```

The `application.properties` file states this service will run on port 8083.

```
spring.application.name=TraceServer-kafka
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
spring.jpa.hibenate.ddl-auto=create

server.port=8083
#http://localhost:server.port/spring.h2.console.path
orders.server=http://localhost:8080/transportationorders/
```

The very first version of `TraceServerApplication` simply sets the id and saves 
the Trace, so its `TraceRepository` must be defined as an attribute. Some 
additional imports are included.

```
package es.upm.dit.apsv.traceserver;

import java.util.function.Consumer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.core.env.Environment;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.RestTemplate;

import es.upm.dit.apsv.traceserver.model.Trace;
import es.upm.dit.apsv.traceserver.model.TransportationOrder;
import es.upm.dit.apsv.traceserver.repository.TraceRepository;

@SpringBootApplication
public class TraceServerApplication {

        @Autowired
        private Environment env;

        public static final Logger log = LoggerFactory.getLogger(TraceServerApplication.class);
        public static void main(String[] args) {
                SpringApplication.run(TraceServerApplication.class, args);
                log.info("Prueba consumer arrancando...");
        }

        @Autowired
        private  TraceRepository traceRepository;

        @Bean("consumer")
        public Consumer<Trace> checkTrace() {
                return t -> {
                        t.setTraceId(t.getTruck() + t.getLastSeen());
                        traceRepository.save(t);
             };
        }
```


In order to check it is running properly:
- kafka server must be running (start-servers.sh)
- `GPSEnabledTrucksSimulator` must be running and injecting Traces in the kafka 
queue.
- `TraceServer` must be running.
- you can access http://localhost:8083/h2-console to see the `Traces` in 
`traces.json` file in `GPSEnabledTrucksSimulator` are inserted in the database.
- you can also access http://localhost:8083/traces to check the REST service is 
running

## Trace consumer: synchronous connection to ordermanager

The `checkTrace` function in `TraceServerApplication` is the core of the 
component itself -it should: (1) store the `Trace`, (2) perform the matching 
between `Trace` and `TransportationOrder` by obtaining the order with the same 
truck id as the trace, and if found (3) update the `TransportationOrder` 
attributes with the `Trace` values and updates the order in the database.

As we have developed the `TransportationOrderServer` and this service 
isolatedly (maybe by different teams, and deployed each on its own application 
and server), the requests to get a `TransportationOrder` with the same truck id 
and later update it in the database, should be sent to a different service 
synchronously -requesting the other service and waiting to get the response. 
The service URI is set in the `orders.server` configuration property and 
interactions will now be done by the `RestTemplate` class provided by SB, which 
is in charge of assembling the HTTP+JSON request and obtaining results as Java 
objects. As we are going to handle `TransportationOrder` objects, the class 
must be in the model package, although no JPA-entity support nor repository is 
required so far.

The `checkTrace` method is in `TraceServerApplication`; `TraceRepository`, and 
`Environment @Autowired` attributes must be defined. Some imports are included 
below for convenience.

```
package es.upm.dit.apsv.traceserver;

import java.util.function.Consumer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.RestTemplate;

import es.upm.dit.apsv.traceserver.model.Trace;
import es.upm.dit.apsv.traceserver.model.TransportationOrder;
import es.upm.dit.apsv.traceserver.repository.TraceRepository;

@SpringBootApplication
public class TraceServerApplication {

    public static final Logger log = LoggerFactory.getLogger(TraceServerApplication.class);
    
    private final TraceRepository tr;

    public TraceServerApplication(TraceRepository tr) {
            this.tr = tr;
    }

    public static void main(String[] args) {
            SpringApplication.run(TraceServerApplication.class, args);
            log.info("Prueba consumer arrancando...");
    }

    @Bean("consumer")
    public Consumer<Trace> checkTrace() {
        return t -> {
            t.setTraceId(t.getTruck() + t.getLastSeen());
            tr.save(t);
            String uri = "http://localhost:8080/transportationorders/";
            RestTemplate restTemplate = new RestTemplate();
            TransportationOrder result = null;
                
            try {                        

              result = restTemplate.getForObject(uri + t.getTruck(), TransportationOrder.class);
            } catch (HttpClientErrorException.NotFound ex)   {
                    result = null;
            }
                
            if (result != null && result.getSt() == 0) {
                    result.setLastDate(t.getLastSeen());
                    result.setLastLat(t.getLat());
                    result.setLastLong(t.getLng());
                    if (result.distanceToDestination() < 10)
                            result.setSt(1);
                    restTemplate.put(uri, result, TransportationOrder.class);
                    log.info("Order updated: "+ result);
            }
        };
    }
}
```

Please notice that we consider a `TransportationOrder` changes its status to 1 
(complete) if the distance of its last seen location to the destination is 
lower than 10 (an arbitrary value). This method should be defined in the 
`TransportationOrder` class as the Euclidean distance between the two points.

In this final configuration, launching the whole system also requires 
`TransportationOrderServer` to be running and fed with valid 
`TransportationOrders` (from `orders.json` for example, see previous document). 
The complete sequence is:
- Docker, kafka servers running (start-servers.sh)
- Launch TransportationOrderServer
- Execute loadorders.sh
- Check TransportationOrderServer is running (via web or h2-console access to its port)
- Launch GPSEnabledTrucksSimulator
- Check GPSEnabledTrucksSimulator is running and putting traces in the kafka queue
- Launch TraceServer
- Check TraceServer is running (via web or h2-console access to its port)
- Check that after some time there are processed traces in TraceServer database
- Check that after some time some TransportationOrders have changed their lastSeen attributes, or even their Status.

You can interact with `transportationorderserver` via browser or Swagger; and 
access the h2 databases (`Traces` in `TraceServer`, `TransportationOrders` in 
`TransportationOrderServer`) to see that some orders get their last position 
updated, and those close to their destination get status 1.

## Assignment

Develop the complete `TraceServer` project in its own folder. Zip-compress it 
in a file called `APSV-24-25-ASSIGN-20-SURNAME1-NAME1–SURNAME2-NAME2.zip`. 
Upload to the entry in moodle.

If you have got to this point I would be grateful if you could send me your 
comments or suggestions for improvement.

