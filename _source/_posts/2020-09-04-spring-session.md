---
layout: blog_post
title: "Multi-Node Session Sharing in Spring Boot with Spring Session"
author:
by: contractor
communities: [devops,java]
description: ""
tags: [java, session, spring-boot, okta, spring-session]
tweets:
- ""
- ""
- ""
image:
type: awareness
---

Session management in multi-node applications presents multiple challenges. When the architecture includes a load balancer, client requests might be routed to different servers each time, and the HTTP session might be lost. In this tutorial, we will walk you through the configuration of session sharing in a multi-node Spring Boot application.

Prerequisites:
- [Java 8+](https://adoptopenjdk.net/)
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

# Session Persistence

Session Persistence is a technique to stick a client to a single server, using application layer information, for example, a cookie.
In this tutorial, we will implement session persistence with the help of [HAProxy](http://cbonte.github.io/haproxy-dconv/2.3/intro.html#3), a reliable, high performance, TCP/HTTP load balancer.

{% img blog/spring-session/haproxy.png alt:"HAProxy logo" width:"300" %}{: .center-image }

So let's first create a web application with Okta authentication, and run three nodes with HAProxy load balancing, using Docker Compose.
Create a Maven project using Spring Initializr API.

```shell
curl https://start.spring.io/starter.zip \
-d dependencies=web,okta \
-d groupId=com.okta.developer \
-d artifactId=webapp \
-d name="Web Application" \
-d description="Demo Web Application" \
-d packageName=com.okta.developer.webapp \
-d javaVersion=11 \
-o web-app.zip
```

Unzip the project:
```shell
unzip web-app.zip -d web-app
cd web-app
```

Run the [Okta Maven Plugin](https://github.com/oktadeveloper/okta-maven-plugin) to register a new account and configure your Spring application for authentication using Okta:
```shell
./mvnw com.okta:okta-maven-plugin:spring-boot
```
It will prompt you for the required information and set up a new OIDC application for you.

Create a `GreetingController`:

```java
package com.okta.developer.webapp.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.context.request.RequestContextHolder;

import java.net.InetAddress;

@Controller
public class GreetingController {

    private static final Logger logger = LoggerFactory.getLogger(GreetingController.class);


    @GetMapping(value = "/greeting")
    @ResponseBody
    public String getGreeting(@AuthenticationPrincipal OidcUser oidcUser) {
        String serverUsed = "unknown";
        try {
            InetAddress host = InetAddress.getLocalHost();
            serverUsed = host.getHostName();
        } catch (Exception e){
            logger.error("Could not get hostname", e);
        }
        String sessionId = RequestContextHolder.currentRequestAttributes().getSessionId();
        logger.info("Request responded by " + serverUsed);
        return "Hello " + oidcUser.getFullName() + ", your server is " + serverUsed + ", with sessionId " + sessionId;
    }
}
```

Run the application with:

```shell
./mvnw spring-boot:run
```

Go to http://localhost:8080 and you should be redirected to the Okta login page.

{% img blog/spring-session/okta-login.png alt:"Okta login form" width:"500" %}{: .center-image }

Now let's configure three Docker containers, one for each application node, and an HAProxy container. In the project root folder, create a file `docker/docker-compose.yml`, with the following content:

```yml
version: '3.1'
services:
  webapp1:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp1
    ports:
      - 8081:8080
  webapp2:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp2
    ports:
      - 8082:8080
  webapp3:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp3
    ports:
      - 8083:8080
  haproxy:
    build:
      context: .
      dockerfile: Dockerfile-haproxy
    image: my-haproxy
    ports:
      - 80:80
    depends_on:
      - "webapp1"
      - "webapp2"
      - "webapp3"
```

Create the file `docker/.env` with the following content:
```shell
OKTA_OAUTH2_ISSUER={yourOrgUrl}/oauth2/default
OKTA_OAUTH2_CLIENT_ID={clientId}
OKTA_OAUTH2_CLIENT_SECRET={clientSecret}
```
You can find the **orgUrl**, **clientId** and **clientSecret** in the `src/main/resources/application.properties`, after running the okta-maven-plugin.

Create a `Dockerfile` for the HAProxy container, at `docker/Dockerfile-haproxy`, and add the following:

```
FROM haproxy:2.2
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
```

Create the configuration file for the HAProxy instance at `docker/haproxy.cfg`:

```
global
    debug
    daemon
    maxconn 2000

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    cookie SERVERUSED insert indirect nocache
    option httpchk /
    option redispatch
    default-server check
    server webapp1 webapp1:8080 cookie webapp1
    server webapp2 webapp2:8080 cookie webapp2
    server webapp3 webapp3:8080 cookie webapp3
```
We are not going to dive deep into how to configure HAProxy, but notice in the `backend servers` section, we are using the following options:

- `balance roundrobin` sets round-robin as the load balancing strategy
- `cookie SERVERUSED` adds a cookie SERVERUSED to the response, indicating the server responding to the request. The client requests will stick to that server.
- `option redispatch` makes the request to be re-dispatched to a different server, if the current server fails


Edit the `pom.xml` to add the [Jib Maven Plugin](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin) to containerize the `webapp`.

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>2.5.2</version>
  <configuration>
    <to>
      <image>webapp</image>
    </to>
  </configuration>
</plugin>

```

Build the `webapp` container image with the following maven command:
```shell
./mvnw compile jib:dockerBuild
```

Start all the services with docker-compose:

```shell
cd docker
docker-compose up
```

HAProxy will be ready after you see the following lines in the logs:

```
haproxy_1  | [WARNING] 253/130140 (6) : Server servers/webapp2 is UP, reason: Layer7 check passed, code: 302, check duration: 5ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_1  | [WARNING] 253/130141 (6) : Server servers/webapp3 is UP, reason: Layer7 check passed, code: 302, check duration: 4ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_1  | [WARNING] 253/130143 (6) : Server servers/webapp1 is UP, reason: Layer7 check passed, code: 302, check duration: 7ms. 3 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

In a browser, go to http://localhost/greeting. After the Okta login, inspect the request cookie SERVERUSED. An example value is:

```
Cookie: SERVERUSED=webapp3; JSESSIONID=5AF5669EA145CC86BBB08CE09FF6E505
```
Shut down the current node with the following docker command:
```shell
docker stop docker_webapp3_1
```

Check the SERVERUSED cookie to verify that HAProxy re-dispatched the request to a different node, and the sessionId has changed, meaning the old session was lost.


# Session Sharing with Spring Session

Storing sessions in an individual node can affect scalability. When scaling up, active sessions will remain in the original nodes and traffic will not be spread equally among nodes. Also, when a node fails, the session in that node is lost. With Session Sharing, the user session lives in a shared data storage that all server nodes can access.

Then, for a transparent failover, with the `redispatch` option in HAProxy, let's add session sharing between nodes with Spring Session. For this tutorial, we are using MySQL for the session storage.

First, add the following dependencies to the `pom.xml`:

```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-core</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>com.zaxxer</groupId>
  <artifactId>HikariCP</artifactId>
  <version>3.2.0</version>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>test</scope>
</dependency>
```

Rename `src/main/resources/aplication.properties` to `application.yml`, and add the following content:

```yml
spring:
  session:
    jdbc:
      initialize-schema: always
  datasource:
    url: jdbc:mysql://localhost:3306/webapp
    username: root
    password: example
    driverClassName: com.mysql.cj.jdbc.Driver
    hikari:
      initializationFailTimeout: 0

logging:
  level:
    org.springframework: INFO
    com.zaxxer.hikari: DEBUG
```

We are using HikariCP for the database connection pooling, and the option `initializationFailTimeout` is set to 0, meaning if a connection cannot be obtained, the pool will start anyways.

For this example, we are also instructing Spring Session to always create the schema with the option `spring.session.jdbc.initialize-schema=always`.

Add test properties, create a file `src/test/resources/application-test.yml` with the following content:

```yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password: passord
    driverClassName: org.h2.Driver
```

Modify the `WebApplicationTests` and add the `@ActiveProfiles` annotation:

```java
package com.okta.developer.webapp;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class WebApplicationTests {

    @Test
    void contextLoads() {
    }
}
```

Modify `docker/docker-compose.yml` to add the database container and the admin application to inspect the session tables. The final `yml` should look like the following:

```yml
version: '3.1'
services:
  webapp1:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp1
    ports:
      - 8081:8080
    depends_on:
      - "db"
  webapp2:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp2
    ports:
      - 8082:8080
    depends_on:
      - "db"
  webapp3:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/webapp
      - OKTA_OAUTH2_ISSUER=${OKTA_OAUTH2_ISSUER}
      - OKTA_OAUTH2_CLIENT_ID=${OKTA_OAUTH2_CLIENT_ID}
      - OKTA_OAUTH2_CLIENT_SECRET=${OKTA_OAUTH2_CLIENT_SECRET}
    image: webapp
    hostname: webapp3
    ports:
      - 8083:8080
    depends_on:
      - "db"

  db:
    image: mysql
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: example
      MYSQL_DATABASE: webapp
    ports:
      - 3306:3306

  adminer:
    image: adminer
    restart: always
    ports:
      - 8090:8080

  haproxy:
    build:
      context: .
      dockerfile: Dockerfile-haproxy
    image: my-haproxy
    ports:
      - 80:80
    depends_on:
      - "webapp1"
      - "webapp2"
      - "webapp3"
```

Delete the previous containers and previous `webapp` docker image with the following commands:

 ```shell
 docker-compose down
 docker rmi webapp
 ```

 In the root folder of the project, rebuild the webapp docker image with maven:

```shell
./mvnw compile jib:dockerBuild
```
Start all the services again, and repeat the re-dispatch test. The session should be maintained after changing the node.
You can inspect the session data in the adminer UI at http://localhost:8090. Login with `root` and `MYSQL_ROOT_PASSWORD` set in the `docker-compose.yml`


{% img blog/spring-session/adminer-3.png alt:"Spring Session Table " width:"1000" %}{: .center-image }

# Learn More

I hope you enjoyed this tutorial and could see the advantages of the session sharing technique for multi-node applications. Know that there are multiple options for session storage, we selected a database because of the ease of setup, but it might slow down your application. To learn more about session management, check out the following links:

- [Spring Session](https://docs.spring.io/spring-session/docs/2.3.1.RELEASE/reference/html5/index.html)
- [What's New with OAuth and OpenID Connect?](https://developer.okta.com/blog/2020/04/09/whats-new-with-oauth-and-oidc)
- [Build Single Sign-on in Java](https://developer.okta.com/blog/2020/01/29/java-single-sign-on)
- [HAProxy](http://cbonte.github.io/haproxy-dconv/2.3/intro.html#3)
