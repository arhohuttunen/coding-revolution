---
title: Testing Spring Security
date: 2022-11-20
author: Arho Huttunen
summary: Spring Security has good support for MockMvc and WebTestClient. Learn to test authentication and authorization of Spring Boot applications.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
  - Spring Security
  - Integration testing
image:
  focal_point: center
  preview_only: true
---

Security plays a major role in software. Eventually, everyone needs to add security to their project. In this article, we look at how to test authentication and authorization of Spring Boot applications. We will cover both MVC servlet applications and reactive WebFlux applications.

{{< toc >}}

Also, if you are interested in a complete course on Spring Boot testing, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. You can support me by buying through that link because I get a share.

## Testing Spring MVC Applications With Security

Spring Security integrates well with the Spring Web MVC framework. It also has a comprehensive integration with Spring MVC Test.

### Example Spring MVC Application With Security

Let's start with a simple application that manages customers. We want to create, get, and delete customers.

```java
@RestController
@RequiredArgsConstructor
public class CustomerController {
    private final CustomerRepository customerRepository;

    @GetMapping("/customer/{id}")
    Customer getCustomer(@PathVariable Long id) {
        return customerRepository.findById(id).orElseThrow();
    }

    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    Customer createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    void deleteCustomer(@PathVariable Long id) {
        customerRepository.deleteById(id);
    }
}
```

We probably don't want unauthorized people to create and delete customers, though. Thus, we are going to use a simple security configuration to add authentication.

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfiguration {
}
```

Now, to secure endpoints, we can use the `@PreAuthorized` annotation to enable method security. We are going to do that for `POST` and `DELETE` operations.

```java
    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('ADMIN')")
    Customer createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    void deleteCustomer(@PathVariable Long id) {
        customerRepository.deleteById(id);
    }
```

From the testing perspective, it doesn't matter much how the security configuration has been setup. For the sake of simplicity, we will keep the example configuration short.

### Spring Security With @WebMvcTest

#### Configure the Test
To test our controllers in isolation, we can use the Spring Boot Test `@WebMvcTest` test slice. Check out the [Testing Web Controllers With @WebMvcTest](/spring-boot-webmvctest/) for more information.

To test Spring Security, let's start with the endpoints that don't require admin rights.

```java
@WebMvcTest(CustomerController.class)
class CustomerControllerTests {
    @MockBean
    private CustomerRepository customerRepository;

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getCustomer() throws Exception {
        when(customerRepository.findById(1L))
                .thenReturn(Optional.of(new Customer(1L, "John", "Doe")));

        mockMvc.perform(get("/customer/{id}", 1L))
                .andExpect(status().isOk());
    }
}
```

If we run the application and try to access the endpoint unauthenticated, we can observe that the application denies access. However, the test passes. What is the problem in the test then?

Using `@WebMvcTest` loads beans needed for the controller, but it doesn't know which other configuration beans to load. We have to tell Spring to load the security configuration by using the `@Import(SecurityConfiguration.class)` annotation.

```java
@WebMvcTest(CustomerController.class)
@Import(SecurityConfiguration.class)
```

Now running the test results into 401 Unauthorized, so the request lacks authentication. This is the same HTTP status code that the running application gives us.

```text
Status expected:<200> but was:<401>
Expected :200
Actual   :401
```

#### Mock Authentication

To run the test as a user, we can use the `@WithMockUser` annotation to provide fake authentication for the user.

```java
    @Test
    @WithMockUser
    void getCustomer() throws Exception {
        // ...
    }
```

This will make the user authenticated, and the test will pass.

To make sure that our security configuration is working, we can also add a test to verify the response is 401 Unauthorized if we haven't authenticated. We can use the `@WithAnonymousUser` annotation, which is optional but emphasizes the fact it's an unauthenticated user.

```java
    @Test
    @WithAnonymousUser
    void cannotGetCustomerIfNotAuthorized() throws Exception {
        mockMvc.perform(get("/customer/{id}", 1L))
                .andExpect(status().isUnauthorized());
    }
```

If we use `@WithMockUser` with an endpoint that requires more permissions, we will get 403 Forbidden. This means the authenticated user was not authorized to access the resource. We can add the required roles to the `@WithMockUser` annotation.

```java
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanCreateCustomers() throws Exception {
        when(customerRepository.save(any())).thenReturn(new Customer(1L, "John", "Doe"));

        mockMvc.perform(post("/customer")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                )
                .andExpect(status().isCreated());
    }
