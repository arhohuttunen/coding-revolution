---
title: Introduction to Testing Spring Boot Applications
date: 2022-09-20
author: Arho Huttunen
summary: Find out where to start learning to test your Spring Boot applications.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
---

When learning to test Spring Boot applications it can be a difficult task to know where to begin. The easiest way to get started is the Spring Boot Starter Test which allows us to start writing tests right away. The starter includes a number of utilities and libraries to help with testing applications. 

In this article, we will look at different steps that you could take on your journey to learn Spring Boot testing. It's not a comprehensive guide but rather an overview of what's available and a starting point to look for more information.

This article is part of the [Spring Boot Testing Tutorial](/spring-boot-testing-tutorial).

## Introduction to JUnit

JUnit is by far the most popular testing framework for Java. If you are not familiar with JUnit, this is where you should start.

We won't be looking into JUnit in detail. There is already a thorough [JUnit 5 Tutorial](/junit-5-tutorial/) that you should take a look at.

Here's what a plain JUnit 5 test looks like:

```java
class CalculatorTest {
    @Test  
    void addNumbers() {  
        Calculator calculator = new Calculator();  
        assertEquals(3, calculator.add(1, 2));  
    }
}
```

At the bare minimum, you should know how to write [JUnit 5 Assertions](/junit-5-assertions/) and understand the [JUnit 5 Test Lifecycle](/junit-5-test-lifecycle/). Learning JUnit 5 forms the foundation to build your testing habits upon.

## Introduction to Mockito

Mockito is the most popular mocking framework for unit tests in Java. Simply put, mock objects simulate the behavior of real objects. When we want to test a part of an application in isolation we can replace some parts with mock objects.

{{< figure src="mock-objects.svg" caption="Replacing real objects with mock objects" theme="light" >}}

There are basically two ways to use Mockito: stubbing responses and verification of interactions.

A stub is something that replaces a real object and returns predefined data to the test. For example, we might not want to make a database call, so we can replace our database code with a stub.

Verification of interactions happen by checking that a certain method was called. Verification happens after execution.

Something worth mentioning is that we don't need Mockito to write mocks. It's possible to write our own as well. Mockito just happens to be so popular that you are almost guaranteed to stumble upon it. To learn more about it take a look at the [Mockito documentation](https://site.mockito.org).

## Unit Testing Spring Boot Applications

Unit tests are the basic building block of any testing strategy. Unit tests are fast and we run them often.

The beauty of dependency injection is that it makes our code easier to test. However, there's nothing special about Spring Boot applications that would require us to use Spring for dependency injection. In fact, you can construct any objects without Spring using the `new` operator.

What this means from the testing perspective is that we don't need Spring Boot to instantiate objects for our tests. We can do that manually in our tests and be independent from the framework.

We can write unit tests for parts of our Spring Boot applications without the Spring Boot testing support. This is why we should learn the basics of JUnit 5 and Mockito before looking into the Spring Boot testing support.

Here is a simple unit test that does not need Spring Boot to run:

```java
@ExtendWith(MockitoExtension.class)  
class OrderServiceTests {  
    @Mock  
    private OrderRepository orderRepository;  
    @Mock  
    private PaymentRepository paymentRepository;  
    @InjectMocks  
    private OrderService orderService;  
  
    @Test  
    void payOrder() {  
        Order order = new Order(1L, false);  
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));  
        when(paymentRepository.save(any())).then(returnsFirstArg());  
  
        Payment payment = orderService.pay(1L, "4532756279624064");  
  
        assertThat(payment.getOrder().isPaid()).isTrue();  
        assertThat(payment.getCreditCardNumber()).isEqualTo("4532756279624064");  
    }
}
```

To learn how to make your code unit-testable take a look at the [Spring Boot Unit Testing](/spring-boot-unit-testing/) article.

Of course, unit testing alone is not enough. When testing the web layer or the persistence layer we want to make sure that the endpoints respond correctly and data is stored correctly in the database.

## Testing a Slice of Spring Boot Application

When we move on to integration testing we need to load a Spring `ApplicationContext` in our tests. That's when the Spring auto-configurations come handy.

The auto-configuration annotations provided by Spring Boot only load parts of the configuration. These test slices load only Spring beans required for testing a slice of the application. Starting only a part of the application improves the test execution time. It also allows us to write simpler test setup.

For example, when testing the MVC controllers, we don't necessarily want to involve database calls in those tests. Similarly, when testing testing the JPA queries, we don't care about the web layer. Spring Boot provides the annotations `@WebMvcTest` and `@DataJpaTest` that allow us to do exactly that.
 
To get started with the test slices, take a look at these two most commonly used annnotations:

- [Testing Web Controllers With @WebMvcTest](/spring-boot-webmvctest/)
- [Testing the Persistence Layer With @DataJpaTest](/spring-boot-datajpatest/)

There are also other annotations that target specific slices like `@JsonTest`, `@WebFluxText`, `@DataJdbcTest`, `@DataMongoTest` and `@RestClientTest`.

Some people call these tests unit tests because they test a part of the application in isolation. However, the tests rely on the Spring application context which is why some people call them integration tests. To make it more explicit we can call tests that use Spring Boot test slices such as `@WebMvcTest` or `@DataJpaTest` **narrow integration tests**.

## Integration Testing Spring Boot Applications

Spring Boot provides us the `@SpringBootTest` annotation which creates the application context through `SpringApplication`. These tests can be either started with a mock web environment or by starting a real web environment. The difference between these two is that with the real web environment we have to do real HTTP requests to our application.

Even though we are running the test with the entire application we don't have to connect to external services. We can replace an external database by an instance that is running inside a Docker container by using `Testcontainers`. We can also mock external web services by using `MockWebServer`.

To learn more about `MockWebServer` take a look at the [Testing Spring Boot WebClient With MockWebServer](/spring-boot-webclient-mockwebserver/) article.

Some people think that your integration tests have to always test the entire application. As we already saw it's possible to write integration tests for smaller slices too. We can call tests that load the entire application context with `@SpringBooTest` **broad integration tests**.

To learn more about `@SpringBootTest` take a look at the [Spring Boot Integration Testing](/spring-boot-integration-testing/) article.

## Next Steps

This is only the start but will give us a solid foundation to get started on testing our Spring Boot applications. Once we know the basics of writing unit and integration tests there is more to explore.

If you want to avoid a steep learning curve without spending too much time searching for information, check out the [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. If you buy through the link I will get a share, but I only recommend it because I think it's a great package.