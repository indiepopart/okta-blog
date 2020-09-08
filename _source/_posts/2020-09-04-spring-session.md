---
layout: blog_post
title: "Session Sharing in Spring-Boot Application with Multiple Nodes"
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

# About Session Management
(introduction)

Prerequisites:
- Java 8+
- Docker
- Docker Compose

# Session Persistence

Let's first create a web application with Okta authentication, and run three nodes with HAProxy loadbalancing, using Docker Compose.


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

Run the Okta Maven Plugin to Register a new account and configure your spring application for authentication using Okta:
```shell
./mvnw com.okta:okta-maven-plugin:spring-boot
```

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
        return "Hello " + oidcUser.getFullName() + ", you are connected to " + serverUsed + ", with sessionId " + sessionId;
    }
}
```

Run the application with:

```shell
./mvnw spring-boot:run
```

Go to http://localhost:8080 and you should be redirected to the Okta login page.

{% img blog/spring-session/okta-login.png alt:"Okta login form" width:"500" %}{: .center-image }

Now let's configure three Docker containers, one for each application node, and an HAProxy container.

In the project root folder, create a file `docker/docker-compose.yml`, with the following content:

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

Create a `Dockerfile` for the HAProxy container, at `docker/Dockerfile-hapoxy`, and add the following:

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

- **balance roundrobin**: the servers will be loadbalanced in a roundrobin fashion
- **cookie SERVERUSED**: the server responding the request will be indicated in a cookie named SERVERUSED, and the http session will stick to that server
- **option redispatch**: if the request fails, it will be redispatched to a different server


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
docker-compose up
```

In a browser, go to http://localhost/greeting. After the Okta login, inspect the request cookie SERVERUSED. An example value is:

```
Cookie: SERVERUSED=webapp3; JSESSIONID=5AF5669EA145CC86BBB08CE09FF6E505
```
Shut down the current node with the following docker command:
```shell
docker stop docker_webapp3_1
```

Check the SERVERUSED cookie to verify that HAProxy redispatched the request to a different node, and the sessionId has changed, meaning the old session was lost.


# Session Sharing with Spring Session
