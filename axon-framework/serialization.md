# Serialization

The flow of messages between (micro)services and storage of events requires preparation of the messages through serialization.
Axon uses the `XStreamSerializer` by default, which uses [XStream](http://x-stream.github.io/) to serialize into and deserialize from XML.
XStream is reasonably fast, and the result of serialization is human-readable.
This makes it quite useful for logging and debugging purposes.

The `XStreamSerializer` allows further customization if that's required.
You can, for example, define aliases for specific packages, classes, or even fields.
In addition to being an excellent way to shorten potentially long names, you can also use aliases when class definitions of the serialized objects change.
For more information about aliases, visit the [XStream website](http://x-stream.github.io/).

Additionally, Axon provides the `JacksonSerializer`. 
This `Serializer` implementation uses [Jackson](https://github.com/FasterXML/jackson) to serialize objects into and deserialize from JSON.
It produces a more compact serialized form, while requiring those classes to stick to Jackson's conventions \(or configuration\).
The compact format makes it ideal for events, commands, and queries, as it minimizes the storage space and package size.

You may also implement your own serializer simply by creating a class that implements `Serializer` and setting it within Axon's configuration for the desired infrastructure components.

## Serializer Implementations

`Serializers` come in several flavors in Axon Framework and are used for various things. 
Currently, you can choose between the `XStreamSerializer` and `JacksonSerializer` to serialize messages \(commands/queries/events\), tokens, snapshots, deadlines and sagas in an Axon application.

As there are several objects to be serialized, it is typically desired to chose which serializer handles which object. 
To that end, the `Configuration` API allows you to define default, `message` and `event` serializers, which lead to the following object-serialization break down:

1. The Event `Serializer` is in charge of (de)serializing event message payload and metadata.
   Events are typically stored in an event store for a long period of time.
   This is the main driver for choosing the event serializer implementation.

2. The Message `Serializer` is in charge of (de)serializing the command and query message payload and metadata \(used in a distributed application setup\).
   Messages are shared between nodes and typically need to be interoperable and/or compact.
   Take this into account when choosing the message serializer implementation.

3. The default `Serializer` is in charge of (de)serializing the remainder, being the messages (except the payload and metadata), tokens, snapshots, deadlines and sagas.
   These objects are generally not shared between different applications, and most of these classes aren't expected to have some of the getters and setters that are, for example, typically required by Jackson based serializers.
   For example, a `QueryMessage` consists of a payload and `ResponseType`, which will respectively be (de)serialized using the `message` and the `default` serializer, the query request and response payload will be (de)serialized using the`message`serializer.
   A flexible, general-purpose serializer like [XStream](http://x-stream.github.io/) is quite ideal for this purpose.

By default, all three `Serializer` flavors are set to use the `XStreamSerializer`, which internally uses [XStream](http://x-stream.github.io/) to serialize objects to an XML format. 
XML is verbose, but XStream has the major benefit of being able to serialize virtually anything.

> *XStream and JDK 17*
> 
> Although XStream can "serialize virtually anything," more recent versions of the JDK impede its flexibility.
> This predicament comes down to XStream's reflective approach to finding out how to de-/serialize *any* object, which has become problematic with Java's intent to secure its internals.
> Hence, if you're using JDK 17, the chances are that objects (e.g., your sagas) intended for serialization require additional configuration.
> 
> On some occasions configuring XStream's security settings is sufficient.
> Other times you will have to introduce custom [`Converters`](https://x-stream.github.io/converters.html) to de-/serialize specific types.
> If you prefer not to deal with specific XStream settings, it might be better to use the `JacksonSerializer` throughout your Axon application.

XML's verbosity is typically fine when storing tokens, sagas, or snapshots, but for messages \(and specifically events\) XML might cost too much due to its serialized size. 
Thus for optimization reasons you can configure different serializers for your messages. 
Another very valid reason for customizing serializers is to achieve interoperability between different \(Axon\) applications, where the receiving end potentially enforces a specific serialized format.

There is an implicit ordering between the configurable serializer. If no event `Serializer` is configured, the event de-/serialization will be performed by the message `Serializer`. In turn, if no message `Serializer` is configured, the default `Serializer` will take that role.

See the following example on how to configure each serializer specifically, were we use the `XStreamSerializer` as the default and the `JacksonSerializer` for all our messages:

{% tabs %}
{% tab title="Axon Configuration API" %}
```java
public class SerializerConfiguration {

    public void serializerConfiguration(Configurer configurer) {
        // By default, we want the XStreamSerializer
        XStream xStream = new XStream();
        // Set the secure types on the xStream instance
        XStreamSerializer defaultSerializer = XStreamSerializer.builder()
                                                               .xStream(xStream)
                                                               .build();
        
        // But for all our messages we'd prefer the JacksonSerializer due to JSON's smaller format
        JacksonSerializer messageSerializer = JacksonSerializer.defaultSerializer();

        configurer.configureSerializer(configuration -> defaultSerializer)
                  .configureMessageSerializer(configuration -> messageSerializer)
                  .configureEventSerializer(configuration -> messageSerializer);
    }
}
```
{% endtab %}
{% tab title="Spring Boot AutoConfiguration - Configuration class" %}
```java
@Configuration
public class SerializerConfiguration {

   // By default, we want the XStreamSerializer
   @Bean
   public Serializer defaultSerializer() {
      // Set the secure types on the xStream instance
      XStream xStream = new XStream();
      return XStreamSerializer.builder()
                              .xStream(xStream)
                              .build();
   }

   // But for all our messages we'd prefer the JacksonSerializer due to JSON's smaller format
   @Bean
   @Qualifier("messageSerializer")
   public Serializer messageSerializer() {
      return JacksonSerializer.defaultSerializer();
   }
}
```
{% endtab %}

{% tab title="Spring Boot AutoConfiguration - Properties file" %}
```text
# Possible values for these keys are `default`, `xstream`, `java`, and `jackson`.
axon.serializer.general
axon.serializer.events
axon.serializer.messages
```
{% endtab %}

{% tab title="Spring Boot AutoConfiguration - YML file" %}
```yaml
# Possible values for these keys are `default`, `xstream`, `java`, and `jackson`.
axon:
    serializer:
        general: 
        events: 
        messages:
```
{% endtab %}
{% endtabs %}

## Serializer Tuning

Several things might be considered when the serialization process proofs to not be up to par with the expectations.

### XStreamSerializer

XStream is very configurable and extensible. If you just use a plain `XStreamSerializer`, there are some quick wins ready to pick up. XStream allows you to configure aliases for package names and event class names. Aliases are typically much shorter \(especially if you have long package names\), making the serialized form of an event smaller. And since we're talking XML, each character removed from XML is twice the profit \(one for the start tag, and one for the end tag\).

A more advanced topic in XStream is creating custom converters. The default reflection based converters are simple, but do not generate the most compact XML. Always look carefully at the generated XML and see if all the information there is really needed to reconstruct the original instance.

Avoid the use of upcasters when possible. XStream allows aliases to be used for fields, when they have changed name. Imagine revision 0 of an event, that used a field called `"clientId"`. The business prefers the term `"customer"`, so revision 1 was created with a field called `"customerId"`. This can be configured completely in XStream, using field aliases. You need to configure two aliases, in the following order: alias `"customerId"` to `"clientId"` and then alias `"customerId"` to `"customerId"`. This will tell XStream that if it encounters a field called `"customerId"`, it will call the corresponding XML element `"customerId"` \(the second alias overrides the first\). But if XStream encounters an XML element called `"clientId"`, it is a known alias and will be resolved to field name `"customerId"`. Check out the XStream documentation for more information.

For ultimate performance, you're probably better off without reflection based mechanisms altogether. In that case, it is probably wisest to create a custom serialization mechanism. The `DataInputStream` and `DataOutputStream` allow you to easily write the contents of the events to an output stream. The `ByteArrayOutputStream` and `ByteArrayInputStream` allow writing to and reading from byte arrays.

### Preventing duplicate serialization

Especially in distributed systems, event messages need to be serialized on multiple occasions. Axon's components are aware of this and have support for `SerializationAware` messages. If a `SerializationAware` message is detected, its methods are used to serialize an object, instead of simply passing the payload to a serializer. This allows for performance optimizations.

When you serialize messages yourself, and want to benefit from the `SerializationAware` optimization, use the `MessageSerializer` class to serialize the payload and metadata of messages. All optimization logic is implemented in that class. See the JavaDoc of the `MessageSerializer` for more details.

### Different serializer for events

When using event sourcing, serialized events can stick around for a long time. Therefore, consider the format to which they are serialized, carefully. Consider configuring a separate serializer for events, carefully optimized for the way they are stored. The JSON format generated by Jackson is generally more suitable for the long term than XStream's XML format. For more information on how to configure your`EventSerializer` to something different, check out the documentation about [Serializers](serialization.md).

### Lenient Deserialization

"Being lenient" from the `Serializer`'s perspective means the `Serializer` can ignore unknown properties.
If it thus was handling a format to deserialize, it would not fail when it is incapable of finding a field / setter / constructor parameter for a given field in the serialized format. 

Enabling lenient serialization can be especially helpful to accommodate different message versions.
This situation would occur naturally when using an event store, as the format of the events would change overtime.
But this might also happen between commands and queries if several distinct versions of an application are run concurrently.
A scenario when you would hit this is when going for a rolling upgrade pattern to deploying a new service.

To accommodate more closely with the desire to ignore unknown fields, both the `XStreamSerializer` and `JacksonSerializer` can be enabled as such.
How to achieve this is shown in the following snippet:

{% tabs %}
{% tab title="`XStreamSerializer`" %}
```java
public class SerializerConfiguration {

    public Serializer buildSerializer() {
        return XStreamSerializer.builder()
                                .lenientDeserialization()                        
                                .build();
    }
}
```
{% endtab %}

{% tab title="`JacksonSerializer`" %}
```java
public class SerializerConfiguration {

    public Serializer buildSerializer() {
        return JacksonSerializer.builder()
                                .lenientDeserialization()                        
                                .build();
    }
}
```
{% endtab %}
{% endtabs %}

### Default Typing

Sometimes the objects serialized by Axon will contain lists or collections of data.
For XStream, this poses no problem, as it will automatically add the type information to the serialized format.
Jackson does not do this out of the box, however.

You can configure the `ObjectMapper` to add default typing information, but the `JacksonSerializer's` builder also provides a method to enable this for you.
With `JacksonSerializer.Builder#defaultTyping`, you will automatically enable the addition of types to the serialized format for lists and collections.
Consider the following sample on how to enable default typing for the `JacksonSerializer`:

```java
public class SerializerConfiguration { 
    // ...
    public Serializer buildSerializer() {
          return JacksonSerializer.builder()
                                  .defaultTyping()
                                  .build();
    }
}
```

### ContentTypeConverters

An [upcaster](events/event-versioning.md#event-upcasting) works on a given content type \(e.g. dom4j Document\). 
To provide extra flexibility between upcasters, content types between chained upcasters may vary. 
Axon will try to convert between the content types automatically by using a `ContentTypeConverter`. 
It will search for the shortest path from type `x` to type `y`, perform the conversion and pass the converted value into the requested upcaster. 
For performance reasons, conversion will only be performed if the `canUpcast` method on the receiving upcaster yields true.

The `ContentTypeConverter` may depend on the type of serializer used. 
Attempting to convert a `byte[]` to a dom4j `Document` will not make any sense unless a `Serializer` was used that writes an event as XML. 
Axon Framework will only use the generic content type converters (such as the one converting a `String` to `byte[]` or a `byte[]` to `InputStream`) and the converters configured on the Serializer that will be used to deserialize the message. 
That means if you use a JSON based serializer, you would be able to convert to and from JSON-specific formats.

> **Tip**
>
> To achieve the best performance, ensure that all upcasters in the same chain \(where one's output is another's input\) work on the same content type.

If Axon does not provide the content type conversion that you need, you can always write one yourself by implementing the `ContentTypeConverter` interface.

The `XStreamSerializer` supports dom4j as well as XOM as XML document representations. 
The `JacksonSerializer` supports Jackson's `JsonNode` and `ObjectNode`.