```

Running the test, we are still getting 403 Forbidden. It's not obvious what is wrong. What is happening?

#### Enable CSRF Token

To debug Spring security issues, we can enable security debug logging to see what happens.

```text
logging:
  level:
    org:
      springframework:
        security: DEBUG
```

After enabling security logging, we can see the reason. Spring Security enables CSRF protection by default and the request is missing the CSRF token.

```text
o.s.security.web.csrf.CsrfFilter : Invalid CSRF token found for http://localhost/customer
```

One could disable CSRF for tests, but it's not a great option. We can provide a CSRF token with a request post processor instead.

```java
        mockMvc.perform(post("/customer")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                        .with(csrf())
                )
                .andExpect(status().isCreated());
```

Now we can see that the test passes.

#### Use Request Post Processors

The `@WithMockUser` annotation is handy, but we don't like annotations, we can use other request post processors instead.

```java
    @Test
    void adminCanDeleteCustomer() throws Exception {
        mockMvc.perform(delete("/customer/{id}", 1L)
                        .with(csrf())
                        .with(user("admin").roles("ADMIN"))
                )
                .andExpect(status().isNoContent());
    }
```

To verify unauthorized status, we can add an `anonymous()` post processor. Using it is optional, but it highlights again the fact that it's an unauthenticated user.

```java
    @Test
    void cannotDeleteCustomerIfNotAuthorized() throws Exception {
        mockMvc.perform(delete("/customer/{id}", 1L)
                        .with(csrf())
                        .with(anonymous())
                )
                .andExpect(status().isUnauthorized());
    }
```

It’s not complicated to cover the different cases. The level of verification we want depends on how complex our security configuration is. Since this is a pretty vital part of the application, it is good to test it throughly.

### Spring Security and MockMvc in @SpringBootTest

If we want to test a larger slice of the application with `@SpringBootTest`, we have to set up the `MockMvc` for the tests. We could use `@AutoconfigureMockMvc` annotation here instead, but it's good to know how to initialize `MockMvc` manually.

```java
@SpringBootTest
class CustomerMockEnvTests {
    @Autowired
    private WebApplicationContext context;

    private MockMvc mockMvc;

    @BeforeEach
    public void setup() {
        mockMvc = MockMvcBuilders
                .webAppContextSetup(context)
                .apply(springSecurity())
                .build();
    }
}
```

The important part here is to add `MockMvcConfigurer.springSecurity()` to the configuration. The problem with the `@AutoconfigureMockMvc` annotation is that it could mess up with the Spring Boot application context test caching.

Once `MockMvc` has been setup, there is nothing different in using it compared to testing with `@WebMvcTest`.

### Spring Security and MockMvc WebTestClient in @SpringBootTest

What about if we'd like to write end-to-end tests for the application? Since the test will start the application in another process, one option is to use the `WebTestClient` or to make requests to the application. There is one catch though: we cannot auto-wire the `WebTestClient` bean directly in the test.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class CustomerServerEnvTests {
    @Autowired
    private WebTestClient webClient;

    @Test
    void createCustomer() {
        webClient.mutateWith(csrf()).post().uri("/customer")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
}

```

The test will fail with an error as soon as we try to call `mutateWith(csrf())` to add the CSRF token.

```text
Cannot invoke "org.springframework.web.server.adapter.WebHttpHandlerBuilder.filters(java.util.function.Consumer)" because "httpHandlerBuilder" is null
```

