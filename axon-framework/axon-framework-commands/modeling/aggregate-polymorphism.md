# Aggregate Polymorphism

In certain cases it is beneficial to have a polymorphic hierarchy in aggregate structure. 
Subtypes in polymorphic aggregate hierarchy inherit `@CommandHandler`s, `@EventSourcingHandler`s
 and `@CommandHandlerInterceptor`s from the super aggregates. 
Based on `@AggregateIdentifier` the correct aggregate type is loaded and command is executed on it. 
Let's take a look at the following example:

```java
public abstract class Card {}

public class GiftCard extends Card {}

public class ClosedLoopGiftCard extends GiftCard {}

public class OpenLoopGiftCard extends GiftCard {}

public class RechargeableGiftCard extends ClosedLoopGiftCard {}
```

We can define this structure as Polymorphic Aggregate of type `GiftCard` and subtypes of `ClosedLoopGiftCard`,
 `OpenLoopGiftCard`, and `RechargeableGiftCard`. 
If there are handlers present on `Card` class, those will be present on all aggregates as well.

While modeling a polymorphic aggregate hierarchy it is important to keep these constraints in mind:

* It is not allowed to have a constructor annotated with `@CommandHandler` on abstract aggregate. 
The rationale for this is that an abstract aggregate can never be created.

* Having creational command handlers of the same command name on different aggregates in the same hierarchy is
   forbidden too, since Axon cannot derive which one to invoke.

* In a polymorphic aggregate hierarchy it is not allowed to have multiple `@AggregateIdentifier`
   and `@AggregateVersion` annotated fields.

## Registering aggregate subtypes

A polymorphic aggregate hierarchy can be registered via the `AggregateConfigurer` by invoking
 `AggregateConfigurer#withSubtypes(Set<Class<? extends A>>)`.

Do note that children of the parent aggregate that you do not register as a subtype will be **automatically**
 registered by the framework as a subtype.
So, in the following example, the `ClosedLoopGiftCard` is transitively registered as a subtype of `GiftCard`.
However, if there is a `LimitedRechargeableGiftCard extends RechargeableGiftCard` defined,
 the framework will not pick it up **automatically** since it is not a direct child of the parent aggregate.

```java
class AxonConfiguration {

    public AggregateConfigurer<GiftCard> giftCardConfigurer() {
        Set<Class<? extends GiftCard>> subtypes = new HashSet<>();
        subtypes.add(OpenLoopGiftCard.class);
        subtypes.add(RechargeableGiftCard.class);

        return AggregateConfigurer.defaultConfiguration(GiftCard.class)
                                  .withSubtypes(subtypes);
    }

    // ...
}

class GiftCard {
    // omitted implementation for brevity
}

class OpenLoopGiftCard extends GiftCard {
    // omitted implementation for brevity
}

class RechargeableGiftCard extends GiftCard {
    // omitted implementation for brevity
}
```

> **Polymorphic Aggregates in Spring**
>
> If you are using Spring,
>  Axon will automatically detect polymorphic aggregates based on the `@Aggregate` annotations and class hierarchy. 
> The `@Aggregate` annotation needs to be put on the shared parent class that contains the aggregate identifier,
>  as well as every subclass that is a potential instance type of that shared parent class.
