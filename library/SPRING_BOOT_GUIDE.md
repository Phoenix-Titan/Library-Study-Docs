# Spring Boot — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** A Java developer who can already write classes, interfaces, generics, lambdas, and use Maven — and now wants to build **production-grade, scalable, maintainable backend services and REST APIs** with Spring Boot, without an internet connection. This guide assumes the language fundamentals are in place; if records, sealed types, streams, or virtual threads are unfamiliar, read the [Java](JAVA_GUIDE.md) guide first and come back. Everything here is explained **prose-first**: *what* a thing is, *the logic / why* Spring does it that way, *what it's for and when* you reach for it, *how* to use it, the *key annotations / options / parameters*, *best practices*, and *security recommendations* — and only then the heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Spring Boot 3.4 (Nov 2024) and 3.5 (May 2025)**, running on **Spring Framework 6.2**, **Java 21 LTS** as the baseline (Spring Boot 3 *requires* Java 17+; Java **25 LTS** is also fully supported). Build is with **Maven** primarily, with **Gradle** noted. Modern themes run throughout: **Jakarta EE namespaces** (`jakarta.*`, **not** `javax.*` — the single biggest breaking change in Spring Boot 3), **records for DTOs**, the **lambda Security DSL**, **virtual threads** (Java 21), **GraalVM native images** (AOT), and **Spring Initializr**. **⚡ Version note: Spring Boot 4.0 / Spring Framework 7** shipped around **November 2025** — the newest generation, raising the Java baseline and modularizing the framework further (API-versioning support, a refreshed HTTP client, tighter null-safety with JSpecify). Where 4.0 changes something it is flagged with **⚡ Version note**; this guide's code targets 3.4/3.5 because that is what most production systems run in June 2026. The author is on **Windows 11**, so OS-specific notes (`mvnw.cmd`, paths) are called out where they matter.
>
> **Cross-references:** Spring Boot is a backend framework, so it sits in a stack. The language layer is [Java](JAVA_GUIDE.md) (and Spring works beautifully with [Kotlin](KOTLIN_GUIDE.md) — see the Kotlin note in §1). The database layer leans on [PostgreSQL](POSTGRESQL_GUIDE.md) and sound schema design in [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md); operational concerns live in [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md). Caching uses [Redis](REDIS_GUIDE.md). In production you run **behind** [Nginx](NGINX_GUIDE.md) as a reverse proxy, packaged with [Docker](DOCKER_GUIDE.md), with the underlying [Networking](NETWORKING_GUIDE.md) concepts assumed. For RPC concepts (an alternative to REST) see [gRPC & RPC with Go](GO_GRPC_RPC_GUIDE.md). Source control throughout is [Git](GIT_GUIDE.md). Cross-links appear inline.

---

## Table of Contents

