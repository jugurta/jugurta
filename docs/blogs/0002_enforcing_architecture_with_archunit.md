---
title: "Enforcing and Testing your Java Clean Architecture Project with ArchUnit"
date: "2023-09-27T07:09:01.130Z"
description: "In this blog we’ll explore how to enforce architectural rules for your Java/Spring project with archunit. We decided to design our project within the boundaries of clean architecture as it is a…"
---
In this blog we’ll explore how to enforce architectural rules for your Java/Spring project with archunit.

We decided to design our project within the boundaries of clean architecture as it is a robust approach for designing software.

Clean Architecture is a software architectural pattern that promotes the separation of concerns, maintainability, and testability in software systems.

It consists on organizing your software in multiple layers , the domain layer, the usecase layer, application layer and infrastructure layer.

Clean architecture has multiple benefits :

*   **Maintainability**: Clean architecture promotes modularity and separation of concerns, making it easier to understand and modify code. This leads to long-term maintainability, reducing the risk of technical debt.
*   **Testability**: A well-structured clean architecture encourages the use of automated tests, including unit tests, integration tests, and acceptance tests.
*   **Scalability**: Clean architecture facilitates scalability by allowing for the easy replacement of components and the addition of new features without affecting the existing software stability.
*   **Independence from Frameworks**: Clean architecture is designed to be independent of external frameworks, databases, and delivery mechanisms.

**Archunit**
============

ArchUnit is a library that allows the developer to define architectural rules in a concise and expressive manner.

Here are some common architectural rules:

*   **Naming Conventions**: Enforce naming conventions for packages, classes, and methods to maintain consistency throughout the project.
*   **Dependency Direction**: Ensure that dependencies flow from outer layers to inner layers, following the Dependency Rule of Clean Architecture.
*   **Package Structure**: Enforce the proper packaging of your application layers. For example, all classes related to the domain should reside in a well-defined package.

Let’s get started
=================

Let’s create a Spring project with a clean architecture that will have four layers, the presentation layer , the usecase layer , the domain layer and the infrastrcture layer.

Below is the architecture of our java project.

```
  
├───java  
│   └───com  
│       └───jai  
│           └───cleanarchitecture  
│               │   CleanarchitectureApplication.java  
│               │  
│               ├───domain  
│               │   ├───model  
│               │   │       Person.java  
│               │   │  
│               │   └───repository  
│               │           PersonRepository.java  
│               │  
│               ├───infrastructure  
│               │   └───persistence  
│               │       ├───adapter  
│               │       │       PersonRepositoryAdapter.java  
│               │       │  
│               │       ├───entity  
│               │       │       PersonEntity.java  
│               │       │  
│               │       ├───mapper  
│               │       │       PersonMongoMapper.java  
│               │       │  
│               │       └───repository  
│               │               ReactivePersonMongoRepository.java  
│               │  
│               ├───presentation  
│               │   ├───controller  
│               │   │       PersonController.java  
│               │   │  
│               │   ├───dto  
│               │   │       PersonDTO.java  
│               │   │  
│               │   └───mapper  
│               │           PersonDTOMapper.java  
│               │  
│               └───usecase  
│                       CreatePersonUseCase.java  
│                       FetchPersonUseCase.java  
│  
└───resources  
        application.yml
```

In order to ensure that new classes respect the architecture that we agreed to we’ll have to write a set of rules with **Archunit.**

In order to use Archunit we need to add the following dependencies

```
<dependency>  
    <groupId>com.tngtech.archunit</groupId>  
    <artifactId>archunit-junit5</artifactId>  
    <version>${archunit-junit5.version}</version>  
    <scope>test</scope>  
</dependency>
```

**Enforcing the general architecture of the project**

First we’ll check the general architecture of project is enforced

The presentation layer cannot be accessed by any layer, it can only access to Usecase layer.

The infrastructure layer cannot be accessed directly by any layer (we’ll only use the abstractions defined in domain)

