---
title: "Optimize your Spring Reactive API with Redis Cache"
date: "2023-10-03T17:31:24.427Z"
description: "In order to to improve the performance, scalability, and responsiveness of your application you may need to implement a cache mechanism that will allow us to enhance the responses of our API. This…"
---
In order to to improve the performance, scalability, and responsiveness of your application you may need to implement a cache mechanism that will allow us to enhance the responses of our API.

Redis and its Benefits
======================

Redis is an open-source, in-memory data store, offers several benefits when used as a cache.

*   Key-Value Store: Redis is a simple key-value store, which makes it easy to use for caching purposes. Each key can be associated with a value, and you can set an expiration time on keys to implement automatic cache expiration.
*   High Performance: Redis is designed for speed and is known for its exceptional read and write performance.
*   Low Latency: Redis operates entirely in-memory, which means it can provide extremely low-latency data access.

Prerequisites
=============

To run this project you should have the following on your development environment :

*   Java 17 or Higher
*   Maven
*   Your Favorite IDE ( IntelliJ in my case)
*   Docker

Let’s get started
=================

This project will be a simple Rest Reactive API with an hexagonal architecture (seen in the previous [blog](https://medium.com/@jugurtha.aitoufella/facilitate-your-integration-tests-with-spring-docker-compose-31963469e789))

Our project will use a MongoDB database and a Redis cache, so first we’ll define a docker compose with the services we need at the root of the repository.

```
version: '3.7'  
  
services:  
  redis:  
    image: redis:latest  
    container\_name: redis-hexagonal  
    ports:  
      - "6379:6379"  
  mongodb:  
    image: mongo:latest  
    container\_name: mongodb-hexagonal  
    restart: always  
    environment:  
      MONGO\_INITDB\_ROOT\_USERNAME: test  
      MONGO\_INITDB\_ROOT\_PASSWORD: test  
      MONGO\_INITDB\_DATABASE: personal-db  
    ports:  
      - "27017:27017"
```

In the pom.xml we’ll add the **spring-boot-starter-data-redis-reactive** dependency and the **jackson-datatype-jsr310** (for serializing dates in our case)

```
<dependency>  
    <groupId>com.fasterxml.jackson.datatype</groupId>  
    <artifactId>jackson-datatype-jsr310</artifactId>  
    <version>${jackson-version}</version>  
</dependency>  
  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>  
</dependency>
```

in our application.yml we’ll add redis as a spring data source

```
server:  
  port: ${SERVER\_PORT:8080}  
management:  
  endpoint:  
    health:  
      enabled: true  
    info:  
      enabled: true  
spring:  
  profiles:  
    active: ${ACTIVE\_PROFILE:dev}  
  data:  
    redis:  
      connect-timeout: 2000  
      host: 127.0.0.1  
      port: 6379  
    mongodb:  
      database: ${DATABASE\_NAME:personal-db}  
      uri: ${DATABASE\_URI:mongodb://test:test@localhost:27017/?retryWrites=true&w=majority}
```

After that we’ll need to set up a config for our redis cache

```
package com.jai.hexagonal.infrastructure.out.persistence.redis.config;  
  
  
import com.jai.hexagonal.infrastructure.out.persistence.mongodb.entity.PersonEntity;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.data.redis.connection.ReactiveRedisConnectionFactory;  
import org.springframework.data.redis.core.ReactiveHashOperations;  
import org.springframework.data.redis.core.ReactiveRedisTemplate;  
import org.springframework.data.redis.serializer.GenericToStringSerializer;  
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;  
import org.springframework.data.redis.serializer.RedisSerializationContext;  
import org.springframework.data.redis.serializer.StringRedisSerializer;  
  
@Configuration  
public class RedisConfig {  
  
    @Bean  
    public ReactiveHashOperations<String, Long, PersonEntity> hashOperations(ReactiveRedisConnectionFactory redisConnectionFactory) {  
        var template = new ReactiveRedisTemplate<>(  
                redisConnectionFactory,  
                RedisSerializationContext.<String, PersonEntity>newSerializationContext(new StringRedisSerializer())  
                        .hashKey(new GenericToStringSerializer<>(Long.class))  
                        .hashValue(new Jackson2JsonRedisSerializer<>(PersonEntity.class))  
                        .build());  
        return template.opsForHash();  
    }  
}
```

we define a ReactiveHashOperations Bean that has the following parameters , the type of the cache Key ( String) , the field type (Long) and the type of the object we put in cache.

In order to put an object in redis cache, the object should be serializable, in our case we added the JsonSerialize and JsonDeserialize on the LocalDate field since it’s not serialized by default, we have to specify a deserializer.

```
package com.jai.hexagonal.infrastructure.out.persistence.mongodb.entity;  
  
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;  
import com.fasterxml.jackson.databind.annotation.JsonSerialize;  
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;  
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;  
import lombok.\*;  
import org.springframework.data.annotation.Id;  
import org.springframework.data.mongodb.core.mapping.Document;  
import org.springframework.data.mongodb.core.mapping.Field;  
  
import java.io.Serializable;  
import java.time.LocalDate;  
  
@Document(collection = "person")  
@Data  
@AllArgsConstructor  
@Builder(toBuilder = true)  
@NoArgsConstructor  
  
public class PersonEntity implements Serializable {  
  
    @Id  
    private Long id;  
    @Field("first\_name")  
    private String firstName;  
    @Field("last\_name")  
    private String lastName;  
    @Field("birth\_date")  
    @JsonSerialize(using = LocalDateSerializer.class)  
    @JsonDeserialize(using = LocalDateDeserializer.class)  
    private LocalDate birthDate;  
}
```

Then we’ll go to our PersonMongoRepositoryAdapter, we’ll define a method to update the redis cache.

```
private Mono<PersonEntity> updateRedisCache(PersonEntity personEntity) {  
    return hashOperations.put(KEY, personEntity.getId(), personEntity)  
            .thenReturn(personEntity);  
}
```

and everytime we have database requests we’ll check the redis cache first for the asked data and search through the database if it is not present in the cache and update the cache accordingly.

```
@Override  
public Mono<Person> save(Person person) {  
    return reactivePersonMongoRepository.save(personMongoMapper.toEntity(person))  
            .flatMap(this::updateRedisCache)  
            .map(personMongoMapper::toDomain);  
}  
  
@Override  
public Mono<Person> findById(Long id) {  
    return hashOperations.get(KEY, id)  
            .switchIfEmpty(reactivePersonMongoRepository.findById(id))  
            .flatMap(this::updateRedisCache)  
            .map(personMongoMapper::toDomain);  
}
```

You can find the complete implementation on [Github](https://github.com/jugurta/hexagonal).

Conclusion
==========

Using Redis as a cache for your API can significantly improve its performance, reduce the load on backend resources, and enhance the overall user experience.

By seamlessly storing frequently accessed data in-memory and offering various data structures and persistence options, Redis significantly enhances an API’s performance, scalability, and responsiveness