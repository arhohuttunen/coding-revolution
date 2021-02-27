---
title: How to Create a Test Data Builder
date: 2021-02-27
author: Arho Huttunen
summary: The purpose of the test data builder pattern is to make it easy to create objects for tests. Learn how to write descriptive test data builders.
categories:
  - Testing
tags:
  - clean code
---

In this article, we will learn how to write test data builders that remove duplication in constructing objects and increase the expressiveness of the test code.

In a [previous article](/dry-damp-tests), we talked about how to remove duplication while at the same time making the code more descriptive. This article is a more practical guide concentrating on the test data builder pattern.

## Constructing Test Data

Constructing objects in tests can become very burdensome. In our production code, we create objects in relatively few places. However, in tests, we might be testing many scenarios and have to provide all the constructor arguments every time creating an object.

Let's take a look at an example.

```java
public class Order {
    private final Long orderId;
    private final Customer customer;
    private final List<OrderItem> orderItems;
    private final Double discountRate;
    private final String couponCode;
    
    // ...
}

public class Customer {
    private final Long customerId;
    private final String name;
    private final Address address;
    
    // ...
}
```

```java
@Test
void constructOrder() {
    Address address = new Address("1216  Clinton Street", "Philadelphia", "19108", null);
    Customer customer = new Customer(1L, "Terry Tew", address);
    Order order = new Order(1L, customer, 0.0, null);
    order.addOrderItem(new OrderItem("Coffee mug", 1));
    order.addOrderItem(new OrderItem("Tea cup", 1));
    
    // ...
}
```

In this example, we use constructors and immutable objects, so the objects do not have setter methods. The code becomes quite hard to read, and it's filled with details that are irrelevant to the behavior that we test. Also, tests become brittle because adding any new parameters will break a lot of tests.

## Test Data Builders

For classes that require complex setup we can create a test data builder. The builder has a field for each constructor parameter and initializes them to safe values.

```java
public class OrderBuilder {
    private Long orderId = 1L;
    private Customer customer = new CustomerBuilder().build();
    private List<OrderItem> orderItems = new ArrayList<>();
    private Double discountRate = 0.0;
    private String couponCode;

    public OrderBuilder withId(Long orderId) {
        this.orderId = orderId;
        return this;
    }

    public OrderBuilder withCustomer(Customer customer) {
        this.customer = customer;
        return this;
    }

    public OrderBuilder withOrderItem(OrderItem orderItem) {
        this.orderItems.add(orderItem);
        return this;
    }

    public OrderBuilder withDiscountRate(Double discountRate) {
        this.discountRate = discountRate;
        return this;
    }

    public OrderBuilder withCouponCode(String couponCode) {
        this.couponCode = couponCode;
        return this;
    }

    public Order build() {
        Order order = new Order(orderId, customer, discountRate, couponCode);
        orderItems.forEach(order::addOrderItem);
        return order;
    }
}
```

We provide the actual values using public "with" methods which can be chained. We can omit any fields that are not relevant to our test.

```java
class BuilderTest {
    @Test
    void buildOrder() {
        Order order = new OrderBuilder()
                .withCustomer(new CustomerBuilder()
                        .withName("Terry Tew")
                        .withAddress(new AddressBuilder()
                                .withStreet("1216  Clinton Street")
                                .withCity("Philadelphia")
                                .withPostalCode("19108")
                                .build()
                        )
                        .build()
                )
                .withOrderItem(new OrderItemBuilder()
                        .withName("Coffee mug")
                        .withQuantity(1)
                        .build()
                )
                .withOrderItem(new OrderItemBuilder()
                        .withName("Tea cup")
                        .withQuantity(1)
                        .build()
                )
                .build();
    }
}
```

Test data builders offer several benefits over constructing objects by hand:

1. They hide most of the syntax noise related to creating objects.
2. They make the simple cases simple, and special cases are not much more complicated.
3. They are resilient to changes that happen in the structure of objects.
4. They make the test code easier to read and spot errors.

Consider an example where you accidentally switch the argument order.

```java
Address address = new Address("1216  Clinton Street", "19108", "Philadelphia", null);
```

We have switched places for the postal code and the city. The error is not easy to spot. Using a test data builder makes it more obvious.

## Passing Builders As Arguments

In our builder example, the builder consumes some arguments that are objects built by other builders. If we pass those builders as arguments instead of the constructed objects, we can simplify the code by removing build methods.

```java
public class OrderBuilder {
    // ...

    public OrderBuilder withCustomer(CustomerBuilder customerBuilder) {
        this.customer = customerBuilder.build();
        return this;
    }
    
    public OrderBuilder withOrderItem(OrderItemBuilder orderItemBuilder) {
        this.orderItems.add(orderItemBuilder.build());
        return this;
    }

    // ...
}
```

We are still duplicating the name of the constructed type in both the "with" methods and the builder names. We can take advantage of the type system and shorten the names of the "with" methods.

```java
public class OrderBuilder {
    // ...

    public OrderBuilder with(CustomerBuilder customerBuilder) {
        this.customer = customerBuilder.build();
        return this;
    }
    
    public OrderBuilder with(OrderItemBuilder orderItemBuilder) {
        this.orderItems.add(orderItemBuilder.build());
        return this;
    }

    // ...
}
```

