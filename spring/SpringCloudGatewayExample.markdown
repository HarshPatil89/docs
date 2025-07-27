# Spring Cloud Gateway with Load Balancer and Circuit Breaker

This example demonstrates setting up a Spring Cloud Gateway with load balancing (via Spring Cloud LoadBalancer) and circuit breaker (via Resilience4J) to route requests to a client service with multiple profiles (e.g., dev, prod). Each configuration and code statement is explained to clarify its purpose.

## Prerequisites
- **Java 17+**: Required for Spring Boot 3.2.0 and Spring Cloud 2023.0.0 compatibility.
- **Maven**: Used as the build tool to manage dependencies.
- **Service Registry (e.g., Eureka)**: Enables service discovery for load balancing.
- **Client Service Instances**: Multiple instances of a client service to demonstrate load balancing.

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
            <!-- Enables Spring Cloud Gateway functionality for routing and filtering -->
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
            <!-- Enables Eureka client for service discovery and load balancing -->
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

### 2. Configure the Gateway

#### Application Configuration (src/main/resources/application.yml)
```yaml
server:
  port: 8080
  # Sets the HTTP port for the Gateway, default for web applications

spring:
  application:
    name: gateway-service
    # Names the application, used in logs and service registry
  cloud:
    gateway:
      routes:
        - id: client-service-route
          # Unique identifier for the route
          uri: lb://client-service
          # Uses load balancer (lb) to route to 'client-service' registered in Eureka
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
```

### 3. Setup Eureka Server

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
        <!-- Empty relativePath to use the parent POM from the repository -->
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

### 4. Create a Client Application

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

### 5. Gateway Fallback Controller
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
        return "Service is temporarily unavailable. Please try again later.";
        // Provides a fallback response when the client service is down
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
   - **Purpose**: Starts the gateway to route requests and apply load balancing and circuit breaker logic.
   - Access the client service via the gateway at `http://localhost:8080/api/message`.
   - **Behavior**: The gateway load balances between client instances and uses the circuit breaker if the service is unavailable.

### 7. Testing Load Balancing and Circuit Breaker

- **Load Balancing**:
  - Start multiple instances of the client service (e.g., two with `dev` profile, one with `prod` profile).
  - Send repeated requests to `http://localhost:8080/api/message`.
  - **Expected Behavior**: The gateway distributes requests across available instances, alternating responses (e.g., "Development Environment" or "Production Environment").

- **Circuit Breaker**:
  - Stop all client service instances to simulate failure.
  - Send a request to `http://localhost:8080/api/message`.
  - **Expected Behavior**: The circuit breaker trips, redirecting to the `/fallback` endpoint, returning "Service is temporarily unavailable."

### Notes
- Ensure the Eureka server is running before starting the gateway and client services.
  - **Reason**: The gateway and client rely on Eureka for service discovery.
- The circuit breaker configuration can be fine-tuned in `application.yml` (e.g., failure thresholds, timeout durations).
  - **Reason**: Allows customization based on application needs.
- For production, secure the gateway endpoints and Eureka server with authentication.
  - **Reason**: Protects sensitive routing and service discovery operations.
- Add more profiles by creating additional `application-{profile}.yml` files in the client service.
  - **Reason**: Supports multiple environments with distinct configurations.