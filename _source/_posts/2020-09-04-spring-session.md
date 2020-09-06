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








# Session Sharing with Spring Session
