# Enviroment Configuration <img src="../statics/common/under_construction_sign.png" alt="sign_under_construction" width="35"/>

Modern software development relies on components, libraries and other 
artifacts so developers focus on the core of their job. Reusing predefined 
elements enhances productivity but as application size grows, managing this 
large number of items can only be done with automation. Programming 
environments bring this automation by assisting in writing the code, providing 
boilerplate code, checking correctness, compiling, linking with libraries, 
keeping these updated, launching applications with their supporting platforms, 
etc. 

This document helps in setting and configuration of an environment such as we 
need for laboratory sessions and assignments in this part of the subject. You 
need to have working knowledge of your operating system. 

Instead of your own computer, you can use the machines offered by 
**Departamento de Ingeniería de Sistemas Telemáticos (DIT)** in several labs. 
The software packages have already been installed for Lab DIT Debian remote 
machines. If you prefer using these please follow the instructions 
[here](www.lab.dit.upm.es).

## Software Installs
In developing the case we have mainly used Visual Studio Code, 
[VSC](https://code.visualstudio.com/) version Darwin, a complete coding 
environment that supports most development activities, for many programming 
languages including Java. VSC installs some other tools such as:

- Java development kit (JDK), which contains the complete set of basic 
libraries and tools for compiling and running Java programs. To install a new 
version, or choose one already installed, in VSC do `Ctrl-Shift-P (Cmd-Shift-P 
for Macos)` and write `Java: Configure Java Runtime` in the command window. 
This offers the option of using the installed versions, or downloading and 
installing a new one. For labs we will use **Adoptium’s Temurin 11 (LTS)**. 
Alternatively, you may download and install JDK from any other 
[source](https://jdk.java.net/archive/) but ensure you are using version 11. To 
do this step you need the proper VSC extensions installed described in 
[this section](#visual-studio-code-for-spring-boots-projects)
  - [Maven](https://maven.apache.org/install.html) is the dependency and 
  builder manager for Java projects VSC will rely on. VSC installation also 
  includes maven so it should not be required its external installation

- [Git](https://git-scm.com/) a free and open source distributed version 
control system. With some versions is directly included in VSC. If not 
installed do it manually.

- [Docker](https://www.docker.com/get-started) a platform that uses OS-level 
virtualization to deliver software in isolated containers with their own 
software, libraries and configuration, that communicate with each other 
through well-defined channels. On first execution, we will download, install 
and launch [kafka](https://kafka.apache.org/) version 3.2, an event streaming 
platform that supports pub-sub asynchronous interaction model 

For the installed packages, version numbers may not be exactly these but using 
unstable newest versions often leads to incompatibility errors, while very old 
versions may contain deprecated elements.

## Software Projects
We will work on three projects, the code for the three of them can be found on 
the GitHub platform:
- [TransportationOrderServer0](
https://github.com/juancarlosduenas/TransportationOrderServer0.git) for 
handling TransportationOrders. 
- [TraceServer0](https://github.com/juancarlosduenas/TraceServer0.git) for the 
actual Traces management code.
- [GPSEnabledTrucksSimulator](
https://github.com/juancarlosduenas/GPSEnabledTrucksSimulator.git) for 
simulation of Traces generation by GPS devices in trucks.

You must create a folder in your file system for which you have complete 
access rights; it will be your development folder (and the entry point for VSC 
workspace). Download and unzip the projects within this folder.

You could also clone the projects with `git` command inside your development 
folder using
```
git clone https://github.com/juancarlosduenas/TransportationOrderServer0.git
git clone https://github.com/juancarlosduenas/TraceServer0.git
git clone https://github.com/juancarlosduenas/GPSEnabledTrucksSimulator.git
```

## Visual Studio Code for Spring Boots projects
VSC is a powerful generic environment for programming. 
[Here](https://www.youtube.com/watch?v=uq4GjRF_860) you may see how to prepare 
it for Java-Spring Boot programming after installation.

First, you should add extensions by clicking the Extensions icon on the left 
side of VSC and writing their names in the Extensions bar:
- `Extension Pack for Java` for programming -it includes several sub-packages; 
install it. If no JDK is found in the machine, this launches its installation 
-make sure you use Java 11.0 version.
- `Spring Boot Extension Pack` install it.

Then you must set your development folder, in VSC, `File->Open folder…` Use 
your development folder with the projects you have just downloaded.

If you need to create a brand-new Java-SB project from scratch, fill out the 
following [form](https://start.spring.io/), answering the questions so you get 
a zip file that you must download and unzip in your development folder.

A series of questions appear:
- project: `Maven`
- Spring Boot version: `3.3.5`
- project language: `Java`
- group: `es.upm.dit.apsv`
- artifact: `traceserver`
- packaging type: `jar`
- Java version: `17`
- add dependencies: `Spring Boot DevTools` and `Spring Web`
- place for the project: a new folder under your development folder.

For this environment configuration is not necessary to create a new Spring Boot 
project as we are going to use the repositories downloaded or cloned from 
GitHub.

## Development of Java-SB projects with VSC
A project such as TransportationOrderServer0 contains sub-folders and files: 
- `pom.xml` gather all the information of libraries and versions to be used in 
the project, so Maven downloads them to keep the project stable. Maven is also 
in charge of producing a jar file to be executed on the server.
- `src/main/java` contains the code following the hierarchical organization of 
Java packages
- `src/main/resources` contains configuration files such as 
`application.properties`, and any other helper file like the shell script 
`loadorders.sh` and the data file `orders.json` (these are used to populate the 
database of this service).

If VSC is not considering the project as a Java or SB one, this means the 
`pom.xml` has not yet been processed. You can force check for all the existing 
projects executing: (ctrl-shift-P / cmd-shift-P) `Java: Reload Projects`

As said, `pom.xml` is the file where Maven gets dependencies and building 
information so we can go from code to a running application. This file is 
automatically created by `Spring Initializer` with the data we provided, but 
often we must add more dependencies to libraries. It comprises several 
sections, such as: `modelVersion`, `parent`, `groupId`, `artifactId`, `name`, 
`properties`, `dependencies`, `dependencyManager` and `build`.

Spring Boot projects are runnable web applications, as they usually contain the 
following piece of code, that launches an embedded web server; it has been 
generated automatically.
```
@SpringBootApplication
public class TransportationOrderServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(TransportationOrderServerApplication.class, args);
    }
```

To run our web application the safer way is to run it using the terminal 
included in VSC. It is a command window for the operating system, so you can 
use the proper commands to change directories, list files, run command line 
applications and so on. We will ask Maven to build the application and run it. 
You can run the project from the terminal window, in the project base 
directory execute:
```
cd TransportationOrderServer0
./mvnw clean package spring-boot:run
```

If you are working on Windows, do this instead:
```
cd TransportationOrderServer0
./mvnw.cmd clean package spring-boot:run
```

It will run; there is nothing to do but traces of server execution will appear.

Now you can access different URLs for information and management:
- The service: http://localhost:8080/
- Different entry points of your service: 
http://localhost:8080/swagger-ui/index.html
- Database management: http://localhost:8080/h2-console

If you have got to this point, you are ready to start the actual development. 
In the next sessions, we will add Java classes in `src/main/java`. The 
contextual menu will help us with entries `New Package` and `New Java Class`. The 
compiler will be running in the background, so potential compiling errors are 
marked as you type.
