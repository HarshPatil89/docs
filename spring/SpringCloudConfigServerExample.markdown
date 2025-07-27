Spring Cloud Config Server with Multiple Profiles
This example demonstrates setting up a Spring Cloud Config Server with multiple profiles (e.g., dev, prod) using a Git repository as the configuration source. Below, each configuration and code statement is explained to clarify its purpose and functionality.
Prerequisites

Java 17+: Required for running Spring Boot 3.2.0 and Spring Cloud 2023.0.0, ensuring compatibility with modern Java features.
Maven or Gradle: Build tools to manage dependencies and build the project. Maven is used in this example for simplicity.
Git repository (local or remote): Stores configuration files, allowing centralized management and version control.

Project Setup
1. Create the Config Server Project
Maven Dependencies (pom.xml)
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- Defines the XML schema and namespace for Maven POM file -->
    <modelVersion>4.0.0</modelVersion>
    <!-- Specifies the Maven POM model version -->
    <groupId>com.example</groupId>
    <!-- Unique identifier for the project group, typically the organization -->
    <artifactId>config-server</artifactId>
    <!-- Unique identifier for the project artifact -->
    <version>0.0.1-SNAPSHOT</version>
    <!-- Version of the project, SNAPSHOT indicates a development version -->
    <name>Config Server</name>
    <!-- Human-readable name of the project -->

    <parent>
        <!-- Inherit from Spring Boot's parent POM for dependency and plugin management -->
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
            <artifactId>spring-cloud-config-server</artifactId>
            <!-- Enables Spring Cloud Config Server functionality -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
            <!-- Adds Spring Security for basic authentication to secure config endpoints -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <!-- Provides testing libraries like JUnit and Mockito, scoped to test phase -->
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
                <!-- Indicates this is a BOM (Bill of Materials) for dependency management -->
                <scope>import</scope>
                <!-- Imports dependency versions to ensure compatibility -->
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

Enable Config Server
Create the main application class with @EnableConfigServer.
package com.example.configserver;
// Defines the package for the Config Server application

import org.springframework.boot.SpringApplication;
// Imports Spring Boot's main application runner
import org.springframework.boot.autoconfigure.SpringBootApplication;
// Enables auto-configuration and component scanning
import org.springframework.cloud.config.server.EnableConfigServer;
// Activates the Config Server functionality

@SpringBootApplication
// Marks this class as the main Spring Boot application, enabling auto-configuration
@EnableConfigServer
// Enables the application to act as a Spring Cloud Config Server
public class ConfigServerApplication {
    public static void main(String[] args) {
        // Entry point of the application
        SpringApplication.run(ConfigServerApplication.class, args);
        // Starts the Spring Boot application with the given class and arguments
    }
}

2. Configure the Config Server
Application Properties (src/main/resources/application.yml)
server:
  port: 8888
  # Sets the HTTP port for the Config Server, 8888 is the default for Config Servers

spring:
  application:
    name: config-server
    # Names the application, used in logs and for config retrieval
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-repo/config-repo.git
          # Specifies the Git repository URL where configuration files are stored
          clone-on-start: true
          # Clones the Git repository on server startup for faster access
          default-label: main
          # Sets the default Git branch to 'main' for configuration retrieval
  profiles:
    active: native, git
    # Enables both native (local file) and Git-based configuration sources
  security:
    user:
      name: config-user
      # Defines the username for basic authentication to secure endpoints
      password: config-password
      # Defines the password for basic authentication

3. Setup Git Repository for Configuration
Create a Git repository (e.g., config-repo) with the following structure:
config-repo/
├── application.yml
├── application-dev.yml
├── application-prod.yml
├── client-service.yml
├── client-service-dev.yml
├── client-service-prod.yml


Purpose: The repository stores configuration files for different applications and profiles.
Naming Convention: Files named application.yml apply globally, while {application-name}-{profile}.yml (e.g., client-service-dev.yml) is specific to an application and profile.

Example Configuration Files
application.yml (Default properties)
app:
  global:
    setting: default-value
    # Defines a default global setting applied to all profiles unless overridden

application-dev.yml
app:
  global:
    setting: dev-value
    # Overrides the global setting for the 'dev' profile
  database:
    url: jdbc:mysql://localhost:3306/dev_db
    # Specifies the database URL for the development environment
    username: dev_user
    # Database username for the development environment
    password: dev_pass
    # Database password for the development environment