```
package com.jai.cleanarchitecture.architecture;  
  
import com.tngtech.archunit.core.importer.ImportOption;  
import com.tngtech.archunit.junit.AnalyzeClasses;  
import com.tngtech.archunit.junit.ArchTest;  
import com.tngtech.archunit.lang.ArchRule;  
  
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;  
  
@AnalyzeClasses(packages = "com.jai.cleanarchitecture", importOptions = ImportOption.DoNotIncludeTests.class)  
class CleanArchitectureTest {  
  
    final static String presentationLayer = "Presentation";  
    final static String useCaseLayer = "UseCase";  
    final static String domainLayer = "Domain";  
    final static String infrastructureLayer = "Infrastructure";  
  
  
    @ArchTest  
    static final ArchRule layer\_dependencies\_are\_respected = layeredArchitecture()  
            .consideringAllDependencies()  
            .layer(presentationLayer).definedBy("com.jai.cleanarchitecture.presentation..")  
            .layer(useCaseLayer).definedBy("com.jai.cleanarchitecture.usecase..")  
            .layer(infrastructureLayer).definedBy("com.jai.cleanarchitecture.infrastructure..")  
  
            .whereLayer(presentationLayer).mayNotBeAccessedByAnyLayer()  
            .whereLayer(useCaseLayer).mayOnlyBeAccessedByLayers(presentationLayer)  
            .whereLayer(infrastructureLayer).mayNotBeAccessedByAnyLayer();  
  
}
```

**Enforcing naming convention**

We can also ensure that the naming convention we agreed on is enforced.

```
package com.jai.cleanarchitecture.architecture;  
  
  
import com.tngtech.archunit.core.importer.ImportOption;  
import com.tngtech.archunit.junit.AnalyzeClasses;  
import com.tngtech.archunit.junit.ArchTest;  
import com.tngtech.archunit.lang.ArchRule;  
import org.mapstruct.Mapper;  
  
  
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;  
  
@AnalyzeClasses(packages = "com.jai.cleanarchitecture", importOptions = ImportOption.DoNotIncludeTests.class)  
class NamingConventionTest {  
      
    @ArchTest  
    static ArchRule useCaseShouldBeSuffixed = classes()  
            .that()  
            .resideInAPackage("..usecase..")  
            .should()  
            .haveSimpleNameEndingWith("UseCase");  
  
    @ArchTest  
    static ArchRule controllerShouldBeSuffixed = classes()  
            .that()  
            .resideInAPackage("..presentation.controller..")  
            .should()  
            .haveSimpleNameEndingWith("Controller");  
  
    @ArchTest  
    static ArchRule mapperShouldBeSuffixed = classes()  
            .that()  
            .areAnnotatedWith(Mapper.class)  
            .should()  
            .haveSimpleNameEndingWith("Mapper");  
}
```

Other than the architecture enforcing we can also define rules that exclude the usage of some components like deprecated classes.

```
package com.jai.cleanarchitecture.architecture;  
  
import com.tngtech.archunit.core.domain.JavaClasses;  
import com.tngtech.archunit.core.importer.ClassFileImporter;  
import com.tngtech.archunit.lang.ArchRule;  
import org.junit.jupiter.api.Test;  
  
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;  
  
class DependenciesTest {  
  
    @Test  
    void doNotCallDeprecatedMethodsFromTheProject() {  
        JavaClasses importedClasses = new ClassFileImporter()  
                .importPackages("com.jai.cleanarchitecture");  
        ArchRule rule = noClasses().should()  
                .dependOnClassesThat()  
                .areAnnotatedWith(Deprecated.class);  
        rule.check(importedClasses);  
    }  
}
```

You can find the complete implementation on [Github](https://github.com/jugurta/cleanarchitecture).

Conclusion
==========

ArchUnit provides a robust framework for enforcing architectural rules and constraints, ensuring that your codebase adheres to the principles of your chosen architecture.

ArchUnit, you can catch architectural violations early in the development process, preventing technical debt and simplifying long-term maintenance.

By incorporating ArchUnit and Clean Architecture into your development workflow, you can foster collaboration among team members, reduce bugs and technical debt, and build software that is not only functional but also maintainable and adaptable to future changes.