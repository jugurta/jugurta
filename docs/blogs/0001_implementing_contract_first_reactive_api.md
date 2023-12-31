---
title: "Implementing a Reactive API using Contract-First approach and Clean architecture with Spring Boot 3"
date: "2023-09-24T13:13:48.883Z"
description: "In this article we will explain how to develop a reactive spring boot api by using the contract-first approach within the boundaries of a clean architecture. A “Contract-First” API design approach is…"
---
In this article we will explain how to develop a reactive spring boot api by using the **contract-first** approach within the boundaries of a **clean architecture**.

**Plan**
========

*   Background
*   Prerequisites
*   Implementation
*   Build and start the API
*   Conclusion

**Background**
==============

A “Contract-First” API design approach is a methodology where you define the contract or specification of your API before you start implementing it. This contract serves as a blueprint for the API, outlining its endpoints, data structures, request/response formats, and other essential details.

As for clean architecture, it’s an approach that provides clear separation of responsibilities and promoting code maintainability.

**Prerequisites**
=================

To run this project you should have the following on your development environment :

*   Java 17 or Higher
*   Maven
*   Your Favorite IDE ( IntelliJ in my case)
*   Docker ( optional : if you want to emulate a mongo database)
*   Postman

Implementation
==============

Defining the contract
---------------------

We will start our implementation by defining the OpenApi contract.

OpenAPI specs has a few sections while defining the APIs:

\- Info

\- Paths

\- Components

First we define the openapi version ( currently the latest version in 3.1.0 but the generator plugin is still in development for this version)

In the info section, we define the description, version and contact information.

```
openapi: "3.0.3"  
info:  
  description: "This project provides endpoints for creating and getting Persons"  
  version: "@project.version@"  
  title: "Person API"  
  contact:  
    email: "jugurtaaito@gmail.com"
```

Then we define the component used by the API , This is to define the objects (models) you use in your API definitions.

```
components:  
  schemas:  
    PersonDTO:  
      description: Person DTO  
      title: PersonDTO  
      type: object  
      required: \[ id, firstName, lastName, birthDate \]  
      properties:  
        id:  
          type: integer  
        firstName:  
          type: string  
        lastName:  
          type: string  
        birthDate:  
          type: string  
          format: date
```

Paths : consists of the API paths, it defines endpoints of the REST operation, description, request, response structures and HTTP response codes.

```
paths:  
  /person:  
    post:  
      tags:  
        - Person API  
      description: Create a Person  
      operationId: createPerson  
      requestBody:  
        required: true  
        content:  
          application/json:  
            schema:  
              $ref: '#/components/schemas/PersonDTO'  
      responses:  
        200:  
          description: Successful Operation  
          content:  
            application/json:  
              schema:  
                $ref: '#/components/schemas/PersonDTO'  
  /person/{id}:  
    get:  
      tags:  
        - Person API  
      description: get a person with given id  
      operationId: getPerson  
      parameters:  
        - in: path  
          name: id  
          required: true  
          schema:  
            type: integer  
      responses:  
        '200':  
          description: the person with specified ID  
          content:  
            'application/json':  
              schema:  
                $ref: '#/components/schemas/PersonDTO'
```

Setting up the technical foundation for the API
-----------------------------------------------

Once we defined our contract we will begin developing our Spring Boot App.