Now using the builder becomes less verbose. 

```java
class BuilderTest {
    @Test
    void buildOrder() {
        Order order = anOrder()
                .with(new CustomerBuilder()
                        .withName("Terry Tew")
                        .with(new AddressBuilder()
                                .withStreet("1216  Clinton Street")
                                .withCity("Philadelphia")
                                .withPostalCode("19108")
                        )
                )
                .with(new OrderItemBuilder().withName("Coffee mug").withQuantity(1))
                .with(new OrderItemBuilder().withName("Tea cup").withQuantity(1))
                .build();
    }
}
```

## Builder Factory Methods

There is still some noise in the tests because we have to construct various builders. We can reduce this noise by adding factory methods for the builders.

```java
public class OrderBuilder {
    // ...

    private OrderBuilder() {}

    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    // ...
}

class BuilderTest {
    @Test
    void buildOrder() {
        Order order = anOrder()
                .with(aCustomer()
                        .withName("Terry Tew")
                        .with(anAddress()
                                .withStreet("1216  Clinton Street")
                                .withCity("Philadelphia")
                                .withPostalCode("19108")
                        )
                )
                .with(anOrderItem().withName("Coffee mug").withQuantity(1))
                .with(anOrderItem().withName("Tea cup").withQuantity(1))
                .build();
    }
}
```

The factory methods hide a lot of details about the builders. The code is now more descriptive as it speaks in domain terms.

## Creating Similar Objects

If we need to create similar objects, we could use different builders for them. However, different builders lead to duplication and make it harder to spot differences.

Let's take a look at the following example.

```java
    @Test
    void buildSimilarOrders() {
        Order orderWithSmallDiscount = anOrder()
                .with(anOrderItem().withName("Coffee mug").withQuantity(1))
                .with(anOrderItem().withName("Tea cup").withQuantity(1))
                .withDiscountRate(0.1)
                .build();
        Order orderWithBigDiscount = anOrder()
                .with(anOrderItem().withName("Coffee mug").withQuantity(1))
                .with(anOrderItem().withName("Tea cup").withQuantity(1))
                .withDiscountRate(0.5)
                .build();
    }
```

Because of the repetition in the construction, the difference with the discount rate gets hidden in the noise. We can extract a builder with a joint state and then provide the differing values for each object separately. 

```java
    @Test
    void buildSimilarOrders() {
        OrderBuilder coffeeMugAndTeaCup = anOrder()
                .with(anOrderItem().withName("Coffee mug").withQuantity(1))
                .with(anOrderItem().withName("Tea cup").withQuantity(1));

        Order orderWithSmallDiscount = coffeeMugAndTeaCup.withDiscountRate(0.1).build();
        Order orderWithBigDiscount = coffeeMugAndTeaCup.withDiscountRate(0.5).build();
    }
```

By reusing the common parts and naming the variables descriptively, the differences become much more apparent.

There is one pitfall in this approach, though. Let's take a look at another example.

```java
    @Test
    void buildSimilarOrders() {
        OrderBuilder coffeeMugAndTeaCup = anOrder()
                .with(anOrderItem().withName("Coffee mug").withQuantity(1))
                .with(anOrderItem().withName("Tea cup").withQuantity(1));

        Order orderWithDiscount = coffeeMugAndTeaCup.withDiscountRate(0.1).build();
        Order orderWithCouponCode = coffeeMugAndTeaCup.withCouponCode("HALFOFF").build();
    }
```

We would expect that only the first order has the discount rate applied. However, since calling the builder methods affects the builder's state, the second order will have a discount rate applied as well! 

One way to solve this is to add a method that returns a copy of the builder.

```java
public class OrderBuilder {
    // ...
    
    private OrderBuilder(OrderBuilder copy) {
        this.orderId = copy.orderId;
        this.customer = copy.customer;
        this.orderItems = copy.orderItems;
        this.discountRate = copy.discountRate;
        this.giftVoucher = copy.giftVoucher;
    }

    public OrderBuilder but() {
        return new OrderBuilder(this);
    }

    // ...
}
```

Now we can make sure that the changes from previous uses do not leak to the next one.

```java
  @Test
  void buildSimilarOrders() {
      OrderBuilder coffeeMugAndTeaCup = anOrder()
              .with(anOrderItem().withName("Coffee mug").withQuantity(1))
              .with(anOrderItem().withName("Tea cup").withQuantity(1));

      Order orderWithDiscount = coffeeMugAndTeaCup.but().withDiscountRate(0.1).build();
      Order orderWithCouponCode = coffeeMugAndTeaCup.but().withCouponCode("HALFOFF").build();
  }
```

There is still room for human error, so if we want to be safe, we could make the "with" methods always return a copy of the builder.

## Summary

Test data builders help to hide syntax noise related to creating objects and make the code easier to read. Test data builders also make the code more descriptive and the tests less brittle.

Passing builders as arguments to other builders allows for making the code more compact. Using factory methods to create the builders hides the noise of constructing the builder classes.

The example code for this article can be found on [GitHub](https://github.com/arhohuttunen/write-better-tests/tree/main/test-data-builder).
