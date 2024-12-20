# JVM Monitoring

1. `JDK_HOME\bin`
2. CLI vs GUI  
   - **CLI**: `jcmd`, `jps`, `jstack`, `jmap`,...  
   - **GUI**: `jconsole` -> JMX/MBean Console  
3. Client-Server -> Remote Monitoring  
4. JMX (Java Management eXtension) and MBeans  
    - i. Metric  
    - ii. Command  
    - iii. Observer/Event-Driven  
    - MBean Development  
5. JFR (JDK/JVM Flight Recorder)  
    - Custom Event  
6. VisualVM (NetBeans RCP)  
    - Monitoring, Profiling  
        - i. Sampling  
        - ii. Instrumentation  
    - MBean Console  
7. JMC (Oracle, Mission Control)  
    - i. JFR  
    - ii. Monitoring, MBean Console
    
# G1GC Configuration and Tuning: Throughput <--> Latency

```bash
-XX:+UseG1GC
-Xms4g -Xmx8g
-XX:MaxGCPauseMillis=50ms
-XX:G1HeapRegionSize=4m
```
increasing regions size -> GC Pause Time
                                     decreaseses overhead due to RSet, CSet          
```bash
-XX:InitiatingHeapOccupancyPercent=45
```
Marking Phase -> Decreasing the value -> decreaseses FullGC
                 increases # of Minor GC
```bash
-XX:+StringDeduplication
-XX:+PrintStringDeduplicationStatistics
```
use it if you %10 gain

```bash
-Xlog:gc+heap+stats,age*=debug:file=c:/tmp/myapp.log:time,uptime,level,tags			 
```
Humoungous: size(Object) >= RegionSize/2

# Automatic G1GC Configuration
```bash
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+UseAdaptiveSizePolicy \
     -XX:+G1UseAdaptiveIHOP \
     -XX:+ParallelRefProcEnabled \
     -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -jar your-application.jar
```
**-XX:+ParallelRefProcEnabled**: Enables parallel processing for reference objects, reducing latency during garbage collection
# G1GC Improvements for Large Heaps
In **Java 23**, **G1GC** handles large heaps (*up to* **16 TB**) more efficiently with less fragmentation and better pause time consistency. 

When the JVM allocates heap memory, it reserves the memory but doesn't immediately map it to physical memory. By default, physical memory pages are only allocated when the JVM writes to them for the first time during runtime (this is called "demand paging").

With **-XX:+AlwaysPreTouch**, the JVM proactively touches (writes to) every page of the heap during startup. This forces the operating system to allocate physical memory for the heap in advance.

For applications with very large heaps:
```bash
-XX:+AlwaysPreTouch
```

### Benefits of **-XX:+AlwaysPreTouch**
- **Reduced Runtime Latency**:
Pre-touching ensures that memory pages are already backed by physical memory, avoiding the overhead of demand paging during application runtime. This eliminates potential page faults, which can cause unpredictable pauses.
- **Heap Initialization Verification**:
By pre-touching the heap, the JVM verifies at startup that the operating system can actually allocate the requested amount of memory. This can prevent situations where an application starts successfully but fails later due to insufficient memory.
- **Improved Performance for Large Heaps**:
For applications with large heaps (e.g., >10GB), demand paging can cause significant latency during the first GC cycles. Pre-touching ensures that this latency is handled upfront, making GC behavior more predictable.

### Downsides of -XX:+AlwaysPreTouch
- **Longer JVM Startup Time**:
Since the JVM touches every page of the heap during startup, applications may take longer to initialize. This overhead can be noticeable for very large heaps.
- **Higher Initial Resource Usage**:
Pre-touching consumes CPU cycles during startup, and it may also stress the system's memory resources if multiple JVM instances are started simultaneously.
- **Not Necessary for Small Applications**:
For applications with small heaps or those that don't require low-latency behavior, this option may not provide significant benefits.

# Software Architecture

## I. Single Process, Multiple Threads
In this model, a single process is created, and multiple threads are spawned within the same process. Threads share the same memory space and resources of the parent process, allowing for lightweight task management. This approach is common in applications that need concurrent execution within the same task, such as:
- **Examples**: Web servers handling multiple client requests, GUI applications with a main thread and worker threads.
- **Advantages**:
  - Lower resource overhead compared to multiple processes.
  - Faster communication between threads (shared memory).