application-prod.yml
app:
  global:
    setting: prod-value
    # Overrides the global setting for the 'prod' profile
  database:
    url: jdbc:mysql://prod-host:3306/prod_db
    # Specifies the database URL for the production environment
    username: prod_user
    # Database username for the production environment
    password: prod_pass
    # Database password for the production environment

client-service.yml
service:
  name: client-service
  # Defines the service name, applied to all profiles of this application
  timeout: 5000
  # Sets a default timeout value (in milliseconds) for the service

client-service-dev.yml
service:
  timeout: 3000
  # Overrides the timeout value for the 'dev' profile, setting a shorter timeout

client-service-prod.yml
service:
  timeout: 10000
  # Overrides the timeout value for the 'prod' profile, setting a longer timeout

4. Create a Client Application
Maven Dependencies (pom.xml for Client)
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- Defines the XML schema and namespace for the client POM file -->
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
        <!-- Inherit from Spring Boot's parent POM for dependency management -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <!-- Specifies Spring Boot version for consistency -->
        <relativePath/>
        <!-- Empty relativePath to use the parent POM from the repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
            <!-- Enables the client to fetch configurations from the Config Server -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- Adds Spring Web MVC for building RESTful endpoints -->
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
</project>

Client Bootstrap Configuration (src/main/resources/bootstrap.yml)
spring:
  application:
    name: client-service
    # Specifies the application name, matching the config files in the Git repo
  cloud:
    config:
      uri: http://localhost:8888
      # URL of the Config Server to fetch configurations from
      username: config-user
      # Username for authenticating with the Config Server
      password: config-password
      # Password for authenticating with the Config Server
      profile: dev
      # Specifies the active profile (e.g., 'dev') to load profile-specific configs

Client Controller to Test Configuration
package com.example.clientservice;
// Defines the package for the client application

import org.springframework.beans.factory.annotation.Value;
// Imports annotation to inject configuration properties
import org.springframework.web.bind.annotation.GetMapping;
// Imports annotation for mapping HTTP GET requests
import org.springframework.web.bind.annotation.RestController;
// Imports annotation to mark this class as a REST controller

@RestController
// Marks this class as a REST controller, enabling HTTP request handling
public class ConfigController {
    @Value("${app.global.setting}")
    // Injects the value of 'app.global.setting' from the Config Server
    private String globalSetting;

    @Value("${service.timeout}")
    // Injects the value of 'service.timeout' from the Config Server
    private int timeout;

    @GetMapping("/config")
    // Maps HTTP GET requests to '/config' to this method
    public String getConfig() {
        return "Global Setting: " + globalSetting + ", Timeout: " + timeout;
        // Returns a string with the fetched configuration values
    }
}

5. Running the Application

Start the Config Server:

Run the ConfigServerApplication.
Purpose: Starts the Config Server to serve configuration files from the Git repository.
Access configuration at:
http://localhost:8888/client-service/dev: Retrieves configurations for the client-service application with the dev profile.
http://localhost:8888/client-service/prod: Retrieves configurations for the client-service application with the prod profile.




Start the Client Application:

Set the active profile in bootstrap.yml (e.g., dev or prod).
Purpose: Determines which profile-specific configurations are loaded.
Run the client application.
Access http://localhost:8080/config to see the loaded configuration.
Purpose: Verifies that the client correctly fetches and applies the configuration.



6. Testing Different Profiles

Change the spring.cloud.config.profile in the client's bootstrap.yml to prod and restart to load production configurations.
Purpose: Demonstrates how the client switches between profile-specific configurations.


The Config Server automatically serves the appropriate application-{profile}.yml and client-service-{profile}.yml based on the profile.
Purpose: Shows the Config Server’s ability to dynamically serve configurations based on the client’s profile.



Notes

Ensure the Git repository is accessible from the Config Server.
Reason: The Config Server needs to fetch configuration files from the specified Git repository.


Use Spring Security for basic authentication (as shown in application.yml).
Reason: Secures Config Server endpoints to prevent unauthorized access.


For production, secure the Git repository and Config Server endpoints properly.
Reason: Protects sensitive configuration data in a production environment.


You can add more profiles (e.g., test, staging) by creating additional {application}-{profile}.yml files in the Git repository.
Reason: Allows flexibility to support multiple environments with distinct configurations.


