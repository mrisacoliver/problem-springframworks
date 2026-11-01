# microvices-springframworks

Spring optimized memory usage in the Kubernetes POD when using Tomcat

1. Native Container Awareness
In the past (early Java 8), the JVM did not know it was running inside a Docker/Kubernetes container. It relied on the host machine’s (Worker Node) memory. If the node had 64 GB, the JVM would set the initial heap to up to 25% of that value (~16 GB), immediately exceeding the POD’s `limits.memory` and triggering an `OOMKilled` error (Exit Code 137).


Solution: Starting with Java 8u191 (and by default in Java 11+ / Spring Boot 2+), the JVM began reading Linux cgroups. If you set a 1 GB limit on the POD, the JVM understands that 1 GB is its “total available RAM.”


2. Spring Boot 2.x / 3.x and Off-Heap Memory
The memory used by Tomcat and Spring in a POD isn’t limited to the heap (where Java objects reside). There’s also off-heap/native memory:


Metaspace: Classes loaded by Spring/Tomcat.


Threads: Each Tomcat thread allocates native stack space (typically 1 MB per thread on Linux).


Native buffers (NIO): Used by Tomcat for high-performance network I/O connections.


Spring Boot has adjusted its defaults, and the paketo-buildpacks library (used when generating images via `mvn spring-boot:build-image`) automatically calculates the optimal Heap and Off-Heap sizes based on Kubernetes limits.


3. Embedded Tomcat Tuning
The Tomcat embedded in Spring Boot has received significant optimizations:


Max Threads: The historical Tomcat default was up to 200 concurrent threads. In a POD with CPU limitations (e.g., cpu: 500m), 200 threads cause high context-switching overhead and consume ~200 MB of stack memory alone.


Recommended setting: In microservices and Kubernetes, it is common practice to limit Tomcat threads (server.tomcat.threads.max=50 or 100), which significantly reduces memory consumption outside the heap.


4. Support for Virtual Threads (Spring Boot 3.2+ / Java 21)
With Spring Boot 3.2 and Java 21, setting `spring.threads.virtual.enabled=true` causes Tomcat to process requests using Virtual Threads (Project Loom) instead of creating native operating system threads. This drastically reduces memory consumption per concurrent connection, allowing the POD to process thousands of requests with a fraction of the previous memory usage.


Recommended Configuration in Kubernetes
When defining resources in Kubernetes for a Spring Boot + Tomcat application, configure:

