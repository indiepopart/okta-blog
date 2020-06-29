---
layout: blog_post
title: "Spring Cloud Gateway OAuth2 Patterns"
author: jimena-garbarino
by: contractor
communities: [java]
description: ""
tags: [spring gateway, oidc, java, spring, spring boot, spring cloud gateway]
tweets:
- ""
- ""
- ""
image:
type: awareness|conversion
---

Spring Cloud Gateway is the Reactive API Gateway of the Spring Ecosystem, built on Spring Boot, WebFlux and Project Reactor. Its job is to proxy and route requests to services, and provide cross cutting concerns such as security, monitoring and resilience. As Reactive models gain popularity, there is a chance that your microservices architecture becomes a mix of Spring MVC blocking applications and Spring WebFlux non-blocking applications.

In this tutorial you will use Spring Cloud Gateway for routing to traditional Servlet API microservices, and you will learn the required configuration for three common OAuth2 patterns, using Okta as authorization server:

- Spring Cloud Gateway OpenID Connect Login
- Cart Microservice Authorization by Token Relay
- Pricing Microservice Client Credentials Grant

**Prerequisites**: [Java 8](https://adoptopenjdk.net/)+


## Pattern 1: Authentication with Authorization Code Flow



{% img blog/spring-gateway/authorization-code-flow.png alt:"Okta OAuth Code Flow" width:"600" %}{: .center-image }

Create a base folder for all the projects:

```shell
mkdir oauth2-patterns
cd oauth2-patterns
```

For testing this OAuth2 pattern, let's create the API Gateway with service discovery. With Spring Initializr, create an Eureka server:

```
curl https://start.spring.io/starter.zip -d dependencies=cloud-eureka-server \
-d groupId=com.okta.developer \
-d artifactId=discovery-service  \
-d name="Eureka Service" \
-d description="Discovery service" \
-d packageName=com.okta.developer.discovery \
-d javaVersion=11 \
-o eureka.zip
```

Unzip the file:
```
unzip eureka.zip -d eureka
cd eureka
```
Edit the `pom.xml` and add the following dependency:
```xml
<dependency>
  <groupId>org.glassfish.jaxb</groupId>
  <artifactId>jaxb-runtime</artifactId>
</dependency>
```
Edit `EurekaServiceApplication` to add `@EnableEurekaServer` annotation:

```java
package com.okta.developer.discovery;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServiceApplication.class, args);
	}

}
```


Rename `src/main/resources/application.properties` to `application.yml` and add the following content:

```yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

Start the service:

```shell
./mvnw spring-boot:run
```

Go to http://localhost:8761 and you should see the Eureka home.
Now let's create an API Gateway with Spring Cloud Gateway, using Spring Initializr again.

```shell
curl https://start.spring.io/starter.zip \
-d dependencies=cloud-eureka,cloud-gateway,webflux,okta,security,thymeleaf \
-d groupId=com.okta.developer \
-d artifactId=api-gateway  \
-d name="Spring Cloud Gateway Application" \
-d description="Demo project of a Spring Cloud Gateway application and OAuth flows" \
-d packageName=com.okta.developer.gateway \
-d javaVersion=11 \
-o api-gateway.zip
```
Unzip the project:

```
unzip api-gateway.zip
cd api-gateway
```
Edit `SpringCloudGatewayApplication` to add `@EnableEurekaClient` annotation.

```java
package com.okta.developer.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
public class SpringCloudGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudGatewayApplication.class, args);
	}
}
```

Add a package `controller` and the controller `src/main/java/com.okta.developer.gateway.controller.GreetingController`. The controller will allow to test the login without having configured any routes yet.

```java
package com.okta.developer.gateway.controller;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClient;
import org.springframework.security.oauth2.client.annotation.RegisteredOAuth2AuthorizedClient;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class GreetingController {


    @RequestMapping("/greeting")
    public String greeting(@AuthenticationPrincipal OidcUser oidcUser, Model model,
                           @RegisteredOAuth2AuthorizedClient("okta") OAuth2AuthorizedClient client) {
        model.addAttribute("username", oidcUser.getEmail());
        model.addAttribute("idToken", oidcUser.getIdToken());
        model.addAttribute("accessToken", client.getAccessToken());

        return "greeting";
    }
}
```

Add a greeting template `src/main/resources/templates/greeting.html`:

```html
<!DOCTYPE HTML>
<html xmlns:th="https://www.thymeleaf.org">
<head>
    <title>Greeting</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>

