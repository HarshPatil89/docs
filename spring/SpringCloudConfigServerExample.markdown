# Spring Cloud Config Server with Multiple Profiles

This example demonstrates setting up a Spring Cloud Config Server with multiple profiles (e.g., dev, prod) using a Git repository as the configuration source.

## Prerequisites
- Java 17+
- Maven or Gradle
- Git repository (local or remote) for configuration storage

## Project Setup

### 1. Create the Config Server Project

#### Maven Dependencies (pom.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>config-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Config Server</name>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2023.0.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

#### Enable Config Server
Create the main application class with `@EnableConfigServer`.

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### 2. Configure the Config Server

#### Application Properties (src/main/resources/application.yml)
```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-repo/config-repo.git
          clone-on-start: true
          default-label: main
  profiles:
    active: native, git
  security:
    user:
      name: config-user
      password: config-password
```

### 3. Setup Git Repository for Configuration

Create a Git repository (e.g., `config-repo`) with the following structure:

```
config-repo/
├── application.yml
├── application-dev.yml
├── application-prod.yml
├── client-service.yml
├── client-service-dev.yml
├── client-service-prod.yml
```

#### Example Configuration Files

**application.yml** (Default properties)
```yaml
app:
  global:
    setting: default-value
```

**application-dev.yml**
```yaml
app:
  global:
    setting: dev-value
  database:
    url: jdbc:mysql://localhost:3306/dev_db
    username: dev_user
    password: dev_pass
```

**application-prod.yml**
```yaml
app:
  global:
    setting: prod-value
  database:
    url: jdbc:mysql://prod-host:3306/prod_db
    username: prod_user
    password: prod_pass
```

**client-service.yml**
```yaml
service:
  name: client-service
  timeout: 5000
```

**client-service-dev.yml**
```yaml
service:
  timeout: 3000
```

**client-service-prod.yml**
```yaml
service:
  timeout: 10000
```

### 4. Create a Client Application

#### Maven Dependencies (pom.xml for Client)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>client-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Client Service</name>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2023.0.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

#### Client Bootstrap Configuration (src/main/resources/bootstrap.yml)
```yaml
spring:
  application:
    name: client-service
  cloud:
    config:
      uri: http://localhost:8888
      username: config-user
      password: config-password
      profile: dev
```

#### Client Controller to Test Configuration
```java
package com.example.clientservice;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ConfigController {
    @Value("${app.global.setting}")
    private String globalSetting;

    @Value("${service.timeout}")
    private int timeout;

    @GetMapping("/config")
    public String getConfig() {
        return "Global Setting: " + globalSetting + ", Timeout: " + timeout;
    }
}
```

### 5. Running the Application

1. **Start the Config Server**:
   - Run the `ConfigServerApplication`.
   - Access configuration at:
     - `http://localhost:8888/client-service/dev` (for dev profile)
     - `http://localhost:8888/client-service/prod` (for prod profile)

2. **Start the Client Application**:
   - Set the active profile in `bootstrap.yml` (e.g., `dev` or `prod`).
   - Run the client application.
   - Access `http://localhost:8080/config` to see the loaded configuration.

### 6. Testing Different Profiles

- Change the `spring.cloud.config.profile` in the client's `bootstrap.yml` to `prod` and restart to load production configurations.
- The Config Server automatically serves the appropriate `application-{profile}.yml` and `client-service-{profile}.yml` based on the profile.

### Notes
- Ensure the Git repository is accessible from the Config Server.
- Use Spring Security for basic authentication (as shown in `application.yml`).
- For production, secure the Git repository and Config Server endpoints properly.
- You can add more profiles (e.g., `test`, `staging`) by creating additional `{application}-{profile}.yml` files in the Git repository.