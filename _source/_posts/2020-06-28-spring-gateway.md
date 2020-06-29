---
layout: blog_post
title: "Spring Cloud Gateway OAuth2 Patterns"
author: jimena-garbraino
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

spring-gateway introduction

**Prerequisites**: [Java 8](https://adoptopenjdk.net/)+

In this tutorial we will create a microservices architecture to learn the configuration of three common OAuth2 patterns, using Okta as authentication provider:

- Spring Cloud Gateway OpenID Connect Login
- Cart Microservice OAuth2 Authorization
- Pricing Microservice Client Credentials Grant

## Pattern 1: Authentication with Authorization Code Flow

{% img blog/spring-gateway/authorization-code-flow.png alt:"Okta OAuth Code Flow" width:"600" %}{: .center-image }






## Pattern 2: Token Relay to Service


{% img blog/spring-gateway/token-relay.png alt:"Token Relay Flow" width:"600" %}{: .center-image }



TokenRelayGatewayFilterFactory deprecated
From the principal, it gets the client registration id and finds the authorized client, to obtain the access token.

It would be the actual relay of the access_token was sent by the client, but it is in session.
Or if service A receives accessToken and relays to service B.

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