<h1><p th:text="'Hello, ' + ${username} + '!'" /></h1>
<p th:text="'idToken: ' + ${idToken.tokenValue}" /><br/>
<p th:text="'accessToken: ' + ${accessToken.tokenValue}" /><br>
</body>
</html>
```

Rename `src/main/resources/application.properties` to `application.yml` and add the following properties:

```yml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    reactor.netty: DEBUG
```

Add `src/main/java/com.okta.developer.gateway.OktaOAuth2WebSecurity` to make the API Gateway a resource server with login enabled:

```java
package com.okta.developer.gateway;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class OktaOAuth2WebSecurity {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http)
    {
         http.authorizeExchange()
                .anyExchange()
                .authenticated()
                .and().oauth2Login()
                .and().csrf().disable()
                .oauth2ResourceServer()
                .jwt();
        return http.build();
    }
}
```
For this example, csrf is disabled.

Now we need to create an authorization client in Okta. Log in to your Okta Developer account (or [sign up](https://developer.okta.com/signup/) if you don't have an account).

1. From the **Applications** page, choose **Add Application**.
2. On the Create New Application page, select **Web**.
3. Name your app _Api Gateway_, add `http://localhost:8080/login/oauth2/code/okta` as a Login redirect URI, select **Refresh Token** (in addition to **Authorization Code**), and click **Done**.

Copy the issuer (found under **API** > **Authorization Servers**), client ID, and client secret.

Start the gateway with:
```shell
OKTA_OAUTH2_ISSUER={yourOktaIssuer} \
OKTA_OAUTH2_CLIENT_ID={yourOktaClientId} \
OKTA_OAUTH2_CLIENT_SECRET={yourOktaClientSecret} \
./mvnw spring-boot:run
```

Go to http://localhost:8080/greeting. The gateway will redirect to Okta login page:

{% img blog/spring-gateway/okta-login.png alt:"Okta login form" width:"500" %}{: .center-image }

After the login, the idToken and accessToken will be displayed in the browser.

```
idToken: eyJraWQiOiIwYVM4bk4tM241emZYRDJfMU1yYUhzYURUZ0trVWZ4aWNaQXZDc0Fwb2lzIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHUxaDcwc2YwdHJSc2JMUzM1NyIsIm5hbWUiOiJKaW1lbmEgR2FyYmFyaW5vIiwiZW1haWwiOiJqaW1lbmFAdG9wdGFsLmNvbSIsInZlciI6MSwiaXNzIjoiaHR0cHM6Ly9kZXYtNjQwNDI5Lm9rdGEuY29tL29hdXRoMi9kZWZhdWx0IiwiYXVkIjoiMG9hNGc1ZzYzYWZTa01qZ3QzNTciLCJpYXQiOjE1OTM0NTUxMjEsImV4cCI6MTU5MzQ1ODcyMSwianRpIjoiSUQuWTZHdVFsWElSeU03U1FNNFRmN2tLdUV0MnlySW84U0l5eUVoMFVfaEpvUSIsImFtciI6WyJwd2QiXSwiaWRwIjoiMDBvMWg3MHNidmRMRjlpczYzNTciLCJub25jZSI6IlphUjZCT25OcGJ5X1dZSUQyRkNjeDJCUzR5ZExsNjJWcldVdEFaWDYwZmciLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJqaW1lbmFAdG9wdGFsLmNvbSIsImF1dGhfdGltZSI6MTU5MzQ1MzUzNSwiYXRfaGFzaCI6ImYtUU9KdWVqUzF1MHNRaDJ4ZWJQUlEifQ.QNgw3EuOXdi-Pq8ZoNmWGTYlOU-Wz608ieAXEEWmk5KgRbdCJdQGBH35ALc_fO-2CLmDamoI2tbOWdz79vcmDfOv-tlztXBS-At3M20ZPRSF39mHVs0vHKqmNIgdo5E6Dw0Yl_vqWdIZukEj6zjSVOpR1znHqWf0Bln_C7w9S9OOE8Z7yclHE7R5fzPVSEdT_1TpW9MRlcQKEmmTrskhsSem1wXEOceE7qlhebw-NhAGOnZD0Vr6WklRjaIMZ5zI-pKioJI9ysN7bByJJ9O9DpmsSswNfDtlIVqq-DbUu15Vze-D4pHTkO8GbYC1xh7YUxuN39viK8nBzxiBEwqx2w


accessToken: eyJraWQiOiIwYVM4bk4tM241emZYRDJfMU1yYUhzYURUZ0trVWZ4aWNaQXZDc0Fwb2lzIiwiYWxnIjoiUlMyNTYifQ.eyJ2ZXIiOjEsImp0aSI6IkFULnAxbElmR0ZpU3drM29kTjZZdEI3VmxBV1A2NllEWGpsNjlkYmJqY0VnN2suZDFBNzVkcitzWTE3MFZzeUFkKzNEQnhkbTlFT080QXdQUjNkVStkdXFpRT0iLCJpc3MiOiJodHRwczovL2Rldi02NDA0Mjkub2t0YS5jb20vb2F1dGgyL2RlZmF1bHQiLCJhdWQiOiJhcGk6Ly9kZWZhdWx0IiwiaWF0IjoxNTkzNDU1MTIxLCJleHAiOjE1OTM0NTg3MjEsImNpZCI6IjBvYTRnNWc2M2FmU2tNamd0MzU3IiwidWlkIjoiMDB1MWg3MHNmMHRyUnNiTFMzNTciLCJzY3AiOlsicHJvZmlsZSIsIm9mZmxpbmVfYWNjZXNzIiwiZW1haWwiLCJwaG9uZSIsIm9wZW5pZCIsImFkZHJlc3MiXSwic3ViIjoiamltZW5hQHRvcHRhbC5jb20ifQ.biSKfFmBupiu7b8vViyVrqRDprxj70bb3xhcFujRV37xknnF2gxgyX3iteEbHVdQpCbnJrLMZ6mqKDxhhnPnOMfgZWXGKYk92PzJ6EoNF99Ui7-3ynZMx3glwYHiLERHZsGkHX88jhYD8IP-5QUQ7phXD8sALp-Di2AvpJVTRam3NmiEZe8hnkUwGNMq7jsPPL953ldvF38fjMMEG6ReHu7wW4yQnVYHskpLHQnZdRNiOxq8QgJ-V6MpWzKSdEI5QCwoOpervMPTvNi0UfSUIMjdvNnCqGCwXZwg6UdCYUyVLRIz9nOzh4yMCEdWD4DHDe6ggiY4rhCI8g2DOpjJWQ

```

