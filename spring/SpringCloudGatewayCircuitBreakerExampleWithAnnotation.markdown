# Spring Cloud Gateway with Circuit Breaker Configuration and Annotations

This example demonstrates a Spring Cloud Gateway with a circuit breaker using Resilience4J, including detailed configuration in `application.yml` and programmatic usage with annotations. It routes requests to a client service, with load balancing via Spring Cloud LoadBalancer and service discovery via Eureka. Each configuration and code statement is explained for clarity.

## Prerequisites
- **Java 17+**: Ensures compatibility with Spring Boot 3.2.0 and Spring Cloud 2023.0.0.
- **Maven**: Manages dependencies and builds the project.
- **Eureka Server**: Enables service discovery for load balancing.
- **Client Service Instances**: Multiple instances to demonstrate load balancing and circuit breaker behavior.

## Project Setup

### 1. Create the Gateway Project

#### Maven Dependencies (pom.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- Defines the XML schema and namespace for the Maven POM file -->
    <modelVersion>4.0.0</modelVersion>
    <!-- Specifies the Maven POM model version -->
    <groupId>com.example</groupId>
    <!-- Unique identifier for the project group -->
    <artifactId>gateway-service</artifactId>
    <!-- Unique identifier for the gateway artifact -->
    <version>0.0.1-SNAPSHOT</version>
    <!-- Version of the project, SNAPSHOT indicates development -->
    <name>Gateway Service</name>
    <!-- Human-readable name of the gateway application -->

    <parent>
        <!-- Inherit from Spring Boot's parent POM for dependency management -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <!-- Specifies Spring Boot version for consistent dependency versions -->
        <relativePath/>
        <!-- Empty relativePath to use the parent POM from the Maven repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
            <!-- Enables Spring Cloud Gateway for routing and filtering -->
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
            <!-- Adds Resilience4J for circuit breaker functionality -->
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
            <!-- Enables Spring Cloud LoadBalancer for client-side load balancing -->
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <!-- Enables Eureka client for service discovery -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <!-- Adds Actuator for health checks and monitoring -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <!-- Provides testing libraries like JUnit, scoped to test phase -->
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-spring-boot3</artifactId>
            <version>2.2.0</version>
            <!-- Adds Resilience4J annotations for programmatic circuit breaker control -->
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2023.0.0</version>
                <!-- Specifies Spring Cloud version compatible with Spring Boot 3.2.0 -->
                <type>pom</type>
                <!-- Indicates this is a BOM for dependency management -->
                <scope>import</scope>
                <!-- Imports dependency versions for compatibility -->
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!-- Enables Spring Boot's build features, like executable JAR creation -->
            </plugin>
        </plugins>
    </build>
</project>
```

### 2. Configure the Gateway with Circuit Breaker

#### Application Configuration (src/main/resources/application.yml)
```yaml
server:
  port: 8080
  # Sets the HTTP port for the Gateway, default for web applications