This is because they originally designed `WebTestClient`  for testing reactive Spring applications and [do not support this yet](https://github.com/spring-projects/spring-security/issues/10841#issuecomment-1048099312).

There is a workaround for this if we create the `WebTestClient` manually using `MockMvcWebTestClient`. The setup is pretty similar to the `MockMvc` setup.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class CustomerServerEnvTests {
    @Autowired
    private WebApplicationContext context;

    private WebTestClient webClient;

    @BeforeEach
    void setup() {
        webClient = MockMvcWebTestClient.bindToApplicationContext(context)
                .apply(springSecurity())
                .defaultRequest(get("/").with(csrf()))
                .configureClient()
                .build();
    }
```

By adding a `defaultRequest()` we can add the CSRF token to all requests.

With this setup, we can use the `@WithMockUser` annotation again.

```java
    @Test
    @WithMockUser(roles = "ADMIN")
    void createCustomer() {
        webClient.post().uri("/customer")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

Since it's an end to end test, instead of mocking the authentication, we might want to provide the authentication headers instead.

```java
    @Test
    void createCustomer() {
        webClient.post().uri("/customer")
                .headers(http -> http.setBasicAuth("username", "password"))
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

## Testing WebFlux Spring Applications With Security

Spring Security integrates well with the Spring WebFlux framework. It also has a comprehensive integration with Spring `WebTestClient`.

### Example Spring WebFlux Application With Security

Let's start with the previous example application and covert that to a reactive application.

```java
@RestController
@RequiredArgsConstructor
public class CustomerController {
    private final CustomerRepository customerRepository;

    @GetMapping("/customer/{id}")
    Mono<Customer> getCustomer(@PathVariable Long id) {
        return customerRepository.findById(id);
    }

    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('ADMIN')")
    Mono<Customer> createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    Mono<Void> deleteCustomer(@PathVariable Long id) {
        return customerRepository.deleteById(id);
    }
}
```

We are going to need to configure security. The annotations differ somewhat from the MVC security configuration.

```java
@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
public class SecurityConfiguration {
}
```

Once the security configuration is in place, the endpoints are now secured.

### Spring Security With @WebFluxTest

To test our reactive controllers in isolation, we can use the Spring Boot Test `@WebFluxText` test slice. We can also use `WebTestClient` directly in these tests since it has been designed to work with reactive applications.

```java
@WebFluxTest(CustomerController.class)
@Import(SecurityConfiguration.class)
class CustomerControllerTests {
    @MockBean
    private CustomerRepository customerRepository;

    @Autowired
    private WebTestClient webClient;

    @Test
    @WithMockUser
    void getCustomer() {
        when(customerRepository.findById(1L))
                .thenReturn(Mono.just(new Customer(1L, "John", "Doe")));

        webClient.get().uri("/customer/{id}", 1)
                .exchange()
                .expectStatus().isOk();
    }
```

For any endpoints who require the CSRF token, we need to add it.

```java
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanCreateCustomers() {
        when(customerRepository.save(any()))
                .thenReturn(Mono.just(new Customer(1L, "John", "Doe")));

        webClient.mutateWith(csrf())
                .post().uri("/customer")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

Again, if we don't want to use the `@WithMockUser` annotation, we can mutate the client with methods coming from `SecurityMockServerConfigurers`.

```java
    @Test
    void adminCanDeleteCustomer() {
        when(customerRepository.deleteById(1L)).thenReturn(Mono.empty());

        webClient.mutateWith(csrf())
                .mutateWith(mockUser().roles("ADMIN"))
                .delete().uri("/customer/{id}", 1)
                .exchange()
                .expectStatus().isNoContent();
    }
```

### Spring WebFlux Security With @SpringBooTest

Moving on to the `@SpringBootTest`, the `WebTestClient` bean is not available by default. We could add the `@AutoconfigureWebTestClient` annotation, but it's good to know how to set up the client manually.

```java
@SpringBootTest
public class CustomerControllerEndToEndTests {
    @Autowired
    private ApplicationContext context;

    private WebTestClient webClient;

    @BeforeEach
    void setup() {
        webClient = WebTestClient.bindToApplicationContext(context)
                .apply(springSecurity())
                .configureClient()
                .build();
    }
}
```

The problem with the `@AutoconfigureWebTestClient` annotation is that it could mess up with the Spring Boot application context test caching.

Here, it doesn't matter if we are running the test in a mock environment or a server environment. `WebTestClient` is configured the same way in both in a reactive application.

Once `WebTestClient` has been setup there is nothing different in using it compared to testing with `@WebFluxTest`. We can either use the `@WithMockUser` or mutate the client with mock security from `SecurityMockServerConfigurers`.

## Summary

Spring Security integrates well with the Spring Web MVC and Spring WebFlux frameworks. It also has comprehensive integration with `MockMvc` and `WebTestClient`.

We can fake the authentication using an annotation or a method-based approach. It’s also possible to provide different roles for testing authorization.

You can find the example code for this article on GitHub [for the MVC application](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-security-testing) and [for the WebFlux application](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-webflux-security-testing).
