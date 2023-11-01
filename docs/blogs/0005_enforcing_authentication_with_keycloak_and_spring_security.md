---
title: "Enforce your Spring Reactive API Authentication with JWT and Keycloak"
date: "2023-10-22T14:58:10.320Z"
description: "When developping and API that is destined to be exposed inside a system or on the web , we need to ensure that only the authorized users and application have an access to our resources. In this…"
---
When developping and API that is destined to be exposed inside a system or on the web , we need to ensure that only the authorized users and application have an access to our resources.

In this article we’ll explore how to add an authentication layer to our Spring Reactive API with Keycloak.

Plan
====

*   Background
*   Prerequisites
*   Setting Up Keycloak
*   Implementing the security layer
*   Testing with Postman
*   Conclusion

Conclusion
==========

Background
==========

JSON web token (JWT) is an open standard that defines a compact and self-contained way for securely transmitting information between parties as a JSON object.

To ensure the authentication , it needs a resource server providing the authentication ( Keycloak in our case).

The client send the client\_id and client\_secret and the resource server send a JWT, the token is used in the header later to request the API resources.

Prerequisites
=============

To run this project you should have the following on your development environment :

*   Java 17 or Higher
*   Maven
*   Your Favorite IDE ( IntelliJ in my case)
*   Docker
*   Postman ( for testing the API)

Setting Up Keycloak
===================

There are many ways to install Keycloak as stated in the [official documentation](https://www.keycloak.org/guides)

We’ll go with the approach using docker :

```
version: '3.7'  
  
services:  
  
  keycloak:  
    image: quay.io/keycloak/keycloak:latest  
    container\_name: keycloak-phonebook  
    restart: always  
    environment:  
      KEYCLOAK\_ADMIN: admin  
      KEYCLOAK\_ADMIN\_PASSWORD: admin  
    ports:  
      - "8085:8080"  
    command: start-dev
```

Once keycloak launched , use the logins admin/admin to access the administration console.

Create a Keycloak Realm:

*   In the Keycloak admin console, create a new realm for your application. Realms are isolated environments to manage users, roles, and clients.

Define a Client in Keycloak:

*   Within the realm, create a client for your Spring Reactive application. Set the client’s authentication to ‘ON.’
*   Choose Service accounts roles as Authentication flow and Save

*   Once the client is created, go to the client tab and get the `Client ID` and `Client Secret` as you'll need them in your Spring application.

With this we have set up our client, let’s go implement our Security for our Spring Reactive API.

Implementing the security layer
===============================

We’ll go with a simple CRUD reactive Spring API, we’ll have to add these dependencies.

```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>  
</dependency>
```

in the application.yml we’ll add this config

```
spring:  
  security:  
    oauth2:  
      resource-server:  
        jwt:  
          jwk-set-uri: ${JWT\_PROVIDER\_URI:http://localhost:8085/realms/custom\_realm/protocol/openid-connect/certs}
```

We’ll need to define to define a config class that enforces the security

```
package com.jai.phonebookapi.infrastructure.in.rest.config.security;  
  
import lombok.RequiredArgsConstructor;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.beans.factory.annotation.Value;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.Profile;  
import org.springframework.security.config.Customizer;  
import org.springframework.security.config.annotation.method.configuration.EnableReactiveMethodSecurity;  
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;  
import org.springframework.security.config.web.server.ServerHttpSecurity;  
import org.springframework.security.oauth2.jwt.NimbusReactiveJwtDecoder;  
import org.springframework.security.oauth2.jwt.ReactiveJwtDecoder;  
import org.springframework.security.web.server.SecurityWebFilterChain;  
  
  
@EnableWebFluxSecurity  
@EnableReactiveMethodSecurity  
@Configuration  
@Profile("auth")  
@RequiredArgsConstructor  
@Slf4j  
public class OAuth2LoginSecurityConfig {  
  
    @Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}")  
    String jwkSetUri;  
  
    @Bean  
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {  
        log.info("Webflux security with auth activated");  
        return http  
                .csrf(ServerHttpSecurity.CsrfSpec::disable)  
                .authorizeExchange(auth -> {  
                    auth.pathMatchers("/actuator/health/\*\*").permitAll();  
                    auth.pathMatchers("/webjars/swagger-ui/\*\*").permitAll();  
                    auth.pathMatchers("/v3/api-docs/\*\*").permitAll();  
                    auth.anyExchange().authenticated();  
                })  
                .oauth2ResourceServer(oAuth2ResourceServerSpec -> oAuth2ResourceServerSpec.jwt(Customizer.withDefaults()))  
                .build();  
    }  
  
    @Bean  
    public ReactiveJwtDecoder jwtDecoder() {  
        return NimbusReactiveJwtDecoder.withJwkSetUri(jwkSetUri)  
                .build();  
    }  
}
```

Testing with Postman
====================

First let’s download the OpenApi spec from our [repository](https://github.com/jugurta/phonebook/blob/main/phonebook--api/src/main/resources/openapi/openapi.yml) and import it in postman to get the full collection.

Go to the Person API collection , in the Authorization tab , select OAuth 2.0

In the section Configure New Token select Client Credentials and complete the Client ID & Client Secret that you got earlier in keycloak, in the scope add email, then click on **Get New Access Token**.

And now we’re authenticated

Conclusion
==========

In this blog, we’ve walked through the essential steps to set up Keycloak, create a realm, define clients, and configure Spring Security to use JWTs for authentication. We’ve also seen how to secure endpoints and customize access controls according to your application’s specific requirements.

In conclusion, securing your Spring Reactive API with Keycloak and JWT authentication is a powerful and robust approach to protect your API endpoints.

This combination offers a standardized method for user authentication and authorization while enabling your application to handle asynchronous, non-blocking operations efficiently.

You can find the complete implementation on [Github](https://github.com/jugurta/phonebook).