Go to [https://start.spring.io/](https://start.spring.io/)

Choose the latest stable version of Spring Boot and add the following dependencies :

*   Spring Reactive Web (WebFlux)
*   Spring Data Reactive MongoDB
*   Spring Boot Actuator ( to display health information for our service)

We’ll also add Lombok and MapStruct dependencies, we’ll also add them in the maven compiler plugin so we can have the annotation processing when building the project.

Last we’ll add spring doc dependency ( so we can generate the html template of the api doc ) and the jackson databind dependency which is mandatory for the usage of the openapi generator.

OpenApi Generator Plugin
------------------------

the openapi generator plugin enables us to generate source code from the openapi specificiation file.

The plugin configuration has many possibilities, we can generate reactive and non reactive APIs, you can find the full configuration options description on this [link](https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin)

Below is the plugin configuration to generate our server, it will generate the model DTOs and the Java Interfaces for the endpoints.

```
<plugin>  
    <groupId>org.openapitools</groupId>  
    <artifactId>openapi-generator-maven-plugin</artifactId>  
    <version>${openapi-generator.version}</version>  
    <executions>  
        <execution>  
            <goals>  
                <goal>generate</goal>  
            </goals>  
            <configuration>  
                <skipValidateSpec>true</skipValidateSpec>  
                <inputSpec>${project.basedir}/src/main/resources/openapi/openapi.yml</inputSpec>  
                <generatorName>spring</generatorName>  
                <generateSupportingFiles>false</generateSupportingFiles>  
                <configOptions>  
                    <reactive>true</reactive>  
                    <openApiNullable>true</openApiNullable>  
                    <interfaceOnly>true</interfaceOnly>  
                    <useSpringBoot3>true</useSpringBoot3>  
                    <skipDefaultInterface>true</skipDefaultInterface>  
                    <library>spring-boot</library>  
                    <apiPackage>${project.groupId}.app.application.rest.controller</apiPackage>  
                    <modelPackage>${project.groupId}.app.application.rest.dto</modelPackage>  
                    <useResponseEntity>false</useResponseEntity>  
                </configOptions>  
            </configuration>  
        </execution>  
    </executions>  
</plugin>
```

Setting up our project architecture
-----------------------------------

Clean Architecture promotes modularity, testability, and flexibility, allowing you to change the infrastructure components without affecting the core business logic of the application

Since we decided to implement the clean architecture for ou java project, we will proceed according to its rules.

We’ll have four layers :

*   **The Application layer** : where we’ll have the controllers and presenters that are responsible for executing use cases. They receive input from the outer layers
*   **The Usecase layer :** which contains the components that defines the use cases of the application, Each use case represents a specific operation or feature provided by the application.
*   **The domain layer** : which contains the data model and the business logic of our application.
*   **The infrastructure layer :** it contains the technical details and implementation-specific code, such as database access, network communication, and user interface components

Once the architecture is set we’ll begin implementing our service.

**Implementing the Domain Layer**

First we’ll begin by defining our domain, in our case we have a single model class Person.

To transfer objects from layer to layer we’ll use a mapStruct mapper that map from DTO to Model , from Model to DTO , from Model to Database Entity and from Database Entity to Model so that each layer can manipulate only its objects.

```
package com.jai.app.domain.model;  
  
import lombok.\*;  
  
import java.time.LocalDate;  
  
@Data  
@Builder(toBuilder = true)  
@AllArgsConstructor  
public class Person {  
    private Long id;  
    private String firstName;  
    private String lastName;  
    private LocalDate birthDate;  
}
```

We’ll also define an interface in the domain to perform operation on the model, with this we’ve finished the domain layer.

```
package com.jai.app.domain.repository;  
  
import com.jai.app.domain.model.Person;  
import reactor.core.publisher.Mono;  
  
public interface PersonRepository {  
    Mono<Person> save(Person person);  
    Mono<Person> findById(Long id);  
}
```

**Implementing the UseCase Layer**

We’ll start implementing our usecase layer.

In our case we’ll have two usecases Creating a Person and Fetching a Person

```
package com.jai.app.usecase;  
  
  
import com.jai.app.domain.model.Person;  
import com.jai.app.domain.repository.PersonRepository;  
import lombok.RequiredArgsConstructor;  
import org.springframework.stereotype.Component;  
import reactor.core.publisher.Mono;  
  
@RequiredArgsConstructor  
@Component  
public class PersonCreator {  
    private final PersonRepository personRepository;  
    public Mono<Person> createPerson(Person person) {  
        return personRepository.save(person);  
    }  
}
```
```
package com.jai.app.usecase;  
  
import com.jai.app.domain.model.Person;  
import com.jai.app.domain.repository.PersonRepository;  
import lombok.RequiredArgsConstructor;  
import org.springframework.stereotype.Component;  
import reactor.core.publisher.Flux;  
import reactor.core.publisher.Mono;  
  
@Component  
@RequiredArgsConstructor  
public class PersonFetcher {  
  
    private final PersonRepository personRepository;  
    public Mono<Person> findById(Long id) {  
        return personRepository.findById(id);  
    }  
  
}
```

**Implementing the Application Layer**

In order to implement the application layer ( Rest Endpoint in our case ) we’ll use the interfaces, methods and DTO generated by OpenApi Generator.

We’ll implement the generated interface so we can add our operations

```
package com.jai.app.application.rest.controller;  
  
import com.jai.app.application.rest.dto.PersonDTO;  
import com.jai.app.application.rest.mapper.PersonDTOMapper;  
import com.jai.app.usecase.PersonCreator;  
import com.jai.app.usecase.PersonFetcher;  
import lombok.RequiredArgsConstructor;  
import org.springframework.web.bind.annotation.RestController;  
import org.springframework.web.server.ServerWebExchange;  
import reactor.core.publisher.Mono;  
  
@RestController  
@RequiredArgsConstructor  
public class PersonControllerApi implements PersonApi {  
  
    private final PersonDTOMapper personDTOMapper;  
    private final PersonCreator personCreator;  
    private final PersonFetcher personFetcher;  
  
    @Override  
    public Mono<PersonDTO> createPerson(Mono<PersonDTO> personDTOMono, ServerWebExchange exchange) {  
        return personDTOMono  
                .map(personDTOMapper::toDomain)  
                .flatMap(personCreator::createPerson)  
                .map(personDTOMapper::toDTO);  
    }  
  
    @Override  
    public Mono<PersonDTO> getPerson(Integer id, ServerWebExchange exchange) {  
        return personFetcher.findById(Long.valueOf(id)).map(personDTOMapper::toDTO);  
    }  
  
}
```

**Implementing the Infrastructure Layer**

In this part we’ll implement the technical operation to store the data, in our case we chose a mongoDB database.

We’ll start by implementing the Database Entity

```
package com.jai.app.infrastructure.database.mongo.entity;  
  
  
import lombok.\*;  
import org.springframework.data.annotation.Id;  
import org.springframework.data.mongodb.core.mapping.Document;  
import org.springframework.data.mongodb.core.mapping.Field;  
  
import java.time.LocalDate;  
  
@Document(collection = "person")  
@Data  
@AllArgsConstructor  
@Builder(toBuilder = true)  
public class PersonEntity {  
  
    @Id  
    private Long id;  
    @Field("first\_name")  
    private String firstName;  
    @Field("last\_name")  
    private String lastName;  
    @Field("birth\_date")  
    private LocalDate birthDate;  
}
```

Then we’ll create the ReactiveMongoRepository that will serve us in performing operation on the database.

```
package com.jai.app.infrastructure.database.mongo.repository;  
  
import com.jai.app.infrastructure.database.mongo.entity.PersonEntity;  
import org.springframework.data.mongodb.repository.ReactiveMongoRepository;  
  
public interface ReactiveMongoPersonRepository extends ReactiveMongoRepository<PersonEntity, Long> {  
}
```

Finally we’ll implement the Adapter for the PersonRepository Interface

```
package com.jai.app.infrastructure.database.mongo.adapter;  
  
import com.jai.app.domain.model.Person;  
import com.jai.app.domain.repository.PersonRepository;  
import com.jai.app.infrastructure.database.mongo.mapper.MongoPersonMapper;  
import com.jai.app.infrastructure.database.mongo.repository.ReactiveMongoPersonRepository;  
import lombok.RequiredArgsConstructor;  
import org.springframework.stereotype.Component;  
import reactor.core.publisher.Mono;  
  
@Component  
@RequiredArgsConstructor  
public class PersonRepositoryAdapter implements PersonRepository {  
    private final ReactiveMongoPersonRepository reactiveMongoPersonRepository;  
    private final MongoPersonMapper mongoPersonMapper;  
    @Override  
    public Mono<Person> save(Person person) {  
        return reactiveMongoPersonRepository.save(mongoPersonMapper.toEntity(person)).map(mongoPersonMapper::toDomain);  
    }  
    @Override  
    public Mono<Person> findById(Long id) {  
        return reactiveMongoPersonRepository.findById(id).map(mongoPersonMapper::toDomain);  
    }  
}
```

and with this we’ve finished the implementation of our micro-service.

**Build and Start the API**
===========================

After setting all the configurations in the application.yml we’ll build the application.

```
server:  
  port: ${SERVER\_PORT:8080}  
management:  
  health:  
    binders:  
      enabled: true  
  endpoint:  
      include: health,info,metrics,prometheus  
spring:  
  profiles:  
    active: ${ACTIVE\_PROFILE:dev}  
  data:  
    mongodb:  
      database: ${DATABASE\_NAME:personal-db}  
      uri: ${DATABASE\_URI:mongodb://<username>:<password>@localhost:27017/?retryWrites=true&w=majority}
```

Once our application is launched, we can check if it’s ready at the URL, The call should return the status UP if everything is fine

```
http://localhost:{SERVER\_PORT}/actuator/health
```

We can also check the API documentation at the URL (thanks to spring-doc dependency)

```
http://localhost:{SERVER\_PORT}/webjars/swagger-ui/index.html
```

Let’s start consuming our API with Postman

*   First we’ll import our openAPI specs to generate the collection ([download](https://raw.githubusercontent.com/jugurta/apiFirstApp/master/src/main/resources/openapi/openapi.yml))

*   In the variable t ab of the collection we’ll set our base URL

Let’s send our first request then , Go to create Person Request, change the fields at your convenience and send.

Now let’s get the person with the ID = 1 , we’re getting John Doe which we created earlier.

So here we completed our reactive api implemented using the contract first approach within the borders of clean architecture.

You can find the complete implementation on [Github](https://github.com/jugurta/apiFirstApp).

Conclusion
==========

In summary, Clean Architecture and Contract-First Development both promote maintainability, flexibility, and collaboration in software development.

Adopting an API-first approach to software development offers numerous advantages. By starting with API design, you establish a clear and structured foundation for your application, enhancing overall project clarity and providing an always up to date documentation.

Clean architecture promotes separation of concerns, making codebases more modular and maintainable, its flexibility empowers developers to adapt to changing requirements and technology shifts without too much effort.