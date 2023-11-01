---
title: "Facilitate your integration tests with Spring Docker Compose"
date: "2023-10-01T15:53:16.594Z"
description: "In the process of building robust software, we’ll need to test our component individually with unit tests and as a whole with integration tests. In this blog we’ll explore a new feature on Spring…"
---
In the process of building robust software, we’ll need to test our component individually with unit tests and as a whole with integration tests.

There are some libs like test containers that can emulate it, but there’s a new way to do it.

In this blog we’ll explore a new feature on Spring Boot 3.1 that allows us to launch automatically docker images at runtime or test on the application.

Integration tests with Spring Boot
----------------------------------

Integration tests, in the context of Spring Boot, involve testing the interaction between various components of your application. These components can include:

*   Controllers: Ensure that HTTP endpoints behave as expected.
*   Services: Test the business logic and any service layer components.
*   Repositories: Validate interactions with the database or external data sources.
*   Configuration: Ensure that beans are wired correctly and configuration properties are loaded properly.

Unlike unit tests that isolate and test individual classes and methods, integration tests focus on the collaboration between components. They aim to detect issues that might arise when these units work together.

Setting Up a Spring Boot Project
================================

Prerequisites
-------------

To run this project you should have the following on your development environment :

*   Java 17 or Higher
*   Maven
*   Your Favorite IDE ( IntelliJ in my case)
*   Docker

Let’s get started
=================

This project will be a simple Rest Reactive API with an hexagonal architecture.

Our project will use a MongoDB database, so first we’ll define a docker compose with the services we need at the root of the repository.

```
version: '3.7'  
  
services:  
    mongodb:  
        image: mongo:latest  
        container\_name: mongodb  
        restart: always  
        environment:  
            MONGO\_INITDB\_ROOT\_USERNAME: test  
            MONGO\_INITDB\_ROOT\_PASSWORD: test  
            MONGO\_INITDB\_DATABASE: personal-db  
        ports:  
            - 27017:27017
```

In the pom.xml we’ll add the spring-boot-docker-compose dependency and with the scope of test.

```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-docker-compose</artifactId>  
    <scope>test</scope>  
</dependency>
```

In our the test resource folder we’ll add a config file application.yml

We’ll have to add to set the skip-in-test to false so that our docker compose will be launched at testing.

```
server:  
  port: ${SERVER\_PORT:8080}  
spring:  
  data:  
    mongodb:  
      database: ${DATABASE\_NAME:personal-db}  
      uri: ${DATABASE\_URI:mongodb://test:test@localhost:27017/?retryWrites=true&w=majority}  
  docker:  
    compose:  
      skip:  
        in-tests: false
```

In order to define a test as an integration test we add the SpringBootTest annotation.

Our test will consists on using the API endpoints to ensure that we can create and retrieve a person from our application.

```
package com.jai.hexagonal.integration;  
  
  
import com.jai.hexagonal.infrastructure.in.rest.dto.PersonDTO;  
import com.jai.hexagonal.providers.PersonDTOProvider;  
import org.junit.jupiter.api.BeforeEach;  
import org.junit.jupiter.api.Order;  
import org.junit.jupiter.params.ParameterizedTest;  
import org.junit.jupiter.params.provider.ArgumentsSource;  
import org.springframework.boot.test.context.SpringBootTest;  
import org.springframework.boot.test.web.server.LocalServerPort;  
import org.springframework.http.MediaType;  
import org.springframework.web.reactive.function.BodyInserters;  
import org.springframework.web.reactive.function.client.WebClient;  
import reactor.core.publisher.Mono;  
import reactor.test.StepVerifier;  
  
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM\_PORT)  
class PersonOperationsTest {  
  
  
    @LocalServerPort  
    private int port;  
    WebClient webClient;  
  
    @BeforeEach  
    public void setUp() {  
        webClient = WebClient.builder().baseUrl("http://localhost:".concat(String.valueOf(port))).build();  
    }  
  
    @ParameterizedTest  
    @ArgumentsSource(PersonDTOProvider.class)  
    @Order(1)  
    void given\_personDto\_post\_should\_return\_theSame(PersonDTO personDTO) {  
        // GIVEN  
        Mono<PersonDTO> postResult = webClient.post().uri("/person").contentType(MediaType.APPLICATION\_JSON)  
                .body(BodyInserters.fromValue(personDTO)).retrieve().bodyToMono(PersonDTO.class);  
        // THEN  
        StepVerifier.create(postResult)  
                .expectNext(personDTO)  
                .verifyComplete();  
    }  
  
    @ParameterizedTest  
    @ArgumentsSource(PersonDTOProvider.class)  
    @Order(2)  
    void given\_personDto\_get\_should\_return\_previously\_created(PersonDTO personDTO) {  
        // WHEN  
        Mono<PersonDTO> result = webClient.get()  
                .uri("/person/{id}", 1L)  
                .retrieve().bodyToMono(PersonDTO.class);  
        //THEN  
        StepVerifier.create(result).expectNext(personDTO).verifyComplete();  
    }  
}
```

When the test is launched our docker compose services launch via the spring-docker-compose integration.

You can find the complete implementation on [Github](https://github.com/jugurta/hexagonal).

Drawbacks
=========

As of now , we’re not able to run any container we would like to, there’s a limited number of services availables.

We can use the most commonly needed services like MongoDB, Redis, PostgreSQL, Cassandra, Elasticsearch without any extra configuration.

I think there will be new added services as to cover all the usecases and needs of developers.

Conclusion
==========

As seen in this article this new module spring-boot-docker-compose is pretty powerful and easy to use.

We can use it to simulate the needed services and our testing without any external lib and can be a real alternative to TestContainers.

It’s still a new feature so there’s limited available service, But it’s probably just a matter of time when more services will be added to the spring-boot-docker-compose.