spring:
  application:
    name: gateway-service
    # Names the application, used in logs and Eureka registry
  cloud:
    gateway:
      routes:
        - id: client-service-route
          # Unique identifier for the route
          uri: lb://client-service
          # Uses load balancer to route to 'client-service' registered in Eureka
          predicates:
            - Path=/api/**
            # Routes requests with paths starting with '/api/' to the client service
          filters:
            - name: CircuitBreaker
              args:
                name: clientServiceCircuitBreaker
                # Unique name for the circuit breaker instance
                fallbackUri: forward:/fallback
                # Redirects to '/fallback' endpoint if the circuit breaker trips
    loadbalancer:
      ribbon:
        enabled: false
        # Disables Ribbon in favor of Spring Cloud LoadBalancer
  profiles:
    active: dev
    # Sets the active profile to 'dev' for profile-specific configurations

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
      # Specifies the Eureka server URL for service discovery

resilience4j:
  circuitbreaker:
    instances:
      clientServiceCircuitBreaker:
        slidingWindowSize: 10
        # Number of calls to evaluate for circuit breaker state (e.g., 10 calls)
        slidingWindowType: COUNT_BASED
        # Uses a count-based sliding window for tracking calls
        failureRateThreshold: 50
        # Opens the circuit if 50% of calls fail in the sliding window
        waitDurationInOpenState: 5000
        # Time (in ms) the circuit stays open before transitioning to half-open
        permittedNumberOfCallsInHalfOpenState: 3
        # Number of calls allowed in half-open state to test service recovery
        automaticTransitionFromOpenToHalfOpenEnabled: true
        # Automatically transitions from open to half-open after wait duration
        registerHealthIndicator: true
        # Registers circuit breaker health with Spring Boot Actuator
        slowCallRateThreshold: 50
        # Marks calls as slow if 50% exceed the slow call duration
        slowCallDurationThreshold: 1000
        # Threshold (in ms) for considering a call as slow
```

### 3. Gateway Application with Annotation-Based Circuit Breaker

#### Gateway Application
```java
package com.example.gatewayservice;
// Defines the package for the gateway application

import org.springframework.boot.SpringApplication;
// Imports Spring Boot's main application runner
import org.springframework.boot.autoconfigure.SpringBootApplication;
// Enables auto-configuration and component scanning

@SpringBootApplication
// Marks this class as the main Spring Boot application
public class GatewayServiceApplication {
    public static void main(String[] args) {
        // Entry point of the application
        SpringApplication.run(GatewayServiceApplication.class, args);
        // Starts the Spring Boot application
    }
}
```

#### Circuit Breaker Service with Annotations
```java
package com.example.gatewayservice;
// Defines the package for the gateway service

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
// Imports Resilience4J annotation for circuit breaker
import org.springframework.stereotype.Service;
// Imports annotation to mark this class as a Spring service
import org.springframework.web.client.RestTemplate;
// Imports RestTemplate for making HTTP requests
import java.util.function.Supplier;
// Imports Supplier for functional programming with circuit breaker

@Service
// Marks this class as a Spring service for dependency injection
public class ClientServiceCaller {

    private final RestTemplate restTemplate;

    public ClientServiceCaller(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        // Injects RestTemplate for making HTTP calls
    }

    @CircuitBreaker(name = "clientServiceCircuitBreaker", fallbackMethod = "fallback")
    // Applies circuit breaker with the same name as in application.yml
    // Specifies fallback method to invoke if the circuit breaker trips
    public String callClientService(String path) {
        // Method to call the client service via RestTemplate
        return restTemplate.getForObject("http://client-service" + path, String.class);
        // Makes an HTTP GET request to the client service
    }

    private String fallback(Throwable t) {
        // Fallback method invoked when the circuit breaker trips
        return "Annotation-based fallback: Service is unavailable due to " + t.getMessage();
        // Returns a custom fallback message with the exception details
    }
}
```

#### Gateway Fallback Controller
```java
package com.example.gatewayservice;
// Defines the package for the gateway application

import org.springframework.web.bind.annotation.GetMapping;
// Imports annotation for mapping HTTP GET requests
import org.springframework.web.bind.annotation.RestController;
// Imports annotation to mark this class as a REST controller

@RestController
// Marks this class as a REST controller for handling fallback requests
public class FallbackController {
    @GetMapping("/fallback")
    // Maps HTTP GET requests to '/fallback' when the circuit breaker trips
    public String fallback() {
        return "Configuration-based fallback: Service is temporarily unavailable.";
        // Provides a fallback response for the gateway route
    }
}
```

#### RestTemplate Configuration
```java
package com.example.gatewayservice;
// Defines the package for the gateway application

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
// Imports annotation to enable load balancing for RestTemplate
import org.springframework.context.annotation.Bean;
// Imports annotation to define Spring beans
import org.springframework.context.annotation.Configuration;
// Imports annotation to mark this class as a configuration class
import org.springframework.web.client.RestTemplate;
// Imports RestTemplate for HTTP requests

@Configuration
// Marks this class as a Spring configuration class
public class GatewayConfig {
    @Bean
    @LoadBalanced
    // Creates a load-balanced RestTemplate bean for service discovery
    public RestTemplate restTemplate() {
        return new RestTemplate();
        // Returns a new RestTemplate instance
    }
}
```

### 4. Setup Eureka Server

#### Maven Dependencies for Eureka Server (pom.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- Defines the XML schema and namespace for the Eureka server POM -->
    <modelVersion>4.0.0</modelVersion>
    <!-- Specifies the Maven POM model version -->
    <groupId>com.example</groupId>
    <!-- Unique identifier for the project group -->
    <artifactId>eureka-server</artifactId>
    <!-- Unique identifier for the Eureka server artifact -->
    <version>0.0.1-SNAPSHOT</version>
    <!-- Version of the project, SNAPSHOT indicates development -->
    <name>Eureka Server</name>
    <!-- Human-readable name of the Eureka server -->

    <parent>
        <!-- Inherit from Spring Boot's parent POM -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <!-- Specifies Spring Boot version -->
        <relativePath/>
        <!-- Empty relativePath to use the parent POM -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
            <!-- Enables Eureka server functionality for service discovery -->
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2023.0.0</version>
                <!-- Specifies Spring Cloud version -->
                <type>pom</type>
                <!-- Indicates this is a BOM -->
                <scope>import</scope>
                <!-- Imports dependency versions -->
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

#### Eureka Server Configuration (src/main/resources/application.yml)
```yaml
server:
  port: 8761
  # Sets the HTTP port for the Eureka server, default for Eureka

eureka:
  client:
    register-with-eureka: false
    # Prevents the Eureka server from registering itself as a client
    fetch-registry: false
    # Prevents the Eureka server from fetching its own registry
  server:
    enable-self-preservation: false
    # Disables self-preservation for testing, allowing faster deregistration
```

#### Eureka Server Application
```java
package com.example.eurekaserver;
// Defines the package for the Eureka server

import org.springframework.boot.SpringApplication;
// Imports Spring Boot's main application runner
import org.springframework.boot.autoconfigure.SpringBootApplication;
// Enables auto-configuration and component scanning
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
// Activates the Eureka server functionality

@SpringBootApplication
// Marks this class as the main Spring Boot application
@EnableEurekaServer
// Enables the application to act as a Eureka server
public class EurekaServerApplication {
    public static void main(String[] args) {
        // Entry point of the application
        SpringApplication.run(EurekaServerApplication.class, args);
        // Starts the Eureka server
    }
}
```

### 5. Create a Client Application

#### Maven Dependencies (pom.xml for Client)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- Defines the XML schema and namespace for the client POM -->
    <modelVersion>4.0.0</modelVersion>
    <!-- Specifies the Maven POM model version -->
    <groupId>com.example</groupId>
    <!-- Unique identifier for the project group -->
    <artifactId>client-service</artifactId>
    <!-- Unique identifier for the client application artifact -->
    <version>0.0.1-SNAPSHOT</version>
    <!-- Version of the project, SNAPSHOT indicates development -->
    <name>Client Service</name>
    <!-- Human-readable name of the client application -->

    <parent>
        <!-- Inherit from Spring Boot's parent POM -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <!-- Specifies Spring Boot version -->
        <relativePath/>
        <!-- Empty relativePath to use the parent POM -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <!-- Enables Eureka client for service discovery -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- Adds Spring Web MVC for RESTful endpoints -->
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2023.0.0</version>
                <!-- Specifies Spring Cloud version -->
                <type>pom</type>
                <!-- Indicates this is a BOM -->
                <scope>import</scope>
                <!-- Imports dependency versions -->
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

#### Client Configuration (src/main/resources/application.yml)
```yaml
spring:
  application:
    name: client-service
    # Names the application, used in Eureka and for routing
  profiles:
    active: dev
    # Sets the active profile to 'dev' for profile-specific behavior

server:
  port: 0
  # Sets port to 0 to allow Spring to assign a random port for multiple instances

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
      # Specifies the Eureka server URL for registration
  instance:
    instance-id: ${spring.application.name}:${random.uuid}
    # Creates a unique instance ID for each client instance
```

#### Client Configuration for Profiles
**application-dev.yml**
```yaml
service:
  message: Development Environment
  # Defines a message specific to the 'dev' profile
```

**application-prod.yml**
```yaml
service:
  message: Production Environment
  # Defines a message specific to the 'prod' profile
```

#### Client Controller
```java
package com.example.clientservice;
// Defines the package for the client application

import org.springframework.beans.factory.annotation.Value;
// Imports annotation to inject configuration properties
import org.springframework.web.bind.annotation.GetMapping;
// Imports annotation for mapping HTTP GET requests
import org.springframework.web.bind.annotation.RestController;
// Imports annotation to mark this class as a REST controller

@RestController
// Marks this class as a REST controller for handling HTTP requests
public class ClientController {
    @Value("${service.message}")
    // Injects the value of 'service.message' from the active profile
    private String message;

    @GetMapping("/message")
    // Maps HTTP GET requests to '/message' to this method
    public String getMessage() {
        return message;
        // Returns the profile-specific message
    }
}
```

### 6. Running the Application

1. **Start the Eureka Server**:
   - Run the `EurekaServerApplication`.
   - **Purpose**: Starts the service registry for client and gateway registration.
   - Access at `http://localhost:8761` to view registered services.

2. **Start the Client Service**:
   - Run multiple instances with different profiles (e.g., `dev` and `prod`).
   - **Command for Dev**: `mvn spring-boot:run -Dspring-boot.run.profiles=dev`
   - **Command for Prod**: `mvn spring-boot:run -Dspring-boot.run.profiles=prod`
   - **Purpose**: Simulates multiple service instances for load balancing.

3. **Start the Gateway**:
   - Run the `GatewayServiceApplication`.
   - **Purpose**: Starts the gateway to route requests with circuit breaker logic.
   - Access the client service via:
     - `http://localhost:8080/api/message` (configuration-based circuit breaker).
     - `http://localhost:8080/call-client` (annotation-based circuit breaker, see below).

### 7. Testing Circuit Breaker Configuration and Annotations

#### Configuration-Based Circuit Breaker
- Send requests to `http://localhost:8080/api/message`.
- **Behavior**: The gateway routes requests to `client-service` instances. If 50% of the last 10 calls fail (per `slidingWindowSize` and `failureRateThreshold`), the circuit breaker opens, redirecting to `/fallback` (returns "Configuration-based fallback...").
- **Test Failure**: Stop all client service instances and send requests. The circuit breaker trips after 5 failures, invoking the fallback.

#### Annotation-Based Circuit Breaker
- Add a controller to test the annotation-based circuit breaker:
```java
package com.example.gatewayservice;
// Defines the package for the gateway application

import org.springframework.beans.factory.annotation.Autowired;
// Imports annotation for dependency injection
import org.springframework.web.bind.annotation.GetMapping;
// Imports annotation for mapping HTTP GET requests
import org.springframework.web.bind.annotation.RestController;
// Imports annotation to mark this class as a REST controller

@RestController
// Marks this class as a REST controller
public class TestController {
    @Autowired
    // Injects the ClientServiceCaller service
    private ClientServiceCaller clientServiceCaller;

    @GetMapping("/call-client")
    // Maps HTTP GET requests to '/call-client'
    public String callClient() {
        return clientServiceCaller.callClientService("/message");
        // Calls the client service through the circuit breaker
    }
}
```
- Send requests to `http://localhost:8080/call-client`.
- **Behavior**: The `@CircuitBreaker` annotation applies the same `clientServiceCircuitBreaker` configuration. If the circuit breaker trips, it invokes the `fallback` method in `ClientServiceCaller`.
- **Test Failure**: Stop client services and send requests. The circuit breaker trips, returning the annotation-based fallback message.

### 8. Monitoring Circuit Breaker
- Access Actuator endpoint: `http://localhost:8080/actuator/health`.
- **Purpose**: Checks the circuit breaker’s health (e.g., `CLOSED`, `OPEN`, `HALF_OPEN`) due to `registerHealthIndicator: true`.

### Notes
- The circuit breaker configuration in `application.yml` applies to the gateway route, while the `@CircuitBreaker` annotation is used for programmatic control in the `ClientServiceCaller`.
  - **Reason**: Provides flexibility for both declarative (YAML) and programmatic (annotations) circuit breaker usage.
- Fine-tune `resilience4j.circuitbreaker.instances` parameters based on your application’s needs (e.g., adjust `failureRateThreshold` for stricter failure detection).
  - **Reason**: Allows customization for different service reliability requirements.
- Ensure Eureka server is running before gateway and client services.
  - **Reason**: Service discovery is required for load balancing and routing.
- For production, secure endpoints and consider additional Resilience4J features like retry or rate limiting.
  - **Reason**: Enhances reliability and security in production environments.