- **Disadvantages**:
  - Lack of memory isolation: a crash in one thread can affect the whole process.

## II. Multiple Processes
In this model, multiple independent processes are created, each with its own memory space. These processes do not share memory, and communication occurs via Inter-Process Communication (IPC) mechanisms like pipes, message queues, or shared memory.
- **Examples**: Running multiple instances of the same program, microservices architecture.
- **Advantages**:
  - Memory isolation: one process's crash does not affect others.
  - Scalability across multiple CPUs or machines.
- **Disadvantages**:
  - Higher resource usage (each process requires its own memory and resources).
  - IPC adds overhead compared to thread communication.

## III. Multiple Processes, Multiple Threads
This combines the benefits of both models by creating multiple processes, each with multiple threads. This approach is used in highly concurrent systems or distributed systems.
- **Examples**: Modern web browsers (separate processes for tabs, each with multiple threads), distributed computing frameworks.
- **Advantages**:
  - Combines the benefits of concurrency and isolation.
  - Fault tolerance: failure in one process does not necessarily affect others.
- **Disadvantages**:
  - More complex to implement and manage.
  - Higher resource usage compared to single-process threading.

# Remote Method Invocation (RMI)
RMI is a mechanism in Java that allows objects to invoke methods on remote objects (located on a different JVM). This is useful for creating distributed systems with object-oriented programming.
- **How it works**:
  - The server object is exposed via a remote interface.
  - Clients can call methods on the server object as if it were local.
- **Use cases**:
  - Distributed applications in Java.
  - Example: Banking systems, where remote method calls are required to access account data.

# Web Services

## SOAP Web Services (Simple Object Access Protocol)
SOAP is a protocol for exchanging structured information between web services using XML.
- **Characteristics**:
  - Platform and language-independent.
  - Strictly defined standards and protocols.
  - Uses WSDL (Web Services Description Language) for service definition.
- **Advantages**:
  - High security with WS-Security.
  - Reliable messaging (e.g., with retries and acknowledgments).
- **Disadvantages**:
  - Verbose XML format increases overhead.
  - Slower than lightweight alternatives like REST.

## RESTful Web Services (Representational State Transfer)
REST is an architectural style for designing networked applications using standard HTTP methods.
- **Characteristics**:
  - Stateless communication.
  - CRUD operations (Create, Read, Update, Delete) are mapped to HTTP methods (POST, GET, PUT, DELETE).
  - Lightweight compared to SOAP.
- **Advantages**:
  - Simple and flexible.
  - Wide adoption and easy integration with web-based systems.
- **Disadvantages**:
  - Lacks built-in security standards like SOAP.
  - May require custom solutions for transactions and reliability.

## gRPC (Google Remote Procedure Call)
gRPC is a high-performance RPC framework using Protocol Buffers (Protobuf) for serialization.
- **Characteristics**:
  - Supports bidirectional streaming.
  - Efficient binary serialization (smaller and faster than XML or JSON).
  - Language-agnostic with SDKs for many languages.
- **Advantages**:
  - High performance, suitable for microservices and IoT.
  - Strongly typed APIs with Protobuf definitions.
- **Disadvantages**:
  - Requires Protobuf for data definitions, adding complexity.
  - Less human-readable compared to REST (due to binary format).
## 2. Modularity
Modularity is a software design principle that divides a system into smaller parts called modules. Each module is self-contained, with specific functionality, allowing for better organization, reusability, and maintainability. Below is an explanation of modularity and its evolution in Java and related platforms.
### 1. OSGi (Open Service Gateway Initiative)
- **Overview**: OSGi is a framework for developing modular Java applications. It allows components (bundles) to be dynamically discovered, loaded, updated, and removed during runtime.
- **Key Features**:
  - Dynamic service registry for runtime service discovery.
  - Versioning and dependency management for modules.
  - Isolation of module execution to prevent interference.
- **Applications**:
  - IoT systems, enterprise applications, and any application requiring high modularity.

### 2. JBoss 6 -> MSC (Modular Service Container)
- **Overview**: JBoss 6 introduced MSC as a lightweight and modular service container, enabling more efficient deployment and management of Java applications.
- **Advantages**:
  - Faster startup and resource allocation compared to monolithic containers.
  - Provides a dependency injection mechanism for better modularization.
  - Allows dynamic reloading of services without restarting the container.