1. [What Spring & Spring Boot Are](#1-what-spring--spring-boot-are) **[B]**
2. [Inversion of Control & Dependency Injection](#2-inversion-of-control--dependency-injection) **[B/I]**
3. [Setup: Initializr, Maven, the Main Class](#3-setup-initializr-maven-the-main-class) **[B]**
4. [Configuration & Profiles](#4-configuration--profiles) **[B/I]**
5. [Building REST APIs](#5-building-rest-apis) **[B/I]**
6. [Validation & Error Handling](#6-validation--error-handling) **[I]**
7. [Data Access with Spring Data JPA](#7-data-access-with-spring-data-jpa) **[I/A]**
8. [The Persistence Landscape: JPA, JdbcClient, R2DBC, Pooling](#8-the-persistence-landscape-jpa-jdbcclient-r2dbc-pooling) **[I/A]**
9. [Spring Security](#9-spring-security) **[I/A]**
10. [The Service Layer & Transactions](#10-the-service-layer--transactions) **[I/A]**
11. [Aspect-Oriented Programming (AOP)](#11-aspect-oriented-programming-aop) **[A]**
12. [Testing](#12-testing) **[A]**
13. [Async, Scheduling, Virtual Threads & Messaging](#13-async-scheduling-virtual-threads--messaging) **[A]**
14. [Caching](#14-caching) **[I/A]**
15. [Observability & Actuator](#15-observability--actuator) **[A]**
16. [Production Deployment](#16-production-deployment) **[A]**
17. [Project Structure & Maintainability at Scale](#17-project-structure--maintainability-at-scale) **[A]**
18. [Performance & Scaling](#18-performance--scaling) **[A]**
19. [Security Hardening](#19-security-hardening) **[A]**
20. [Gotchas & Best Practices](#20-gotchas--best-practices) **[I/A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What Spring & Spring Boot Are

### What "Spring" is

**Spring** is not one library; it is a *framework ecosystem* for the JVM whose original and still-central job is to manage the **objects** that make up your application and the **dependencies between them**. The piece that does this is the **Spring Framework** core, and inside it lives the **IoC container** (Inversion of Control — covered in depth in §2). On top of that core, the ecosystem layers dozens of focused projects: **Spring MVC** (web/REST), **Spring Data** (databases), **Spring Security** (auth), **Spring AOP** (cross-cutting concerns), **Spring Batch**, **Spring Integration**, **Spring Cloud** (distributed systems), and many more. They all share one mental model — beans wired by a container — which is why learning the core pays off everywhere.

The reason Spring exists at all is historical and instructive. In the early 2000s, "enterprise Java" (J2EE / EJB) was *heavy*: building a simple service meant XML descriptors, container-managed objects, and code that was tangled together and almost impossible to unit-test. Spring's founding idea (Rod Johnson, 2003) was that your business objects should be **Plain Old Java Objects (POJOs)** — ordinary classes with no framework base class to extend — and that a lightweight container should *wire them together from the outside*. That idea, **dependency injection**, is the heart of Spring and the thing you must internalize (§2).

### What "Spring Boot" adds — the value proposition

Classic Spring was powerful but **configuration-heavy**: you assembled the pieces yourself (which web server, which JSON library, how to connect the datasource, how MVC is set up) typically through pages of XML or `@Configuration` classes. Spring Boot, launched in 2014, is a layer on top of Spring whose entire purpose is to make Spring **immediately usable** by encoding sensible defaults. Three ideas define it:

1. **Convention over configuration.** Boot assumes you want the common, reasonable setup, and gives it to you for free. You override only the bits that differ from the convention. A new app is productive in minutes, not days.
2. **Auto-configuration.** This is the magic. When Boot starts, it inspects your **classpath** and your settings and *conditionally* configures beans for you. Is `spring-boot-starter-web` on the classpath? Boot configures an embedded Tomcat, Spring MVC, a Jackson JSON mapper, and an error handler — all without you writing a line. Add a JDBC driver and a datasource URL, and Boot configures a connection pool and a `JdbcTemplate`. Each auto-configuration is guarded by `@Conditional` annotations (e.g. `@ConditionalOnClass`, `@ConditionalOnMissingBean`), so it only activates when relevant and **always backs off the moment you define your own bean**. Auto-config never fights you — your beans win.
3. **Starters.** A *starter* is a curated, transitive dependency bundle. Instead of hunting for the right versions of fifteen libraries that work together, you add `spring-boot-starter-web` and get a tested, compatible set. The **Bill of Materials (BOM)** — inherited via the Spring Boot parent POM — pins every version so you never specify a version number for a Spring-managed dependency. Upgrading Boot upgrades the whole consistent set.

Boot also bundles an **embedded server** (Tomcat by default; Jetty or Undertow optionally), so the deliverable is a single executable "fat jar" you run with `java -jar app.jar` — no external application server to install or configure. This is the shift that made Java a first-class citizen for cloud-native and Docker-based deployment (§16).

### Spring Boot vs plain Java / Jakarta EE

| Concern | Plain Java / Servlet | Jakarta EE (app server) | Spring Boot |
|---|---|---|---|
| Wiring objects | Manual `new` everywhere | CDI container | IoC container, DI by convention |
| Web/REST | Raw `HttpServlet` | JAX-RS | Spring MVC `@RestController` |
| Persistence | Raw JDBC | JPA via container | Spring Data JPA / JdbcClient |
| Server | You embed/deploy | Deploy WAR to WildFly/etc. | **Embedded**, runnable fat jar |
| Config | Properties by hand | Server-managed | Auto-config + profiles |
| Boilerplate | Very high | Medium | **Low** |
| Testability | DIY | Container-bound | First-class slice tests |

The crucial post-Spring-Boot-3 fact: Spring Boot 3 moved from the old `javax.*` package namespace to **`jakarta.*`** (e.g. `jakarta.persistence.Entity`, `jakarta.validation.Valid`, `jakarta.servlet.Filter`). This is because Oracle transferred Java EE to the Eclipse Foundation, which had to rename the packages for trademark reasons. **Any tutorial or Stack Overflow answer using `javax.persistence` is pre-Spring-Boot-3 and will not compile.** This is the number-one source of confusion for developers coming from older codebases.

### ⚡ Version note — Spring Boot 4.0 / Spring Framework 7

Released **~November 2025**, Spring Boot 4.0 (on Spring Framework 7) is the newest generation. It modularizes the framework, raises the minimum Java version, leans further into **null-safety via JSpecify annotations**, ships first-class **API versioning** for REST, refreshes the HTTP client story, and continues to invest in AOT/native and virtual threads. Adopt it for greenfield work once your dependency ecosystem has caught up; the concepts in this guide transfer directly, and specific 4.0-only behaviors are flagged inline. This guide's code targets 3.4/3.5 — the production mainstream in June 2026 — to keep examples runnable for the widest audience.

### How auto-configuration actually works (the mental model)

Auto-configuration feels like magic until you see the machinery, and seeing it is what turns Spring Boot from "framework I cargo-cult" into "framework I control." At startup, `@EnableAutoConfiguration` (inside `@SpringBootApplication`) loads a list of **auto-configuration classes** registered in each starter's `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file. Each of those classes is a normal `@Configuration` guarded by **`@Conditional`** annotations that decide whether it activates:

| Conditional | Activates the config only if… |
|---|---|
| `@ConditionalOnClass` | a class is on the classpath (e.g. `DataSource`) |
| `@ConditionalOnMissingBean` | you have **not** already defined that bean |
| `@ConditionalOnProperty` | a property has a given value |
| `@ConditionalOnBean` | another bean exists |
| `@ConditionalOnWebApplication` | this is a servlet/reactive web app |

The crucial pattern is `@ConditionalOnMissingBean`: auto-config defines a bean **only if you haven't**. So the instant you declare your own `ObjectMapper` (or `DataSource`, or `SecurityFilterChain`), Boot's default **backs off** and yours wins. This is why Spring Boot is opinionated *without being bossy* — every default is a fallback you can override. To see exactly what activated and why, run with `--debug` (or `logging.level...`) to print the **auto-configuration report**, which lists every positive and negative match. When something "isn't being configured," that report is the first place to look.

You can write your *own* auto-configuration (for an internal shared library) the same way: a `@Configuration` with conditionals, listed in the imports file. That's how internal "platform" starters standardize cross-team setup.

### A note on Kotlin

Spring has **first-class [Kotlin](KOTLIN_GUIDE.md) support**: the same annotations work, and Spring even ships Kotlin-friendly DSLs (a bean-definition DSL, a routing DSL for WebFlux) and the `kotlin-spring` compiler plugin that auto-opens classes Spring needs to proxy (Kotlin classes are `final` by default; Spring AOP needs to subclass them). If you prefer Kotlin's concise data classes and null-safety, you lose nothing — everything in this guide applies, with `data class` standing in for Java `record`. Kotlin's `?`/non-null types map onto Spring's null-safety, and `runapplication<App>(args)` is the idiomatic `main`. Coroutines are supported in WebFlux (suspending controller functions). For teams already fluent in Kotlin, Spring Boot is arguably *more* pleasant there than in Java.

---

## 2. Inversion of Control & Dependency Injection

This is **the** foundational concept. If you understand nothing else deeply, understand this — every other feature (web, data, security, transactions, AOP) is built on top of the container.

### The problem DI solves

Consider an `OrderService` that needs a `PaymentGateway` and an `OrderRepository`. The naive approach is for the service to *construct* its own collaborators:

```java
public class OrderService {
    // ❌ The service is now WELDED to concrete implementations.
    private final PaymentGateway gateway = new StripePaymentGateway();
    private final OrderRepository repo = new JpaOrderRepository();
}
```

This is *tight coupling*, and it hurts in three concrete ways: (1) you cannot test `OrderService` without a real Stripe and a real database, because it builds them itself; (2) swapping `StripePaymentGateway` for `PayPalPaymentGateway` means editing the service's source; (3) if `StripePaymentGateway` itself needs an API key and an HTTP client, the service must know how to build *those* too, and the knowledge spreads everywhere.

**Inversion of Control (IoC)** inverts *who is in charge of construction*. Instead of the object building its dependencies, an external **container** builds them and **hands them in**. **Dependency Injection (DI)** is the specific technique: a class *declares* what it needs (usually as constructor parameters) and the container *supplies* it. The class no longer says "give me a Stripe gateway"; it says "give me a `PaymentGateway`" and trusts the container to provide the right one.

```java
public class OrderService {
    private final PaymentGateway gateway;
    private final OrderRepository repo;
    // The service DECLARES its needs; it does not construct them.
    public OrderService(PaymentGateway gateway, OrderRepository repo) {
        this.gateway = gateway;
        this.repo = repo;
    }
}
```

Now in a test you pass mocks; in production the container passes real beans; swapping implementations is a configuration change, not a code change.

### The ApplicationContext and beans

The Spring **container** is represented by the **`ApplicationContext`**. When your app starts, Spring scans for classes it should manage, instantiates them, wires their dependencies, and stores them. Each managed object is a **bean**. By default beans are **singletons** — one shared instance per context — which is correct for stateless services and repositories.

How does Spring know *which* classes to manage? Two ways, used together:

- **Component scanning + stereotype annotations.** Spring scans the package of your main class and its sub-packages for classes annotated with `@Component` or one of its specializations. It registers each as a bean.
- **`@Bean` factory methods inside `@Configuration` classes.** For objects you don't own (third-party classes) or need to construct with logic, you write a method that returns the object and annotate it `@Bean`. Spring calls it and registers the return value.

### The stereotype annotations

`@Component` is the generic "this is a Spring bean" marker. Its specializations carry the same registration behavior but communicate *role* and sometimes add behavior:

| Annotation | Layer / role | Extra behavior |
|---|---|---|
| `@Component` | Generic bean | none |
| `@Service` | Business-logic layer | semantic only (marks a service) |
| `@Repository` | Data-access layer | **translates persistence exceptions** into Spring's `DataAccessException` hierarchy |
| `@Controller` | Web MVC controller (returns views) | enables MVC handler mapping |
| `@RestController` | REST controller | `@Controller` + `@ResponseBody` (returns serialized bodies) |
| `@Configuration` | Source of `@Bean` definitions | beans defined here are CGLIB-enhanced so inter-bean calls return singletons |

Use the **most specific** stereotype that fits — it documents intent and, for `@Repository`, gives you free exception translation.

### Constructor injection — the preferred style

There are three injection styles; the correct default is **constructor injection**, for reasons that matter in production:

- **Immutability & final fields.** Dependencies become `final`, so a fully-constructed bean is never in a half-wired state.
- **Mandatory dependencies are explicit.** If a required bean is missing, the app fails fast *at startup*, not at the first request.
- **Trivially testable without Spring.** You just call `new OrderService(mockGateway, mockRepo)` in a plain unit test — no container needed.
- **Avoids hidden coupling.** A constructor with eight parameters *screams* that the class does too much; field injection hides that smell.

**Best practice:** since Spring 4.3, if a class has a **single constructor**, `@Autowired` on it is *implicit* — you can omit it. With Lombok, `@RequiredArgsConstructor` generates the constructor from `final` fields. Field injection (`@Autowired` on a field) is discouraged: it can't be `final`, hides dependencies, and forces reflection in tests.

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service // registered as a singleton bean named "orderService"
public class OrderService {

    private final PaymentGateway gateway;
    private final OrderRepository repo;

    // Single constructor → @Autowired is implicit. Constructor injection:
    // both deps are final, mandatory, and verified at startup.
    public OrderService(PaymentGateway gateway, OrderRepository repo) {
        this.gateway = gateway;
        this.repo = repo;
    }

    @Transactional
    public Order place(NewOrder cmd) {
        var order = Order.from(cmd);            // domain logic
        gateway.charge(order.total());           // collaborator, injected
        return repo.save(order);                 // collaborator, injected
    }
}
```

### @Bean and @Configuration — wiring objects you don't own

When you need to register a third-party object (say an HTTP client, an `ObjectMapper`, or a `RestClient`), you can't annotate its class. Instead, declare it in a `@Configuration` class:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

@Configuration
public class HttpClientConfig {

    // The method name ("paymentApiClient") becomes the bean name.
    // Method PARAMETERS are themselves injected by the container.
    @Bean
    RestClient paymentApiClient(PaymentProperties props) {
        return RestClient.builder()
                .baseUrl(props.baseUrl())       // type-safe config (see §4)
                .defaultHeader("Authorization", "Bearer " + props.apiKey())
                .build();
    }
}
```

A subtle but important rule: `@Configuration` classes are **CGLIB-proxied**, so if one `@Bean` method calls another (`return new Thing(otherBean())`), Spring intercepts the call and returns the *singleton*, not a fresh object. (With `@Configuration(proxyBeanMethods = false)` — sometimes used for a micro-optimization — that interception is off and each call constructs anew; only use it when `@Bean` methods don't call each other.)

### Bean scopes

The **scope** controls how many instances exist and their lifetime:

| Scope | Meaning | When to use |
|---|---|---|
| `singleton` (default) | One instance per container | Almost always — stateless services/repos |
| `prototype` | New instance every time it's requested | Stateful helpers you need fresh copies of |
| `request` | One per HTTP request (web only) | Per-request state |
| `session` | One per HTTP session (web only) | Per-user session state |
| `application` | One per `ServletContext` | App-wide web state |

```java
@Service
@Scope("prototype") // a fresh bean on every injection/lookup
public class ReportBuilder { /* accumulates mutable state */ }
```

**Gotcha:** injecting a `prototype` bean into a `singleton` gives you the prototype *once* (at the singleton's construction) — it does not re-create per call. If you need a fresh prototype each method call, inject an `ObjectProvider<ReportBuilder>` and call `.getObject()`.

### @Autowired, @Qualifier, @Primary

When *one* bean of a type exists, the container injects it unambiguously. When **multiple** beans implement the same interface, Spring can't choose — you must disambiguate:

- **`@Primary`** marks one bean as the default winner.
- **`@Qualifier("name")`** at the injection point names exactly which bean you want.

```java
public interface PaymentGateway { void charge(Money amount); }

@Component @Primary
class StripeGateway implements PaymentGateway { /* ... */ }

@Component("paypal")
class PaypalGateway implements PaymentGateway { /* ... */ }

@Service
class CheckoutService {
    private final PaymentGateway gateway;
    // Without a qualifier this would inject the @Primary (Stripe).
    // The qualifier overrides that to select the PayPal bean.
    CheckoutService(@Qualifier("paypal") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

You can also inject **all** implementations as a `List<PaymentGateway>` or a `Map<String, PaymentGateway>` (keyed by bean name) — handy for the strategy pattern.

### Bean lifecycle hooks

A bean can run logic after wiring (`@PostConstruct`) and before destruction (`@PreDestroy`). Use these to open/close resources. (Prefer constructor injection for the *dependencies* and `@PostConstruct` only for *initialization that needs all dependencies present*.)

```java
import jakarta.annotation.PostConstruct;  // jakarta, not javax!
import jakarta.annotation.PreDestroy;

@Component
class ConnectionWarmer {
    @PostConstruct void warm()  { /* prime caches/pools after wiring */ }
    @PreDestroy   void close()  { /* release resources on shutdown */ }
}
```

---

## 3. Setup: Initializr, Maven, the Main Class

### Spring Initializr — the canonical way to start

Never hand-assemble a Spring Boot project. Use **Spring Initializr** (start.spring.io, or your IDE's "New Spring Boot Project" wizard, which calls the same service). You pick the build tool, language, Spring Boot version, Java version, and the **starters** you want; it generates a ready-to-run project with the parent POM, a main class, and a test. The offline equivalent is the same `pom.xml` shown below — the Initializr is just a generator for it.

Key choices on the form: **Project** = Maven, **Language** = Java, **Spring Boot** = 3.5.x (a stable release, not a SNAPSHOT), **Packaging** = Jar, **Java** = 21. Typical starter set for a REST API: `Spring Web`, `Spring Data JPA`, `Validation`, `PostgreSQL Driver`, `Spring Boot DevTools`, `Spring Boot Actuator`, `Testcontainers`, and `Spring Security` once you're ready for auth.

### Project structure

Boot relies on a conventional layout. The most important convention: **your main class lives at the root package**, because component scanning starts there and walks *downward*. Put it too deep and Spring won't find your beans.

```
my-api/
├── pom.xml                       # Maven build + dependency management
├── mvnw, mvnw.cmd                # Maven Wrapper (use ./mvnw / mvnw.cmd — pins Maven version)
├── src/
│   ├── main/
│   │   ├── java/com/example/api/
│   │   │   ├── ApiApplication.java        # @SpringBootApplication — the root
│   │   │   ├── order/                     # package-by-feature (see §17)
│   │   │   │   ├── OrderController.java
│   │   │   │   ├── OrderService.java
│   │   │   │   ├── OrderRepository.java
│   │   │   │   └── Order.java
│   │   │   └── config/                    # cross-cutting @Configuration
│   │   └── resources/
│   │       ├── application.yml            # configuration
│   │       ├── application-dev.yml        # profile-specific
│   │       ├── db/migration/              # Flyway SQL migrations
│   │       └── static/, templates/        # static assets / views
│   └── test/java/com/example/api/         # mirrors main; tests live here
└── target/                        # build output (the fat jar lands here)
```

**Windows note:** use `mvnw.cmd` (the wrapper) rather than a globally-installed `mvn` so everyone builds with the same Maven version. In Git Bash you can run `./mvnw`.

### The pom.xml — parent, starters, the BOM

The `pom.xml` is the heart of a Maven build. Two things make it a *Spring Boot* POM: the **parent** (`spring-boot-starter-parent`, which imports the BOM and sets sane plugin/compiler defaults) and the **`spring-boot-maven-plugin`** (which repackages your jar into an executable fat jar and enables `mvn spring-boot:run`).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <!-- The PARENT imports the Spring Boot BOM: it pins versions for every
       Spring-managed dependency, sets the Java release, and configures
       plugins. Because of it, the dependencies below carry NO <version>. -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
    <relativePath/> <!-- look up the parent from the repository -->
  </parent>

  <groupId>com.example</groupId>
  <artifactId>api</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>api</name>

  <properties>
    <java.version>21</java.version>   <!-- compile & target Java 21 -->
  </properties>

  <dependencies>
    <!-- Web: embedded Tomcat + Spring MVC + Jackson. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Persistence: Spring Data JPA + Hibernate + HikariCP. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- Bean Validation (jakarta.validation + Hibernate Validator). -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <!-- Production-readiness endpoints (health, metrics). -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- DB migrations. -->
    <dependency>
      <groupId>org.flywaydb</groupId>
      <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
      <groupId>org.flywaydb</groupId>
      <artifactId>flyway-database-postgresql</artifactId>
    </dependency>
    <!-- JDBC driver: runtime-only (not needed to compile). -->
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
    <!-- DevTools: live restart while coding; never shipped to prod. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-devtools</artifactId>
      <scope>runtime</scope>
      <optional>true</optional>
    </dependency>
    <!-- Test slice: JUnit 5, AssertJ, Mockito, MockMvc, etc. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <!-- Real Postgres in integration tests (see §12). -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-testcontainers</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>postgresql</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Repackages into an executable fat jar; enables spring-boot:run. -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

**Gradle note:** the same project in Gradle (Kotlin DSL) applies the `org.springframework.boot` and `io.spring.dependency-management` plugins (the latter imports the BOM, so dependencies are version-less there too) and declares starters with `implementation("org.springframework.boot:spring-boot-starter-web")`. Gradle builds incrementally and is often faster on large multi-module projects; Maven's rigidity is an asset on teams that value uniformity. Either is fine.

### The main class & @SpringBootApplication

```java
package com.example.api;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiApplication {
    public static void main(String[] args) {
        // Bootstraps the ApplicationContext, runs auto-configuration,
        // starts the embedded Tomcat, and blocks until shutdown.
        SpringApplication.run(ApiApplication.class, args);
    }
}
```

`@SpringBootApplication` is a **meta-annotation** combining three:

- **`@SpringBootConfiguration`** — marks this as a configuration class (a specialized `@Configuration`).
- **`@EnableAutoConfiguration`** — turns on the classpath-driven auto-configuration machinery described in §1.
- **`@ComponentScan`** — scans this class's package and *below* for stereotypes. This is **why the main class must sit at the root package.**

### Running and DevTools

- `mvnw.cmd spring-boot:run` — run from source (fastest dev loop).
- `mvnw.cmd clean package` then `java -jar target/api-0.0.1-SNAPSHOT.jar` — run the built fat jar (closest to production).
- **DevTools** watches the classpath and **automatically restarts** the context when compiled classes change, disables template/static caching, and enables a live-reload trigger. It detects "dev" by its presence on the classpath and **excludes itself from the production jar** automatically. Great for the inner loop; never relied upon in production.

---

## 4. Configuration & Profiles

A production service must read its settings — database URLs, credentials, feature flags, timeouts — from **outside the code**, so the *same* artifact runs in dev, staging, and prod with different inputs. Spring Boot's configuration system is one of its best features: layered, type-safe, and profile-aware.

### application.yml / application.properties

Boot reads `application.yml` (or `.properties`) from `src/main/resources`. YAML is preferred for nesting; properties for flat simplicity. They're equivalent — `spring.datasource.url` in properties is the nested `spring: datasource: url:` in YAML.

```yaml
# application.yml — the base/default config (applies to all profiles)
spring:
  application:
    name: orders-api
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: orders_app
    password: ${DB_PASSWORD}        # ← read from an env var, never hard-code
  jpa:
    hibernate:
      ddl-auto: validate            # validate schema; NEVER 'update'/'create' in prod
    open-in-view: false             # disable OSIV — see §20 (important!)
    properties:
      hibernate.jdbc.batch_size: 50

server:
  port: 8080
  shutdown: graceful                # finish in-flight requests on stop (§15)

logging:
  level:
    org.hibernate.SQL: warn
```

### Externalized configuration & precedence

The same property can come from many sources; Boot defines a **precedence order** so a more specific source overrides a more general one. Highest wins. Simplified order (highest first): **command-line args** → **OS environment variables / `SPRING_APPLICATION_JSON`** → **profile-specific files** (`application-prod.yml`) → **the base `application.yml`** → **`@PropertySource`** → **defaults**. This is *the* mechanism for the twelve-factor rule "config in the environment": you bake reasonable defaults into `application.yml`, and override secrets and environment-specific values via env vars at deploy time (in [Docker](DOCKER_GUIDE.md), Kubernetes, or your CI). The env-var form uppercases and replaces dots/dashes with underscores: `spring.datasource.url` → `SPRING_DATASOURCE_URL`. This automatic mapping is **relaxed binding**.

### Profiles

A **profile** is a named group of configuration (and beans) activated for an environment. Conventionally `dev`, `test`, `prod`. Properties in `application-{profile}.yml` are layered on top of the base file when that profile is active. You activate via `spring.profiles.active=prod` (env var `SPRING_PROFILES_ACTIVE=prod` is the production-friendly form).

```yaml
# application-dev.yml — only when the 'dev' profile is active
spring:
  jpa:
    show-sql: true                  # echo SQL in dev for debugging
    hibernate:
      ddl-auto: create-drop         # fine in dev; catastrophic in prod
logging:
  level:
    com.example: debug
```

Beans can also be profile-scoped: `@Profile("dev")` on a `@Component`/`@Bean` includes it only under that profile — useful for, say, a fake email sender in dev versus a real one in prod.

```java
@Configuration
class MailConfig {
    @Bean @Profile("prod")  EmailSender smtp()    { return new SmtpEmailSender(); }
    @Bean @Profile("!prod") EmailSender logOnly()  { return new LoggingEmailSender(); }
}
```

### @Value vs @ConfigurationProperties

**`@Value("${...}")`** injects a single property into a field/parameter. It's fine for one-off values but doesn't scale: no grouping, no validation, no IDE autocomplete, weak typing.

```java
@Component
class TokenService {
    // SpEL placeholder; ":3600" supplies a default if the key is absent.
    @Value("${security.jwt.ttl-seconds:3600}")
    private long ttlSeconds;
}
```

**`@ConfigurationProperties`** is the production-grade approach: bind a whole group of related properties to a **type-safe record/class**, with validation. This is strongly preferred for anything beyond a single value — it's discoverable, validated, and refactor-safe.

```java
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

// Binds all keys under "security.jwt.*". Records make this immutable.
@Validated
@ConfigurationProperties(prefix = "security.jwt")
public record JwtProperties(
        @NotBlank String issuer,
        @NotBlank String secret,           // must be present and non-empty
        @Positive long ttlSeconds           // validated at startup
) {}
```

Enable binding by adding `@EnableConfigurationProperties(JwtProperties.class)` on a config class, or annotate the record with `@ConfigurationProperties` and add `@ConfigurationPropertiesScan` on the main class. With validation, a missing/blank secret makes the app **fail to start** rather than fail mysteriously later — exactly what you want. **Relaxed binding** means `security.jwt.ttlSeconds`, `ttl-seconds`, and `TTL_SECONDS` all bind to `ttlSeconds`.

### Secrets

**Never commit secrets.** Use `${ENV_VAR}` placeholders backed by environment variables (or a secrets manager / Vault / Kubernetes Secrets injected as env vars). Spring Boot 3 also supports **config trees** (each secret as a file, the Kubernetes-mounted-secret pattern) and **Docker Compose / Testcontainers integration** for dev. Keep `application.yml` free of any real credential; the `${DB_PASSWORD}` pattern above is the baseline. See §19 for hardening.

---

## 5. Building REST APIs

This is the bread and butter: HTTP in, JSON out. Spring MVC maps HTTP requests to controller methods and (via Jackson) serializes return values to JSON automatically.

### @RestController and request mapping

A `@RestController` is a `@Controller` whose every method's return value is written *directly* to the response body (serialized to JSON), rather than resolved as a view name. `@RequestMapping` (and its HTTP-method shortcuts) maps URL patterns to methods.

| Annotation | Maps | Notes |
|---|---|---|
| `@GetMapping` | HTTP GET | safe, idempotent, cacheable — reads |
| `@PostMapping` | HTTP POST | create / non-idempotent actions |
| `@PutMapping` | HTTP PUT | full replace (idempotent) |
| `@PatchMapping` | HTTP PATCH | partial update |
| `@DeleteMapping` | HTTP DELETE | remove (idempotent) |
| `@RequestMapping` | any (configurable) | class-level base path + method-level paths |

### Binding the request: path, query, body

- **`@PathVariable`** binds a templated URL segment: `/orders/{id}` → `@PathVariable Long id`.
- **`@RequestParam`** binds a query-string parameter: `?status=PAID` → `@RequestParam OrderStatus status`. Mark optional ones with `required = false` or give a default.
- **`@RequestBody`** binds and deserializes the JSON request body into an object (a DTO).
- **`ResponseEntity<T>`** gives full control over status code, headers, and body. Returning a bare object implies `200 OK`; returning `ResponseEntity` lets you say `201 Created` with a `Location` header, `204 No Content`, etc.

### DTOs vs entities — a non-negotiable boundary

**Never expose your JPA `@Entity` directly in the API.** This is one of the most important architecture rules in this guide. Reasons:

1. **Security/over-exposure.** An entity may carry fields you must not leak (password hashes, internal flags) and, on input, accepting an entity lets a client set fields they shouldn't (mass-assignment).
2. **Coupling.** Tying your wire format to your DB schema means a column rename breaks your API contract.
3. **Lazy-loading hazards.** Serializing an entity with lazy associations triggers `LazyInitializationException` or accidental N+1 queries during JSON serialization (§7, §20).

The fix is **DTOs** (Data Transfer Objects): small, purpose-built shapes for *input* (`CreateOrderRequest`) and *output* (`OrderResponse`). **Records are perfect for DTOs** — immutable, concise, with built-in `equals`/`hashCode`. Map between entity and DTO in the service layer (manually or with MapStruct, §10).

```java
package com.example.api.order;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.net.URI;
import java.util.List;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.data.domain.*;

// ---- DTOs (records): input and output shapes, decoupled from the entity ----
record CreateOrderRequest(
        @NotNull Long customerId,
        @NotEmpty List<@Valid LineItem> items   // @Valid cascades into elements
) {}
record LineItem(@NotBlank String sku, @Positive int quantity) {}

record OrderResponse(Long id, String status, java.math.BigDecimal total) {}

@RestController
@RequestMapping("/api/orders")   // class-level base path
class OrderController {

    private final OrderService service;
    OrderController(OrderService service) { this.service = service; }

    // GET /api/orders/42  → 200 with the order, or 404 (handled in §6)
    @GetMapping("/{id}")
    OrderResponse getOne(@PathVariable Long id) {
        return service.findById(id);   // service returns a DTO, never an entity
    }

    // GET /api/orders?status=PAID&page=0&size=20&sort=createdAt,desc
    // Spring builds the Pageable from the query params automatically.
    @GetMapping
    Page<OrderResponse> list(@RequestParam(required = false) String status,
                             Pageable pageable) {
        return service.search(status, pageable);
    }

    // POST /api/orders  → 201 Created with a Location header to the new resource
    @PostMapping
    ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        OrderResponse created = service.create(req);   // @Valid triggers validation (§6)
        URI location = URI.create("/api/orders/" + created.id());
        return ResponseEntity.created(location).body(created);  // 201 + Location
    }

    // DELETE /api/orders/42 → 204 No Content
    @DeleteMapping("/{id}")
    ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Status codes & content negotiation

Use HTTP status codes correctly: `200` read OK, `201` created, `204` no content, `400` bad input, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` semantically invalid, `500` server error. Spring negotiates content via the `Accept` header and `produces`/`consumes` attributes; with Jackson on the classpath, JSON is the default. You can return XML by adding the Jackson XML module and negotiating on `Accept`.

### Calling other services: RestClient

A REST service often *calls* other REST services. The modern client is **`RestClient`** (Spring Framework 6.1+): fluent, synchronous, and built on the same infrastructure as the reactive `WebClient` but without the reactive types. It supersedes the verbose legacy `RestTemplate` (still supported, but don't start new code with it). On virtual threads (§13), blocking calls through `RestClient` scale fine, so you rarely need the reactive `WebClient` just for outbound calls.

```java
import org.springframework.web.client.RestClient;

// Inject the bean defined in §2's HttpClientConfig.
record CustomerDto(Long id, String name, String email) {}

class CustomerGateway {
    private final RestClient client;
    CustomerGateway(RestClient paymentApiClient) { this.client = paymentApiClient; }

    CustomerDto fetch(long id) {
        return client.get()
                .uri("/customers/{id}", id)        // path variable, safely encoded
                .retrieve()
                .onStatus(s -> s.value() == 404,    // map a 404 to a domain exception
                          (req, res) -> { throw new ResourceNotFoundException("customer"); })
                .body(CustomerDto.class);           // deserialize the JSON response
    }
}
```

For typed, declarative clients, Spring's **HTTP Interface** (`@HttpExchange` on an interface, backed by a `RestClient`) lets you define remote calls as Java methods — similar to Feign, but built in.

### ⚡ Version note — API versioning in Boot 4.0

**Spring Framework 7 / Boot 4.0** adds **first-class API versioning** — declaring and routing by version via header, path, or media-type — as a built-in concern rather than something you hand-roll with separate controllers or path prefixes (see §17). Until then, URI versioning (`/api/v1/...`) remains the simplest approach.

---

## 6. Validation & Error Handling

A robust API rejects bad input with a clear, structured error *before* business logic runs, and turns every exception into a consistent, machine-readable response. Spring gives you both.

### Bean Validation (jakarta.validation)

**Bean Validation** is a Jakarta standard (implemented by Hibernate Validator, pulled in by `spring-boot-starter-validation`). You annotate DTO fields with **constraints**; placing **`@Valid`** on a `@RequestBody` parameter triggers validation, and a failure throws `MethodArgumentNotValidException` *before your handler body runs*.

| Constraint | Asserts |
|---|---|
| `@NotNull` | not null |
| `@NotBlank` | string non-null and not whitespace-only |
| `@NotEmpty` | collection/string not empty |
| `@Size(min, max)` | length/size bounds |
| `@Min` / `@Max` / `@Positive` / `@PositiveOrZero` | numeric bounds |
| `@Email` | valid email format |
| `@Pattern(regexp=...)` | regex match |
| `@Past` / `@Future` | temporal bounds |
| `@Valid` (on a field) | **cascade** validation into a nested object/collection |

Apply constraints to **input DTOs**, not entities, so validation reflects the API contract. For cross-field rules ("end date after start date"), write a custom class-level constraint or validate in the service.

### @ControllerAdvice & @ExceptionHandler

Scattering try/catch through controllers is unmaintainable. Instead, centralize error handling in one **`@RestControllerAdvice`** class containing **`@ExceptionHandler`** methods — one per exception type. This is the global, declarative error layer; it keeps controllers clean and guarantees a *uniform* error shape across the whole API.

### Problem Details (RFC 9457)

The modern, standard error format is **Problem Details for HTTP APIs** (RFC 9457, formerly 7807): a JSON body with `type`, `title`, `status`, `detail`, and `instance` fields, served as `application/problem+json`. Spring supports it first-class via `ProblemDetail` and `ErrorResponse`. **Enable it globally** with `spring.mvc.problemdetails.enabled=true`, which makes Spring's built-in exceptions produce problem+json automatically; then customize your own exceptions in the advice.

```java
package com.example.api.error;

import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import java.util.*;

// A domain exception your services throw for "not found".
class ResourceNotFoundException extends RuntimeException {
    ResourceNotFoundException(String msg) { super(msg); }
}

@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody, applies to all controllers
class GlobalExceptionHandler {

    // 404 as RFC 9457 problem+json.
    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource not found");
        pd.setType(URI.create("https://api.example.com/errors/not-found"));
        return pd; // serialized as application/problem+json
    }

    // 400 for bean-validation failures: collect per-field messages.
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handleInvalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "Validation failed");
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(fe -> errors.put(fe.getField(), fe.getDefaultMessage()));
        pd.setProperty("errors", errors); // custom field in the problem document
        return pd;
    }

    // Last-resort 500: log the real cause; never leak stack traces to clients.
    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex) {
        // log.error("Unhandled", ex);  // <- ALWAYS log here
        return ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "Unexpected error");
    }
}
```

**Security note:** the catch-all `500` handler must **log the real exception but return a generic message** — leaking stack traces, SQL, or class names to clients is an information-disclosure vulnerability (§19).

---

## 7. Data Access with Spring Data JPA

Most Spring Boot services talk to a relational database, and the dominant approach is **Spring Data JPA** over **Hibernate**. Understand the layers: **JPA** is the Jakarta *specification* (annotations like `@Entity`, the `EntityManager` API); **Hibernate** is the *implementation* Boot uses; **Spring Data JPA** is a productivity layer that generates repository implementations for you. See [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) for schema modeling and [PostgreSQL](POSTGRESQL_GUIDE.md) for the database itself.

### Entities and relationships

An **`@Entity`** maps a class to a table; its fields map to columns. Use `jakarta.persistence.*` (not `javax`). Model relationships with `@ManyToOne`, `@OneToMany`, `@ManyToMany`, `@OneToOne`. **The single most important default to override: associations should be `LAZY`.** `@ManyToOne` and `@OneToOne` default to `EAGER`, which silently loads related rows on every query and is a primary cause of the N+1 problem — set `fetch = FetchType.LAZY` explicitly.

```java
package com.example.api.order;

import jakarta.persistence.*;     // jakarta — NOT javax
import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // DB auto-increment / serial
    private Long id;

    @Column(nullable = false)
    private String status;

    @Column(nullable = false)
    private BigDecimal total;

    private Instant createdAt = Instant.now();

    // Many orders belong to one customer. Force LAZY: do not load the customer
    // unless explicitly needed (avoids surprise queries).
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    // One order has many line items. cascade + orphanRemoval ties their
    // lifecycle to the order's. Still LAZY by default for @OneToMany.
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    protected Order() {}  // JPA requires a no-arg constructor (can be protected)
    // ... business constructors, getters; keep setters minimal ...
}
```

**Entity hygiene:** prefer a no-arg protected constructor + business methods over public setters; never put JPA entities in `equals`/`hashCode` by mutable fields. Entities are *not* DTOs (§5).

### JpaRepository — repositories for free

Declare an **interface** extending `JpaRepository<Entity, IdType>` and Spring Data **generates the implementation at runtime**. You get `save`, `findById`, `findAll`, `delete`, `count`, paging, and sorting with zero code. Beyond that, three ways to express queries:

1. **Derived query methods** — Spring parses the *method name* into a query: `findByStatusAndTotalGreaterThan(String status, BigDecimal min)`. Great for simple finders; unreadable past ~3 conditions.
2. **`@Query`** — write JPQL (or native SQL with `nativeQuery = true`) when names get unwieldy or you need joins/projections.
3. **Specifications / `JpaSpecificationExecutor`** — compose dynamic predicates programmatically (for filter-by-anything search).

```java
package com.example.api.order;

import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import java.math.BigDecimal;
import java.util.*;

public interface OrderRepository extends JpaRepository<Order, Long> {

    // 1) Derived query: parsed from the method name.
    List<Order> findByStatus(String status);

    // Paged + sorted finder: pass a Pageable, get a Page back.
    Page<Order> findByStatus(String status, Pageable pageable);

    // 2) JPQL with a named parameter. JOIN FETCH solves N+1 (see below).
    @Query("""
           select o from Order o
           join fetch o.items
           where o.customer.id = :customerId
           """)
    List<Order> findWithItemsByCustomer(@Param("customerId") Long customerId);

    // Projection straight into a DTO via a JPQL constructor expression —
    // selects only needed columns, never loads the full entity graph.
    @Query("""
           select new com.example.api.order.OrderResponse(o.id, o.status, o.total)
           from Order o where o.total > :min
           """)
    List<OrderResponse> summariesAbove(@Param("min") BigDecimal min);
}
```

### Pagination & sorting

For any list endpoint that can grow, **never return all rows** — page. Accept a `Pageable` (Spring builds it from `?page=&size=&sort=` query params), return a `Page<T>` (which carries total count and total pages) or a lighter `Slice<T>` (no count query — cheaper when you only need "is there a next page?").

### Transactions — @Transactional

A **transaction** groups operations so they all commit or all roll back. Annotate **service methods** (not repositories or controllers) with `@Transactional`. Reads can use `@Transactional(readOnly = true)` (a hint that enables optimizations and prevents accidental writes). Full propagation/isolation/rollback semantics are in §10 — the key rule here: keep transactions at the service boundary and as short as possible.

### The N+1 problem — the canonical JPA performance bug

The **N+1 problem** is the most common and most damaging JPA performance mistake. You load **N** orders (1 query); then, for each order, accessing `order.getItems()` (a lazy association) fires *another* query — **N more**. A page of 100 orders becomes 101 queries. It often hides until production load reveals it.

Fixes, in order of preference:

- **`JOIN FETCH`** in a JPQL query (as in `findWithItemsByCustomer` above) — load the association in the *same* query.
- **`@EntityGraph`** on a repository method — declaratively specify which associations to fetch eagerly *for that query only*.
- **Batch fetching** — `@BatchSize(size = N)` / `hibernate.default_batch_fetch_size` turns N lazy loads into ceil(N/size) `IN`-queries.
- **DTO projections** (constructor expressions / interface projections) — select exactly the columns you need, sidestepping the entity graph entirely. Best for read-heavy endpoints.

```java
import org.springframework.data.jpa.repository.EntityGraph;

public interface OrderRepo2 extends JpaRepository<Order, Long> {
    // Fetch 'items' eagerly ONLY for this query — no N+1, no global EAGER.
    @EntityGraph(attributePaths = "items")
    Page<Order> findAll(Pageable pageable);
}
```

**Guard:** enable `LazyInitializationException`-on-misuse by keeping `open-in-view: false` (§20). And set `hibernate.jdbc.batch_size` (e.g. 50) so inserts/updates batch.

### Projections

A **projection** returns a subset of columns shaped to your needs without loading entities:

- **Interface projections** — declare an interface with getters; Spring proxies it: `interface OrderSummary { Long getId(); BigDecimal getTotal(); }` returned from a query method.
- **Class/record (DTO) projections** — JPQL `new` constructor expressions (above) or `@Query` returning a record. Preferred for read APIs because they're explicit and immutable.

### Migrations: Flyway / Liquibase

**Never let Hibernate manage your production schema** (`ddl-auto: update` is a footgun — it can silently alter or miss changes). Instead, version your schema with a **migration tool**. **Flyway** uses plain, ordered SQL files (`V1__init.sql`, `V2__add_orders.sql`) in `src/main/resources/db/migration`; it records applied versions in a `flyway_schema_history` table and runs pending migrations **on startup**. **Liquibase** is the XML/YAML/SQL alternative with more abstraction (database-agnostic changesets, rollbacks). For most teams, **Flyway with raw SQL** is the simplest, most transparent choice. Set `ddl-auto: validate` so Hibernate *checks* the schema matches your entities but never changes it.

```sql
-- src/main/resources/db/migration/V1__create_orders.sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    status      VARCHAR(32) NOT NULL,
    total       NUMERIC(12,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

---

## 8. The Persistence Landscape: JPA, JdbcClient, R2DBC, Pooling

JPA is the default, not the only option. Choosing the right access style per use case is a hallmark of a senior Spring developer.

### When JPA shines and when it hurts

JPA/Hibernate excels at **rich domain models with relationships, change tracking, and caching** — CRUD over an object graph where the ORM's dirty-checking and cascade features earn their keep. It *hurts* for **complex reporting queries, bulk operations, and read-heavy hot paths**, where the abstraction adds overhead and the generated SQL is hard to control. There, drop to JDBC.

### JdbcClient — the modern, fluent JDBC

Spring Framework 6.1 introduced **`JdbcClient`**, a clean, fluent API over plain JDBC (superseding the verbose `JdbcTemplate` for most needs). You write the SQL; Spring handles connection management, parameter binding, and row mapping. It's perfect for precise queries, reports, and bulk operations — full SQL control with none of JPA's surprises.

```java
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;
import java.math.BigDecimal;
import java.util.*;

@Repository
public class OrderReadModel {
    private final JdbcClient jdbc;
    OrderReadModel(JdbcClient jdbc) { this.jdbc = jdbc; }   // auto-configured bean

    public Optional<OrderResponse> findSummary(long id) {
        return jdbc.sql("""
                select id, status, total from orders where id = :id
                """)
                .param("id", id)
                .query(OrderResponse.class)   // maps columns to the record
                .optional();
    }

    public List<OrderResponse> topByValue(BigDecimal min, int limit) {
        return jdbc.sql("""
                select id, status, total from orders
                where total >= :min order by total desc limit :limit
                """)
                .param("min", min).param("limit", limit)
                .query(OrderResponse.class).list();
    }
}
```

**Mixing JPA and JdbcClient in one app is normal and encouraged:** JPA for writes and the domain model, JdbcClient for read models and reports. They share the same datasource and transactions.

### R2DBC — reactive data access (note)

**R2DBC** is the reactive, non-blocking database API used with **Spring WebFlux** (§18). It returns `Mono`/`Flux` instead of blocking on a thread per query. Reach for it only if you've committed to a fully reactive stack for extreme concurrency on limited threads — and note that **virtual threads (Java 21)** now deliver much of reactive's scalability with ordinary blocking, imperative code (§13, §18), so for most teams plain JPA/JdbcClient on virtual threads is the simpler win. R2DBC is not a drop-in for JPA; it has no ORM.

### Connection pooling — HikariCP

Opening a database connection is expensive, so apps reuse a **pool** of them. Spring Boot auto-configures **HikariCP** (the fastest, default pool) whenever a datasource is present. The single most important setting is **`maximum-pool-size`**, and the common mistake is setting it *too high*. A database has finite capacity; a pool larger than the DB can serve just queues work and increases latency. A good starting heuristic is small — often `(2 × CPU cores) + effective spindle count`, frequently in the **10–20** range per instance, tuned with metrics. See [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md) for the DB-side `max_connections` budget across all app instances.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 15        # bounded: do NOT crank this to 100
      minimum-idle: 5
      connection-timeout: 3000     # ms to wait for a connection before failing
      max-lifetime: 1800000        # 30 min; under any DB/proxy idle timeout
      pool-name: orders-pool
```

### Multiple datasources

Some apps talk to two databases (e.g. a primary write DB and a read replica, or two bounded contexts). You define each `DataSource`, `EntityManagerFactory`, and `TransactionManager` as explicit `@Bean`s, mark one `@Primary`, and partition repositories by package with `@EnableJpaRepositories(basePackages=..., entityManagerFactoryRef=..., transactionManagerRef=...)`. It's more wiring than the single-datasource autoconfig, so do it only when genuinely needed.

---

## 9. Spring Security

Security is mandatory for any real service, and Spring Security is the comprehensive, battle-tested framework for it. It's intimidating at first because it's deep — but the mental model is small.

### The filter chain

Spring Security works as a **chain of servlet filters** (the `SecurityFilterChain`) that every request passes through *before* reaching your controllers. Each filter has one job: one extracts credentials, one checks authentication, one enforces authorization, one handles CSRF, one manages CORS, and so on. **Authentication** answers "who are you?"; **authorization** answers "are you allowed to do this?". Understanding that requests flow through this ordered chain is the key to reasoning about security.

The moment `spring-boot-starter-security` is on the classpath, *everything* is locked down by default (HTTP Basic, a generated password) — secure-by-default. You then *open up* what should be public and configure how authentication works.

### The modern lambda DSL

Configure security by declaring a **`SecurityFilterChain` bean** using the **lambda DSL** (the old `WebSecurityConfigurerAdapter` is **removed** — any tutorial using it is obsolete). You compose authorization rules, the auth mechanism, CSRF, CORS, and session policy fluently.

```java
package com.example.api.security;

import org.springframework.context.annotation.*;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Authorization rules, evaluated top-down (first match wins).
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()         // public login
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")   // role-gated
                .anyRequest().authenticated())                       // everything else: auth

            // Stateless JWT API: no server session, so disable CSRF (see below)
            // and don't create sessions.
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS))

            // Validate JWTs as a resource server (config in §below).
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))

            .cors(Customizer.withDefaults());   // honor the CorsConfigurationSource bean
        return http.build();
    }

    // Password hashing: BCrypt (adaptive, salted). Never store plaintext.
    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();   // or Argon2/SCrypt for higher cost
    }
}
```

### JWT & OAuth2 resource server

For stateless REST APIs, the dominant pattern is **JWT** (JSON Web Token): the client authenticates once, receives a signed token, and sends it on every request as `Authorization: Bearer <token>`. The server **validates the signature and claims** on each request — no server-side session. Spring Security's **OAuth2 Resource Server** (`spring-boot-starter-oauth2-resource-server`) does this validation for you; you just point it at your issuer's signing key (a JWK Set URI from an external IdP like Keycloak/Auth0, or a configured secret for symmetric tokens).

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # For an external IdP, validate signatures against its public keys:
          issuer-uri: https://auth.example.com/realms/orders
          # (or jwk-set-uri:, or for HMAC tokens a shared secret-key)
```

If you issue your *own* HMAC-signed tokens, configure a `JwtDecoder` bean with your secret and (recommended) rotate keys. Map JWT scopes/claims to Spring authorities with a `JwtAuthenticationConverter`.

### Method security — @PreAuthorize

Beyond URL rules, you can secure **individual methods** with expression-based annotations after enabling `@EnableMethodSecurity`. This puts authorization next to the business logic it protects and supports rich SpEL (including checks against method arguments and the principal).

```java
import org.springframework.security.access.prepost.PreAuthorize;

@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN')")
    public void cancelAny(Long orderId) { /* admin-only */ }

    // Ownership check: only the order's owner (or an admin) may read it.
    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.name")
    public OrderResponse readOwned(Long orderId, String ownerId) { /* ... */ }
}
```

### CORS and CSRF — when each matters

- **CORS** (Cross-Origin Resource Sharing) governs which *browser origins* may call your API. Configure it explicitly (allowed origins, methods, headers) — never reflect `*` with credentials. Provide a `CorsConfigurationSource` bean and enable `.cors()`.
- **CSRF** (Cross-Site Request Forgery) protection matters for **cookie/session-based** browser apps; for a **stateless, token-in-header API** (JWT in `Authorization`), CSRF is not applicable and is disabled (as above). **Do not** blindly disable CSRF for a session-cookie app — that reintroduces the vulnerability.

```java
@Bean
org.springframework.web.cors.CorsConfigurationSource corsSource() {
    var cfg = new org.springframework.web.cors.CorsConfiguration();
    cfg.setAllowedOrigins(java.util.List.of("https://app.example.com")); // explicit
    cfg.setAllowedMethods(java.util.List.of("GET", "POST", "PUT", "DELETE"));
    cfg.setAllowedHeaders(java.util.List.of("Authorization", "Content-Type"));
    cfg.setAllowCredentials(true);
    var src = new org.springframework.web.cors.UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/api/**", cfg);
    return src;
}
```

### Roles vs authorities

An **authority** is a granted permission string; a **role** is an authority by convention prefixed `ROLE_`. `hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`; `hasAuthority("orders:write")` checks the literal string. Use fine-grained authorities for real permission systems; roles for coarse groupings.

---

## 10. The Service Layer & Transactions

### Layered architecture

A maintainable Spring app separates responsibilities into **layers**, each with one job and a one-directional dependency (controller → service → repository):

- **Controller (`@RestController`)** — HTTP only. Parse/validate input, call a service, shape the response. **No business logic, no DB access.**
- **Service (`@Service`)** — the business logic and the **transaction boundary**. Orchestrates repositories and other services; maps entities ↔ DTOs. This is where domain rules live.
- **Repository (`@Repository` / `JpaRepository`)** — persistence only. No business rules.

Keeping these strict pays off enormously: business logic is reusable (across REST, scheduled jobs, messaging) and unit-testable without HTTP or a database, and each layer can change independently.

### @Transactional semantics

`@Transactional` on a service method wraps it in a database transaction: commit on normal return, **rollback on a runtime exception**. Three attributes you must understand:

- **Propagation** — what happens if a transaction is already running. `REQUIRED` (default): join the existing one. `REQUIRES_NEW`: suspend the current and start an independent one (commits/rolls back on its own — e.g. for audit logging that must persist even if the main work fails). `NESTED`, `SUPPORTS`, `MANDATORY`, `NEVER` exist for special cases.
- **Isolation** — how concurrent transactions see each other's uncommitted data (`READ_COMMITTED` is the common default; `REPEATABLE_READ`/`SERIALIZABLE` trade concurrency for stronger guarantees). See [PostgreSQL](POSTGRESQL_GUIDE.md) for the database's MVCC behavior.
- **Rollback rules** — by default Spring rolls back on **unchecked** (`RuntimeException`/`Error`) but **not on checked** exceptions. To roll back on a checked exception, set `@Transactional(rollbackFor = SomeCheckedException.class)`. This trips up many developers.

```java
@Service
public class TransferService {

    private final AccountRepository accounts;
    private final AuditRepository audit;
    TransferService(AccountRepository a, AuditRepository au) { accounts = a; audit = au; }

    @Transactional   // both updates commit together, or neither does
    public void transfer(Long from, Long to, BigDecimal amount) {
        accounts.debit(from, amount);
        accounts.credit(to, amount);   // if this throws, the debit rolls back too
    }

    // The audit row must survive even if the caller's transaction rolls back:
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAudit(String event) { audit.save(new AuditEntry(event)); }
}
```

**The self-invocation gotcha (critical):** `@Transactional` works via a **proxy** that wraps your bean. If a method calls *another* `@Transactional` method on **the same instance** (`this.other()`), the call bypasses the proxy and the annotation is **ignored**. The same applies to `@Async`, `@Cacheable`, and other proxy-based features. Fix: move the inner method to a *separate bean*, or self-inject and call through the proxy. (Covered again in §20 — it bites everyone once.)

### DTO mapping with MapStruct

Mapping entities to DTOs by hand is repetitive and error-prone. **MapStruct** generates the mapping code at **compile time** (no reflection, fast, debuggable) from an interface you annotate. It's the standard choice for non-trivial mapping.

```java
import org.mapstruct.*;

@Mapper(componentModel = "spring")   // generates a Spring @Component implementation
public interface OrderMapper {
    OrderResponse toResponse(Order entity);          // fields matched by name
    List<OrderResponse> toResponses(List<Order> entities);

    @Mapping(target = "id", ignore = true)           // DB assigns the id
    Order toEntity(CreateOrderRequest req);
}
```

Add the `mapstruct` and `mapstruct-processor` dependencies; the generated `OrderMapperImpl` is injectable like any bean. For simple records, manual mapping (a static factory on the DTO) is also perfectly fine — don't add MapStruct for two fields.

---

## 11. Aspect-Oriented Programming (AOP)

### What AOP is and why it exists

Some concerns — **logging, metrics, security checks, transactions, caching, retry** — cut *across* many methods and classes. Scattering that code into every method is noise and duplication. **Aspect-Oriented Programming** lets you write such a **cross-cutting concern once**, as an **aspect**, and declaratively apply it to many points in your code without touching them. In fact, **`@Transactional`, `@Cacheable`, `@Async`, and `@PreAuthorize` are all implemented with Spring AOP** — understanding AOP demystifies how those "magic" annotations work.

The mechanism: Spring creates a **proxy** around your bean (a JDK dynamic proxy if it implements an interface, otherwise a CGLIB subclass). The proxy intercepts method calls and runs your "advice" before/after/around the real method. (This proxy mechanism is exactly *why* the self-invocation gotcha in §10/§20 exists.)

### Vocabulary

| Term | Meaning |
|---|---|
| **Aspect** | A class (`@Aspect`) bundling cross-cutting behavior |
| **Join point** | A point where advice can run (a method execution) |
| **Pointcut** | An expression selecting which join points (which methods) |
| **Advice** | The action: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around` |

```java
package com.example.api.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.slf4j.*;
import org.springframework.stereotype.Component;

@Aspect      // declares this bean as an aspect
@Component   // ...and registers it in the container
public class TimingAspect {
    private static final Logger log = LoggerFactory.getLogger(TimingAspect.class);

    // @Around wraps the target: you decide when (and whether) to call it.
    // Pointcut: any method in any class under the ..service.. packages.
    @Around("execution(* com.example.api..service..*(..))")
    public Object timeIt(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                 // invoke the real method
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {} ms", pjp.getSignature().toShortString(), ms);
        }
    }
}
```

Custom annotations + pointcuts let you build reusable behaviors (e.g. an `@Audited` annotation whose aspect writes audit logs). **Use AOP judiciously** — overuse makes control flow hard to follow; reserve it for genuinely cross-cutting, generic concerns.

---

## 12. Testing

A production service needs a fast, trustworthy test suite. Spring Boot's testing support is excellent because it lets you load *just enough* of the application for each test.

### The testing pyramid in Spring

- **Unit tests** — plain JUnit 5 + Mockito, *no Spring context*. Test a service in isolation by passing mock collaborators to its constructor. Fastest; the bulk of your tests. (Constructor injection from §2 is what makes this trivial.)
- **Slice tests** — load a *thin* slice of the context for one layer: `@WebMvcTest` (controllers + MVC, no DB), `@DataJpaTest` (repositories + an in-memory or Testcontainers DB, no web). Fast, focused integration.
- **Full integration tests** — `@SpringBootTest` loads the whole context (optionally with a running server). Slowest; reserve for end-to-end flows.

### Unit test with Mockito

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)   // no Spring context — pure JUnit + Mockito
class OrderServiceTest {
    @Mock OrderRepository repo;        // mock collaborator
    @InjectMocks OrderService service; // constructor-injected with the mock

    @Test
    void createsAndSaves() {
        when(repo.save(any())).thenAnswer(i -> i.getArgument(0));
        var resp = service.create(new CreateOrderRequest(1L, List.of()));
        assertThat(resp).isNotNull();
        verify(repo).save(any());      // assert the interaction happened
    }
}
```

### Web slice test with MockMvc

`@WebMvcTest` loads only the web layer (the named controller, MVC infrastructure, JSON), mocking the service so no DB is touched. **`MockMvc`** drives requests without a real network. **⚡ Version note:** `@MockBean`/`@SpyBean` were **deprecated in Boot 3.4** in favor of `@MockitoBean`/`@MockitoSpyBean` — use the new annotations on 3.4+.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OrderController.class)     // only this controller + MVC, no DB
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;  // Boot 3.4+ (was @MockBean)

    @Test
    void rejectsInvalidBody() throws Exception {
        mvc.perform(post("/api/orders")
                .contentType("application/json")
                .content("{}"))                  // missing required fields
           .andExpect(status().isBadRequest());  // validation kicks in (§6)
    }
}
```

### Repository slice with @DataJpaTest

`@DataJpaTest` configures just JPA, a `TestEntityManager`, and a transactional (rolled-back) test. By default it swaps an in-memory DB — but **test against the real database** so SQL dialect and constraints match production. That's what Testcontainers is for.

### Testcontainers — real dependencies in tests

**Testcontainers** spins up real services (PostgreSQL, Redis, Kafka) in **Docker** containers for the test run, then tears them down. This eliminates the "passes on H2, fails on Postgres" class of bugs — your tests hit the *same* database engine as production. Boot 3.1+ integrates it cleanly via `@ServiceConnection`, which auto-wires the container's URL/credentials into the datasource — no manual property plumbing.

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.DynamicPropertyRegistry; // (alt approach)
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.*;
import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use the container, not H2
@Testcontainers
class OrderRepositoryIT {

    @Container
    @ServiceConnection   // auto-binds this container to spring.datasource.* — no manual props
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @org.springframework.beans.factory.annotation.Autowired
    OrderRepository repo;

    @Test
    void persistsAndReads() {
        var saved = repo.save(new Order(/* ... */));
        assertThat(repo.findById(saved.getId())).isPresent();
    }
}
```

**Best practice:** name integration tests `*IT.java` and run them with the Failsafe plugin (separate from fast unit tests via Surefire), so CI can run unit tests on every push and the slower IT suite on merge.

---

## 13. Async, Scheduling, Virtual Threads & Messaging

### @Async — offload work to another thread

Annotate a method `@Async` (with `@EnableAsync` on a config) and Spring runs it on a separate thread, returning immediately (or a `CompletableFuture`). Use it for fire-and-forget work (sending an email, a webhook) so the request thread isn't blocked. **It's proxy-based, so the self-invocation gotcha applies** (call it from another bean). Always configure a bounded `TaskExecutor` — the default can be unbounded.

```java
@Service
public class NotificationService {
    @Async   // runs on a task-executor thread; method returns immediately
    public void sendWelcome(String email) { /* slow I/O, off the request thread */ }
}
```

### @Scheduled — cron and fixed-rate jobs

With `@EnableScheduling`, `@Scheduled` runs methods on a timer: `fixedRate` (every N ms), `fixedDelay` (N ms after the previous finishes), or a `cron` expression. Good for cleanup, polling, report generation. **In a multi-instance deployment, every instance runs the schedule** — use a distributed lock (ShedLock) so a job runs once cluster-wide.

```java
@Component
public class Maintenance {
    // cron: second minute hour day-of-month month day-of-week
    @Scheduled(cron = "0 0 3 * * *")   // 03:00 every day
    public void purgeExpired() { /* delete stale rows */ }

    @Scheduled(fixedDelay = 60_000)    // 60s after the previous run completes
    public void heartbeat() { /* ... */ }
}
```

### Virtual threads (Java 21) in Boot

**Virtual threads** (Project Loom, Java 21) are lightweight threads managed by the JVM — you can have *millions* cheaply. The payoff for a Spring web app is enormous: the classic "thread-per-request" model is limited by the OS thread pool (hundreds), so a slow downstream call ties up a precious platform thread. With virtual threads, each request gets its own *virtual* thread that **cheaply parks while blocked on I/O**, freeing the carrier thread. You write ordinary blocking, imperative code (JPA, JdbcClient, `RestClient`) and get reactive-like scalability *without* the complexity of reactive programming.

Spring Boot 3.2+ makes this a **one-line switch**:

```yaml
spring:
  threads:
    virtual:
      enabled: true     # Tomcat serves each request on a virtual thread; @Async too
```

This is the recommended scalability path for most I/O-bound services on Java 21+ (compare with WebFlux in §18). **Caveat:** avoid `synchronized` blocks around blocking I/O (they can "pin" the carrier thread) — prefer `ReentrantLock`; this is improving in newer JDKs.

### Application events

For **in-process** decoupling, publish events with `ApplicationEventPublisher` and handle them with `@EventListener`. The order service publishes `OrderPlaced`; an unrelated listener sends a notification — neither knows about the other. Make listeners `@Async` for fire-and-forget, and use `@TransactionalEventListener(phase = AFTER_COMMIT)` to fire only after the DB transaction commits (so you never notify about an order that rolled back — a common bug).

```java
record OrderPlaced(Long orderId) {}

@Service
class OrderService2 {
    private final ApplicationEventPublisher events;
    OrderService2(ApplicationEventPublisher events) { this.events = events; }
    @Transactional
    void place(/* ... */) { /* save */ events.publishEvent(new OrderPlaced(42L)); }
}

@Component
class OrderListener {
    @TransactionalEventListener   // fires AFTER the transaction commits
    void onPlaced(OrderPlaced e) { /* send confirmation */ }
}
```

### Messaging (Kafka / RabbitMQ) overview

For **cross-service**, durable, asynchronous communication, use a message broker. **Kafka** (`spring-kafka`) is a distributed log — high-throughput event streaming, durable, replayable, great for event-driven architectures. **RabbitMQ** (`spring-amqp`) is a traditional message queue — flexible routing, good for task distribution and RPC-style work. Spring abstracts both: a `KafkaTemplate`/`RabbitTemplate` to send, and `@KafkaListener`/`@RabbitListener` to consume. This is how you scale beyond a single service and build resilient, decoupled systems. For an alternative synchronous inter-service style, see RPC concepts in [gRPC & RPC with Go](GO_GRPC_RPC_GUIDE.md).

```java
@Component
class OrderEventsConsumer {
    @KafkaListener(topics = "orders", groupId = "fulfillment")
    void consume(String message) { /* process the event */ }
}
```

---

## 14. Caching

### The cache abstraction

Repeatedly recomputing or re-fetching the same data wastes CPU and database load. Spring's **cache abstraction** lets you cache method results **declaratively**, independent of the cache provider. Enable it with `@EnableCaching`, then annotate methods:

| Annotation | Effect |
|---|---|
| `@Cacheable("name")` | Return the cached value if present; otherwise run the method and cache its result |
| `@CachePut` | Always run the method and update the cache (for writes that should refresh it) |
| `@CacheEvict` | Remove an entry (e.g. on update/delete) |

The **cache key** defaults to the method arguments; customize it with `key = "#id"` (SpEL). **The self-invocation gotcha applies** here too — `@Cacheable` is proxy-based.

```java
import org.springframework.cache.annotation.*;

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")  // cache by product id
    public ProductResponse findById(Long id) { /* expensive load */ return null; }

    @CacheEvict(value = "products", key = "#id") // invalidate on update
    public void update(Long id, UpdateProduct cmd) { /* ... */ }

    @CacheEvict(value = "products", allEntries = true) // nuke the cache
    public void reindexAll() { /* ... */ }
}
```

### Redis as the backing store

The default cache is a local, in-JVM `ConcurrentHashMap` — fine for a single instance but **not shared across instances** and lost on restart. For a scaled, multi-instance deployment you need a **distributed cache**: add `spring-boot-starter-data-redis` and `spring-boot-starter-cache`, and Spring routes the cache abstraction to **[Redis](REDIS_GUIDE.md)**. Now all instances share one cache, survive restarts, and you get TTLs.

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    redis:
      time-to-live: 600000     # 10-minute TTL on cached entries
```

### Strategies & pitfalls

- **Cache read-heavy, slow-changing data** (product catalogs, config, reference data). Don't cache rapidly-changing or per-request-unique data.
- **Always set a TTL** so stale data self-heals; rely on `@CacheEvict` for correctness on writes.
- **Beware cache stampede** (many requests miss simultaneously and all hit the DB) — Redis-level locking or request coalescing helps at high scale.
- **Cache DTOs, not entities** — caching detached JPA entities invites lazy-loading and serialization headaches.

See [Redis](REDIS_GUIDE.md) for data structures, eviction policies, and clustering.

---

## 15. Observability & Actuator

You cannot operate what you cannot see. **Observability** — metrics, health, traces, logs — is what turns a deployed jar into a *operable* production service.

### Spring Boot Actuator

**Actuator** (`spring-boot-starter-actuator`) adds production-readiness **endpoints** under `/actuator`: `/health` (is the app and its dependencies up?), `/info`, `/metrics`, `/prometheus`, `/env`, `/loggers` (change log levels at runtime), and more. **Security-critical:** by default expose **only** `health` (and `info`) over HTTP; the rest leak internals. Restrict and authenticate the sensitive ones (§19).

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus   # explicit allow-list ONLY
  endpoint:
    health:
      show-details: when-authorized          # don't leak dependency details publicly
      probes:
        enabled: true                         # /health/liveness & /health/readiness (k8s)
  metrics:
    tags:
      application: orders-api
```

`/health` aggregates **health indicators** (DB, Redis, disk, custom). **Liveness** (is the process alive?) and **readiness** (is it ready to serve traffic?) probes map directly to Kubernetes/orchestrator probes and to graceful rollout.

### Micrometer → Prometheus

**Micrometer** is Spring's metrics facade (the "SLF4J of metrics"): you record timers, counters, and gauges against it, and it exports to your monitoring backend. With `micrometer-registry-prometheus`, Actuator exposes `/actuator/prometheus` for **Prometheus** to scrape, which you then visualize in Grafana. Out of the box you get JVM, HTTP, datasource-pool, and cache metrics for free.

```java
import io.micrometer.core.instrument.*;

@Service
class CheckoutMetrics {
    private final Counter checkouts;
    CheckoutMetrics(MeterRegistry registry) {       // the registry is auto-configured
        this.checkouts = registry.counter("checkout.completed");
    }
    void recordCheckout() { checkouts.increment(); }  // custom business metric
}
```

### Structured logging & tracing

- **Structured (JSON) logging** — Boot 3.4+ has **built-in JSON log format** support (`logging.structured.format.console=ecs` or `logstash`), so logs are machine-parseable for ELK/Loki without extra libraries. Include a **correlation/trace id** in every line.
- **Distributed tracing** — **Micrometer Tracing** + **OpenTelemetry** propagate a trace id across service boundaries so you can follow one request through many services. Boot auto-instruments incoming HTTP, `RestClient`/`WebClient` calls, and more, and exports spans to a collector (Tempo, Jaeger, Zipkin).

### Graceful shutdown

On deploy/restart, you want in-flight requests to *finish* rather than be killed mid-response. Set `server.shutdown=graceful` (and a `spring.lifecycle.timeout-per-shutdown-phase`); on SIGTERM Boot stops accepting new requests, drains active ones, then exits. Pair with the readiness probe flipping to "out of service" so the load balancer stops routing first. This is essential for zero-downtime rolling deploys behind [Nginx](NGINX_GUIDE.md) or Kubernetes.

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # wait up to 30s for in-flight requests
```

---

## 16. Production Deployment

### The executable jar

`mvnw.cmd clean package` produces a **fat (uber) jar** in `target/` containing your code, all dependencies, and an embedded Tomcat. Run it with `java -jar target/api-0.0.1-SNAPSHOT.jar`. This single artifact *is* your deployable — no app server to provision. Pass config via env vars / `--spring.profiles.active=prod`. Set JVM flags for the container: `-XX:MaxRAMPercentage=75` (let the JVM size the heap to the container's memory) rather than guessing `-Xmx`.

### Docker — layered jars & buildpacks

You almost always ship Spring Boot in a **[Docker](DOCKER_GUIDE.md)** container. Two good approaches:

1. **Layered jar + multi-stage Dockerfile.** Boot's jar is organized into **layers** (dependencies, snapshot deps, resources, application classes) that change at different rates. Extracting them into separate Docker image layers means a code change only rebuilds the small app layer, not the huge dependency layer — fast rebuilds and small pushes.
2. **Buildpacks (no Dockerfile).** `mvnw.cmd spring-boot:build-image` uses Cloud Native Buildpacks to produce an optimized, secure OCI image with zero Dockerfile — it picks a JRE, layers correctly, and runs as non-root automatically. Great default.

```dockerfile
# Multi-stage: build the jar, then extract layers into a slim runtime image.
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw -q clean package -DskipTests
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /app/extracted

FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
# Copy layers least- to most-frequently-changed for optimal cache reuse.
COPY --from=build /app/extracted/dependencies/ ./
COPY --from=build /app/extracted/spring-boot-loader/ ./
COPY --from=build /app/extracted/snapshot-dependencies/ ./
COPY --from=build /app/extracted/application/ ./
RUN useradd -r app && chown -R app /app
USER app                                  # never run as root (§19)
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "org.springframework.boot.loader.launch.JarLauncher"]
```

### GraalVM native image (AOT)

For the fastest startup and smallest memory footprint (serverless, scale-to-zero, CLIs), compile a **GraalVM native image**: Spring Boot's **AOT (ahead-of-time) processing** plus the `native` profile produce a standalone native executable that starts in **tens of milliseconds** and uses a fraction of the RAM — at the cost of longer build times and some reflection/dynamic-feature constraints. Build with `mvnw.cmd -Pnative native:compile` (needs a GraalVM JDK) or `spring-boot:build-image` with `BP_NATIVE_IMAGE=true`. Adopt when startup/memory dominate; for a steady-state long-running service, the JIT's peak throughput often wins.

### Running behind Nginx

In production you don't expose the embedded Tomcat directly. Put **[Nginx](NGINX_GUIDE.md)** (or a cloud load balancer) in front as a **reverse proxy** to terminate **TLS/HTTPS**, load-balance across instances, serve static assets, and add rate limiting. Configure Boot to honor forwarded headers so it knows the original scheme/host (for correct redirects and `Location` headers):

```yaml
server:
  forward-headers-strategy: framework   # trust X-Forwarded-* from the proxy
```

### Config, secrets, profiles & migrations on deploy

- **Profiles:** activate `prod` via `SPRING_PROFILES_ACTIVE=prod`; keep prod config minimal and override secrets via env vars / secret store (§4, §19).
- **Health checks:** point the orchestrator at `/actuator/health/readiness` and `/actuator/health/liveness`.
- **DB migrations on deploy:** **Flyway runs pending migrations on application startup**, so a new version migrates the schema as it boots — make migrations **backward-compatible** (expand-then-contract) so old and new instances coexist during a rolling deploy. See [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md).

---

## 17. Project Structure & Maintainability at Scale

How you organize code determines how well the system ages. The themes: **high cohesion, low coupling, clear boundaries.**

### Package-by-feature vs package-by-layer

The common beginner layout is **package-by-layer**: `controller/`, `service/`, `repository/`, `model/`. It groups by technical role — and it scales *badly*, because one feature's code is scattered across four packages, and nothing stops any layer from reaching into any other feature.

The better default is **package-by-feature**: `order/`, `customer/`, `payment/`, each containing *that feature's* controller, service, repository, and DTOs. Benefits: a feature is self-contained and easy to find, change, or delete; you can enforce that features only talk through public interfaces; and it's the natural seam if you later split a feature into its own service.

```
com.example.api
├── order/        OrderController, OrderService, OrderRepository, Order, dtos…
├── customer/     CustomerController, CustomerService, …
├── payment/      PaymentService, gateways…
└── shared/       cross-cutting config, error handling, security
```

### Hexagonal / clean architecture

For complex domains, **hexagonal (ports & adapters)** architecture pushes the dependency direction inward: the **domain** (entities, business rules) has *no* framework dependencies; **ports** are interfaces the domain defines; **adapters** (a JPA repository, a REST controller, a Kafka consumer) implement those ports on the outside. Spring wires the adapters to the ports via DI. The payoff is a domain you can test and reason about without Spring, a database, or HTTP — and the ability to swap infrastructure. It's more ceremony than package-by-feature, so reserve it for genuinely complex, long-lived domains.

### Modular monolith — and when to split services

Don't reach for microservices prematurely; distribution adds enormous operational cost (network failures, distributed transactions, deployment choreography). A **modular monolith** — one deployable, but with strict internal module boundaries (enforce them with **Spring Modulith**, which verifies module dependencies and can event-source between modules) — gives most of the maintainability benefits without the distributed-systems tax. **Split a module into its own service only when** it has a genuinely different scaling profile, an independent deploy cadence, a separate team owning it, or a hard data-isolation requirement — not just because "microservices."

### API versioning

A public API's contract must stay stable while you evolve. Version it so clients don't break: **URI versioning** (`/api/v1/orders`) is the simplest and most visible; **header/media-type versioning** is cleaner but less discoverable. Add a new version additively, deprecate the old with a sunset window, and never break a published contract silently. **⚡ Version note:** Spring Framework 7 / Boot 4.0 adds **built-in API versioning** support, so you declare versions on mappings rather than hand-rolling the routing.

---

## 18. Performance & Scaling

### The big levers

1. **Fix N+1 queries** (§7) — usually the single biggest win. Profile the SQL your endpoints emit; use `JOIN FETCH`/`@EntityGraph`/projections. Turn on `spring.jpa.properties.hibernate.generate_statistics` in a load test to count queries per request.
2. **Tune HikariCP** (§8) — size the pool to what the DB can serve, not arbitrarily high. Watch pool-usage metrics (Micrometer exposes them) and connection-acquisition time.
3. **Cache** read-heavy, slow-changing data (§14) with Redis and TTLs.
4. **Paginate** every list endpoint (§7) — never `findAll()` an unbounded table.
5. **Batch** writes (`hibernate.jdbc.batch_size`) and use `JdbcClient` bulk operations for large imports.
6. **Index** the columns you filter and join on (this is a database concern — see [PostgreSQL](POSTGRESQL_GUIDE.md) and [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)).

### Virtual threads vs reactive (WebFlux) — choosing a concurrency model

There are two ways to handle massive concurrency on limited hardware:

- **Virtual threads (Java 21)** — keep writing simple, blocking, imperative code (Spring MVC, JPA, JdbcClient); the JVM cheaply parks virtual threads on I/O. **This is the right default for almost everyone** on Java 21+: reactive-like scalability with none of reactive's complexity. Enable with one property (§13).
- **Reactive (Spring WebFlux + Project Reactor)** — a fully non-blocking stack (`Mono`/`Flux`, R2DBC, reactive Redis) running on a small event-loop thread pool. It scales superbly under extreme concurrency and is great for streaming and very high fan-out I/O — but it's **viral and hard**: every layer must be non-blocking, debugging stack traces is painful, and one blocking call poisons the event loop. **Choose WebFlux only** when you have a proven need for its model (massive streaming, backpressure, an all-reactive ecosystem). For typical CRUD/REST services, virtual-thread MVC is simpler and just as scalable.

### Horizontal scaling & statelessness

To scale **out** (more instances behind [Nginx](NGINX_GUIDE.md)/a load balancer), your app must be **stateless**: no in-memory session or user state that ties a user to one instance. Push shared state to **[Redis](REDIS_GUIDE.md)** (sessions, cache, rate-limit counters) and the database. A stateless app scales linearly — add instances, balance across them, and any instance can serve any request. (This is why stateless JWT auth in §9 fits scaling better than server sessions.)

### Profiling

Measure before optimizing. Use **Actuator metrics** for HTTP latency percentiles, JVM, and pool stats; **async-profiler** / **JFR (Java Flight Recorder)** for CPU/allocation hotspots; and SQL statistics for query counts. Optimize the proven bottleneck, not your guess.

---

## 19. Security Hardening

Security is a continuous discipline, not a checklist you do once. The essentials for a production Spring service:

- **Secure by default, then open up.** Keep `anyRequest().authenticated()` as the catch-all (§9); explicitly `permitAll()` only what's truly public.
- **Hash passwords** with BCrypt/Argon2; **never** store or log plaintext credentials, tokens, or PII. (See [Java](JAVA_GUIDE.md) for crypto basics.)
- **Validate and sanitize all input** (§6). Use **parameterized queries** — Spring Data/JdbcClient parameter binding prevents SQL injection; **never** concatenate user input into SQL/JPQL.
- **Manage secrets outside the code** (§4): env vars / Vault / Kubernetes Secrets. Audit that no secret is in Git history; rotate anything ever committed.
- **HTTPS everywhere.** Terminate TLS at [Nginx](NGINX_GUIDE.md)/the load balancer, redirect HTTP→HTTPS, and set HSTS. Boot can enforce `requiresChannel().anyRequest().requiresSecure()` behind a proxy that forwards `X-Forwarded-Proto`.
- **Lock down Actuator** (§15): expose only `health`/`info` publicly; require auth (and ideally a separate management port/network) for `env`, `loggers`, `heapdump`, `threaddump`, `prometheus`. An exposed `/actuator/env` can leak secrets.
- **Dependency scanning.** Use the **OWASP Dependency-Check** Maven plugin or Snyk/`mvn versions` to catch known-vulnerable dependencies (CVEs); keep Spring Boot patched (security fixes ship in patch releases — staying current *is* a security control).
- **OWASP basics in Spring:** Spring Security defaults guard against many of the OWASP Top 10 — keep CSRF on for session apps, set safe security headers (CSP, X-Content-Type-Options, frame options — configurable via the `headers` DSL), enforce authorization on *every* endpoint, rate-limit at the proxy, and avoid leaking errors (§6).
- **Method-level authorization** (§9) for defense in depth — don't rely solely on URL rules.
- **Principle of least privilege** for the DB account the app uses (no superuser; only the rights it needs) — see [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md).

```java
// Security headers + HTTPS enforcement (behind a TLS-terminating proxy).
@Bean
SecurityFilterChain hardening(HttpSecurity http) throws Exception {
    http.headers(h -> h
            .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
            .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true)))
        .requiresChannel(c -> c.anyRequest().requiresSecure()); // force HTTPS
    // ... rest of the chain ...
    return http.build();
}
```

---

## 20. Gotchas & Best Practices

The mistakes that bite *everyone* at least once. Internalize these.

- **Field injection over constructor injection.** `@Autowired` on fields hides dependencies, can't be `final`, and is hard to test. **Always use constructor injection** (§2). A bloated constructor is a *feature* — it reveals an over-large class.
- **`javax` → `jakarta`.** On Spring Boot 3, all imports are `jakarta.*` (persistence, validation, servlet). A `javax.persistence.Entity` import means an out-of-date example that won't compile (§1). This single confusion accounts for countless failed builds.
- **The N+1 problem & `LazyInitializationException`.** Lazy associations accessed outside a transaction/session throw `LazyInitializationException`; accessed in a loop they cause N+1 (§7). Fix with `JOIN FETCH`/`@EntityGraph`/projections — *not* by making things `EAGER` (which makes N+1 worse and global).
- **`@Transactional` self-invocation.** Calling a `@Transactional` (or `@Async`/`@Cacheable`/`@PreAuthorize`) method from another method **in the same bean** bypasses the proxy and silently does nothing (§10, §11). Move the method to another bean.
- **Entity-as-DTO leak.** Returning JPA entities from controllers leaks internal fields, couples your API to your schema, and triggers lazy-loading during serialization. **Always map to DTOs/records** (§5).
- **Open Session In View (OSIV).** Boot enables OSIV by default, keeping the Hibernate session open through view rendering. It *hides* N+1 and `LazyInitializationException` in development, then surprises you in production by holding DB connections longer (hurting pool throughput). **Set `spring.jpa.open-in-view=false`** and load what you need explicitly in the service layer (§4, §7).
- **`ddl-auto: update` in production.** Letting Hibernate mutate the schema is unpredictable and can lose data. Use `validate` + **Flyway/Liquibase** migrations (§7).
- **Profile/config mistakes.** Forgetting to activate `prod`, leaking a dev `create-drop` into prod, or hard-coding a secret in `application.yml`. Keep secrets in env vars, validate config with `@ConfigurationProperties` + `@Validated` (fail fast at startup), and double-check the active profile (§4).
- **Unbounded thread pools / `@Async` default executor.** Configure a bounded `TaskExecutor`; an unbounded one can exhaust memory under load (§13).
- **Catching and swallowing exceptions silently.** Always log the cause; return a generic message to clients but keep the real details in your logs (§6, §19).
- **Over-fetching with `findAll()`.** Never load an unbounded table; paginate (§7).
- **Cranking HikariCP `maximum-pool-size`.** Bigger is not faster past the DB's capacity — it adds latency. Tune with metrics (§8).
- **Not using slice tests.** Loading the full context for every test (`@SpringBootTest` everywhere) makes the suite slow. Use `@WebMvcTest`/`@DataJpaTest` slices and pure unit tests for the bulk (§12).

---

## 21. Study Path & Build-to-Learn Projects

Spring Boot is broad; learn it by **building**, layering one concept at a time onto a single growing project. The path below mirrors this guide's order — each step adds one production capability.

### Suggested learning order

1. **Foundations (§1–§3).** Understand IoC/DI deeply — it's everything. Generate a project from Spring Initializr, get `@SpringBootApplication` running, and wire a couple of beans by constructor injection until the container's behavior is obvious.
2. **Config & REST (§4–§5).** Add `application.yml`, profiles, and `@ConfigurationProperties`. Build CRUD endpoints with `@RestController`, records as DTOs, and `ResponseEntity`. Drill the **DTO-vs-entity** boundary until it's reflex.
3. **Validation & errors (§6).** Add Bean Validation and a global `@RestControllerAdvice` emitting RFC 9457 problem details.
4. **Persistence (§7–§8).** Add Spring Data JPA over [PostgreSQL](POSTGRESQL_GUIDE.md), Flyway migrations, pagination, and `@Transactional`. Deliberately *create* an N+1 problem, observe it in the SQL logs, then fix it — that lesson sticks.
5. **Security (§9).** Add Spring Security with JWT/OAuth2 resource server, password hashing, method security, and CORS.
6. **Service layer, AOP, testing (§10–§12).** Refactor to clean layers with MapStruct; add a timing aspect; build a real test suite — unit + slice + a Testcontainers integration test against real Postgres.
7. **Async, caching, observability (§13–§15).** Add `@Async`/events, Redis caching, virtual threads, Actuator + Micrometer/Prometheus, and graceful shutdown.
8. **Production (§16–§19).** Containerize with [Docker](DOCKER_GUIDE.md), run behind [Nginx](NGINX_GUIDE.md), and apply the structure/performance/security hardening lessons.

### Build-to-learn projects (one project, grown in stages)

- **Stage 1 — REST API with JPA + Postgres.** An "Orders" service: customers, orders, line items. CRUD endpoints, DTOs/records, Bean Validation, global error handling, Flyway migrations, pagination, and a clean controller→service→repository split. *Goal: master the core request-to-database flow and the DTO boundary.*
- **Stage 2 — add Spring Security + JWT.** Registration/login issuing JWTs, an OAuth2 resource server validating them, role/authority-based access, `@PreAuthorize` ownership checks, and CORS. *Goal: a fully authenticated, authorized API.*
- **Stage 3 — add caching + Actuator + async.** Cache the read-heavy product/catalog endpoints in [Redis](REDIS_GUIDE.md) with TTLs; publish domain events and send notifications via `@Async` after commit; expose health/metrics through Actuator + Prometheus; enable virtual threads; add graceful shutdown. *Goal: an observable, performant, scalable service.*
- **Stage 4 — containerize & deploy.** Layered-jar [Docker](DOCKER_GUIDE.md) image (or buildpacks), run multiple stateless instances behind [Nginx](NGINX_GUIDE.md) with TLS, externalize all config/secrets, run Flyway on deploy, add readiness/liveness probes, and harden per §19. Optionally build a **GraalVM native image** and compare startup/memory. *Goal: a production-grade, horizontally-scalable deployment.*
- **Stretch — split a module into a second service** communicating via Kafka/RabbitMQ events (§13), or explore a **modular monolith** with Spring Modulith before distributing. *Goal: understand the real cost and seams of going distributed.*

Build each stage, break it deliberately, read the logs, and fix it — that loop, more than any tutorial, is how Spring Boot stops being magic and becomes a tool you command.