## Pattern 2: Token Relay to Service


{% img blog/spring-gateway/token-relay.png alt:"Token Relay Flow" width:"600" %}{: .center-image }

Let's create a cart service.





TokenRelayGatewayFilterFactory deprecated
From the principal, it gets the client registration id and finds the authorized client, to obtain the access token.

Another implementation would be to create a custom filter that gets the client registration and adds the bearer token.


```shell
curl \
  -H 'Accept: application/json' \
  -H "Authorization: Bearer {AccessToken}" \
  https://localhost:8080/cart/1
```
```json
{
   "id":1,
   "customerId":"customer@email.com",
   "total":{
      "amount":278.78,
      "currency":"USD",
      "formatted":"USD278.78"
   },
   "lineItems":[
      {
         "id":2,
         "productName":"jeans",
         "quantity":1,
         "price":{
            "amount":70.46,
            "currency":"USD",
            "formatted":"USD70.46"
         }
      },
      {
         "id":3,
         "productName":"t-shirt",
         "quantity":3,
         "price":{
            "amount":69.44,
            "currency":"USD",
            "formatted":"USD69.44"
         }
      }
   ]
}  
```      

## Pattern 3: Service to Service Client Credentials Grant

{% img blog/spring-gateway/credentials-grant.png alt:"Credentials Grant Flow" width:"800" %}{: .center-image }


## Learn More

- [Secure Reactive Microservices with Spring Cloud Gateway](https://developer.okta.com/blog/2019/08/28/reactive-microservices-spring-cloud-gateway)
- [Secure Legacy Apps with Spring Cloud Gateway](https://developer.okta.com/blog/2020/01/08/secure-legacy-spring-cloud-gateway)
- [Secure Server-to-Server Communication with Spring Boot and OAuth 2.0](https://developer.okta.com/blog/2018/04/02/client-creds-with-spring-boot)
- [Secure Service-to-Service Spring Microservices with HTTPS and OAuth 2.0](https://developer.okta.com/blog/2019/03/07/spring-microservices-https-oauth2)
- [Use Okta Token Hooks to Supercharge OpenID Connect](https://developer.okta.com/blog/2019/12/23/extend-oidc-okta-token-hooks)
- [Building a Gateway](https://spring.io/guides/gs/gateway/)