- **Use Case**:
  - Enterprise-level applications requiring scalable and modular architectures.

### 3. Since Java SE 9+
Java SE 9 introduced the **Java Platform Module System (JPMS)**, also known as **Project Jigsaw**, marking a significant shift in modularity for the Java platform.

#### i. Platform Modularity
- **Definition**: The Java runtime itself is modularized into a set of components.
- **Benefits**:
  - Reduces the size of applications by including only required modules.
  - Improves security by isolating internal APIs.
  - Provides better performance and maintainability.
- **Key Components**:
  - `java.base` module: The foundational module for all Java applications.
  - `java.sql`, `java.xml`, and others for specific functionalities.

#### ii. Application Modularity
- **Definition**: Applications can now define and use their own modules.
- **Advantages**:
  - Enforces strong encapsulation of classes and resources.
  - Simplifies dependency management by declaring module requirements (`module-info.java`).
- **Features**:
  - Readability: Modules explicitly state which other modules they depend on.
  - Accessibility: Controls which parts of the module are accessible to other modules.

---

By understanding and utilizing these modularity features, developers can design more efficient, secure, and maintainable Java applications.

## 3. Reactive Systems -> Reactive Programming

### Overview
Reactive Systems are a set of design principles used to build distributed systems that are more flexible, loosely-coupled, and scalable. The core characteristics of a Reactive System include being responsive, resilient, elastic, and message-driven.

Reactive Programming, on the other hand, is a programming paradigm focused on working with asynchronous data streams and propagating changes automatically through these streams.

### Since Java SE 9
Java SE 9 introduced new tools and APIs to support Reactive Programming. One of the most significant additions is the `java.util.concurrent.Flow` API, which provides a framework for building asynchronous and reactive data pipelines. This API incorporates the **Reactive Streams** standard, making it easier to implement non-blocking, backpressure-aware applications.

#### Key Features Introduced:
1. **`java.util.concurrent.Flow` API**
   - Includes four main interfaces: `Publisher`, `Subscriber`, `Processor`, and `Subscription`.
   - Enables the development of components that process streams of data asynchronously.

2. **Support for Backpressure**
   - Ensures that subscribers can control the rate of data emission from publishers to avoid being overwhelmed.

3. **Interoperability with Existing Reactive Libraries**
   - The `Flow` API serves as a bridge between Java's standard library and popular reactive libraries like Project Reactor and RxJava.

### Advantages of Reactive Programming with Java SE 9+
- **Improved Scalability:** By leveraging non-blocking operations, applications can handle more concurrent users with fewer resources.
- **Simplified Asynchronous Code:** Reactive Programming abstracts complexities like callbacks and thread management.
- **Enhanced Responsiveness:** Enables systems to remain highly responsive even under high load.

### Example Usage:
Below is a simple example of using the `Flow` API:

```java
import java.util.concurrent.Flow;

class SimplePublisher implements Flow.Publisher<String> {
    private final String[] messages = {"Hello", "Reactive", "World"};

    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        subscriber.onSubscribe(new Flow.Subscription() {
            private int index = 0;
            private boolean completed = false;

            @Override
            public void request(long n) {
                for (long i = 0; i < n && index < messages.length; i++) {
                    subscriber.onNext(messages[index++]);
                }
                if (index == messages.length && !completed) {
                    subscriber.onComplete();
                    completed = true;
                }
            }

            @Override
            public void cancel() {
                completed = true;
            }
        });
    }
}

public class ReactiveExample {
    public static void main(String[] args) {
        SimplePublisher publisher = new SimplePublisher();
        publisher.subscribe(new Flow.Subscriber<>() {
            @Override
            public void onSubscribe(Flow.Subscription subscription) {
                System.out.println("Subscribed!");
                subscription.request(3);
            }

            @Override
            public void onNext(String item) {
                System.out.println("Received: " + item);
            }

            @Override
            public void onError(Throwable throwable) {
                System.err.println("Error: " + throwable.getMessage());
            }

            @Override
            public void onComplete() {
                System.out.println("All items received!");
            }
        });
    }
}
```
