# Spring Boot Paketo Debug buildpack issue: "handshake failed - connection prematurally closed"

Sample project to reproduce an issue the the Paketo debug JVM buildpack

## Prerequisites 

Make sure to have Java JDK 11 installed. 
I'm using AdoptOpenJDK 11.0.11.

## Context

I'm using pack and Paketo buildpacks for building OCI containers for my Spring Boot applications (version 2.4.6).
To clarify I'm not using the Spring Boot Maven plugin to create an OCI image for my applications.
The applications are built using Maven and Java 11 and are running in the OCI container using JRE 11.
Yesterday I ran into the issue of not being able to start a remote JVM debugging session from IntelliJ to a running container.

```
Error running 'Remote Debug': Unable to open debugger port (:8000): java.io.IOException "handshake failed - connection prematurally closed"
```

## How to reproduce

1. Build the project:
   
```
./mvnw clean package
```

2. Build the OCI container using pack with debug enabled: 
   
```
pack build application/demo --builder paketobuildpacks/builder:base --path target/demo-0.0.1-SNAPSHOT.jar --env BP_JVM_VERSION=11 --env BP_DEBUG_ENABLED=true
```

3. Start the container: 
   
```
docker run --rm --tty --publish 8080:8080 --publish 8000:8000 --env BPL_DEBUG_ENABLED=true application/demo
```

From the IDEA start a remote JVM debug session:

![](/img/intelliJ-remote-debug-settings.png)

Error:

![](/img/intelliJ-remote-debug-settings-error.png)

## Java 8 works fine

When I downgrade the Spring Boot application to Java 8:

pom.xml:

```xml
<properties>
    <java.version>1.8</java.version>
</properties>
```

and build using JDK 8:

```
./mvnw clean package
```

And build the OCI container using `BP_JVM_VERSION=8`:

```
pack build application/demo --builder paketobuildpacks/builder:base --path target/demo-0.0.1-SNAPSHOT.jar --env BP_JVM_VERSION=8 --env BP_DEBUG_ENABLED=true
```

and start the OCI container:

```
docker run --rm --tty --publish 8080:8080 --publish 8000:8000 --env BPL_DEBUG_ENABLED=true application/demo
```

I can start the remote JVM debug session without any problem:
Connected to the target VM, address: ':8000', transport: 'socket'

## Work around

The workaround I use now is to use Java 11 and build without the `BP_DEBUG_ENABLED=true` flag

```
pack build application/demo --builder paketobuildpacks/builder:base --path target/demo-0.0.1-SNAPSHOT.jar --env BP_JVM_VERSION=11
```

and add JVM remote debug settings myself using the `JAVA_TOOL_OPTIONS`:

```
docker run --rm --tty --publish 8080:8080 --publish 8000:8000 --env JAVA_TOOL_OPTIONS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n
address=*:8000" application/demo
```

## Possible cause why it's not working for Java 11 

To me, it looks like the debug buildpack is not taking the difference between Java 8 and Java 11 command line arguments into account:
Java 8:

```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000
```

Java 11:

```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000
```

