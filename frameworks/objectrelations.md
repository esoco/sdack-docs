---
description: >-
  Summary: The ObjectRelations Framework provides an extension of
  object-oriented programming for the modelling of relations between objects.
---

# ObjectRelations

### Introduction

Several years ago, after working on multiple Java projects I noticed that a considerable amount of the work for each project consisted of re-creating very similar code. After further contemplation I realized that although object-orientation allowed to model the single components of a problem domain \(the objects\), it did not provide a generic way to model the relations of these components with each other. But building the relations between objects is a major and maybe even the most important part of the software development process.

The ObjectRelations concept and framework that is described here addresses this issue. It does so by providing a generic mechanism for the modelling of the relations between objects. It is built on the foundation of object-orientation and consists of only a few new elements. That makes it easy to understand and it's application simple, even in existing projects.

Due to the generality of the idea the obvious way to implement this concept would be as an extension of an object-oriented programming language. Such a solution would need additional specifications, keywords, and tools like compilers, IDE support, etc. But the problem with such an approach is that it would require fundamental changes to the development process. And similar projects in the past have shown that it is difficult if not impossible to convince developers to switch to such new technologies if the tools lack maturity. And that which will almost certainly be the case with every new technology.

For that reason the ObjectRelations framework is implemented as a standard Java library that can be easily applied to any project, new or existing. Using relations therefore doesn't require any special tools or changes to the development process, only a new approach to solve the problems at hand. The restrictions imposed by implementing the concept as a library compared to a language feature are actually quite minor and can be circumvented without too much effort. 

Although the current implementation is for the Java programming language, the concept can be applied to any object-oriented language. Some more recent languages may even support an easier integration than Java. For example, in Kotlin it would be possible to use extension functions to provide native relation support for any object \(which is not possible in Java\). And because Java code can be used directly from Kotlin code \(and in the most JVM languages\) the ObjectRelations Java library can be used directly in a Kotlin project. 

While the concept has only recently been published, it has already been used successfully in production for years. Additional frameworks that are built upon object relations have simplified typical development tasks like object persistence and business process modelling. Development time and code size have been reduced considerably by applying relation-based programming.

### The Framework

As described above, the idea behind the framework is to model the relations between objects. So, what exactly are relations? Essentially, a relation is a reference from one object to another. This is the same as any other object reference in Java. But in the ObjectRelations concept, this reference is a distinctive object of it's own, not just an object pointer managed by the runtime environment. And the reference is now called a _relation_. 

Like the objects in object-orientation, relations need to be created from a template. In the case of objects the template new instances are create from is the class. For relations, the template is called a _relation type_. A relation type defines the properties and behavior that a certain relation of that type has. Like classes relation types are defined during development and are applied when executing an application to create or modify relations.

{% hint style="info" %}
**This is the key point of the concept:** relations between objects are modeled as objects themselves, with their own state and behavior. This allows to abstract typical properties of inter-object relations into relation types which can then be re-applied in different areas and projects without the repetition of code.
{% endhint %}

A very useful "side-effect" of relations becoming objects is that relations \(and relation types\) can also have relations. This allows to apply meta-information to any relation-enabled \(= relatable\) object. And because this metadata is based on relations it can easily be re-used for different purposes and projects. In fact, the availability of meta-relations constitutes one important advantage of the ObjectRelations concept over "non-relational" programming.

To implement this concept the ObjectRelations framework defines three core elements: relation types, objects that contain relations, and the relations themselves. The following sections explain these elements in detail. The base package for all classes is `org.obrel`, with the main classes in `org.obrel.core`. The [source code is available on GitHub](https://github.com/esoco/objectrelations) and the library can be easily included as a _Gradle_ or _Maven_ dependency.

The framework has only one dependency, to the project esoco-common. That project contains some essential classes that are also used in other projects that cannot depend on ObjectRelations. Besides the `org.obrel` package the framework also contains some classes in the `de.esoco.lib` package. These classes provide some fundamental classes that are used by the ObjectRelations framework although they are not part of the concept. Therefore the packages have been separated.

### Relation Types

As mentioned above, relation types are the template from which new relations are created at application runtime. Technically, relation types are instances of the class `RelationType`. That class can either be used directly or it may be sub-classed. Inheriting from `RelationType` allows to create extended relation functionality, although the base class is sufficient for many applications.

The class is declared as a generic type: `RelationType<T>`. The type parameter T stands for the datatype of the target objects to which relations of that type refer to. It ensures type safety when relations are accessed. The type of the source objects, i.e. the objects that relate to the targets, is not constrained  because relations of a certain type can be set on any source object.

Apart from the target type the only other required property of a relation type is it's name. The name is a string that needs must be unique in a running application. Like the packages of classes it may be prefixed with a namespace, separated by a dot. Although the name and namespace of a relation type can be provided explicitly the framework also provides a mechanism to assign the name automatically with a namespace that is based on the name and package of the enclosing class.

An optional property of relation types are relation type modifiers \(defined in the enum `RelationTypeModifier`\). Similar to Java modifiers for classes, methods, and fields they control the behavior and visibility of relation types. For example, relations with a type that has the modifier PRIVATE  are only visible in the context the type has been declared in, even if the relations of an object are iterated over. And relations that are declared as FINAL can only be assigned once.

#### Creating Relation Types

Like classes, most relation types are declared once at development time and are then used to work with relations of these types when the program executes. Such declarations are typically done as final static constants. But it is also possible to create types at runtime to dynamically create new relation types based on the state of an application, e.g. based on user input.

In principle, new relation types can be created by invoking a public constructor of `RelationType` with the type's name and datatype. And in the case of dynamically created relation types that is the recommended way. But for the more typical way of declaring static relation types this would require a lot of redundancy with the potential for errors. The name of the relation type needs to be provided to the constructor but also the static final constant declaration would need to have a name. And obviously these names should be the same to prevent inconsistencies. The same is true for the target datatype which is expected \(in form of it's class\) by the constructor to ensure type safety.

To make the declaration of static relation types simpler the framework provides a set of static factory methods in the class `RelationTypes`. These methods create new relation types in a special state that indicates their static usage. If used with static imports they allow a concise declaration of static relation types as this example from `org.obrel.type.StandardTypes` shows:

```java
public static final RelationType<String> NAME = newType();
```

The function `newType()` creates a new relation type \(with an optional modifiers parameter\). This type is then assigned to the final static constant `NAME` which represents a relation type with a string target type. By the way, the naming is according to the Java coding standards which require the names of constants to be uppercase. 

By using these factory methods the declaration of static relation types in a class can be performed easily without the need to keep declarations and instances in sync with respect to the name and datatype. The only question is, how to the new relation type instances "know" their name and target type? The answer is: they don't \(yet\). `newType()` and it's siblings create the new relations types so that the framework can recognize them as not \(yet\) initialized. A try to access a relation with such a type will result in an exception to be thrown. To avoid this the relation type instances need to be synchronized with their declarations first.

As mentioned before the ObjectRelations concept in principle is a new language feature. If implemented as such it would have it's own compiler that would check relation type declarations. But to avoid the need for such new tools the concept it has been chosen to implement it as a simple Java library. However, to complete the uninitialized relation types something like compilation is required. For that purpose the framework provides the method `RelationTypes.init(Class)`. It must be invoked once for each class that declares relation types with the factory methods. The typical way to do so would be to add a static initializer after the declaration of all relation types:

```java
static {
    RelationTypes.init(CurrentClass.class);
}
```

This "compilation" will evaluate all relation type constants in the class, checking for declaration errors and assigning them a name that is composed from the name of the declared constant with the namespace of the class \(including the class name\). This naming ensures that equally named relation types in other classes don't conflict with each other.

The need for an initialization in the static block is a small downside caused by the decision to implement the concept in form of a library. But as static relation types are typically declared in relatively few places adding this block isn't really a big deal. Furthermore it can be mitigated by applying the call to `init()` automatically if it is possible. This is done by some of the frameworks that are built upon relations, like persistent entities and business processes.

#### Standard Relation Types

When creating relation types it is good practice to identify common kinds of relations and to collect them in some central place so that different applications \(or parts of an application\) don't need to create the same types again. After all, the sharing of relation types is a key point of the ObjectRelations concept. The framework itself already does this by providing the two classes `StandardTypes` and `MetaTypes`. The former contains a collection of typical property relations \(like the `NAME` example above\) that can and should be re-used wherever it's applicable. `MetaTypes` on the other hand declares types that are intended to be used as metadata on other relations.

#### Extending `RelationType`

Although default relation types are sufficient for man uses it is also possible to extend the default functionality. A simple way to do so is by providing certain functions \(as instances to the relation type constructor or factory method. One of these functions can provide a default value for non-existing relations and the other produces initial values for new relations. The difference is that querying a relation with a type that has a default value will not create a new relation but only return the default. Initial values on the other hand will create a new relation when a relation is queried for the first time.

Another way to create extended relation types is by subclassing `RelationType`. The package `org.obrel.type` contains some examples of  such types. Such types can override the methods of the base class and add their own functionality. Typical methods to be overridden would be `addRelation()`, `newRelation()`, or `deleteRelation()` to intercept the respective points in the life-cycle of relations.

### Related Objects

To use relations they need to be added to the objects that should relate to the target objects. Were the concept implemented as a language extension this could be possible on any existing object. But as a simple library that cannot be achieved in pure Java \(at least not directly\). Instead, the framework provides a separate class that can be used as the base of all relation-enabled object: `RelatedObject`. A project that wants to apply relations should simply derive it's classes from this base class.

Obviously there are cases were subclassing is not an option, most notably when using external libraries \(including the Java standard libraries\). For such cases the Framework provides the method `ObjectRelations.getRelatable(Object)` which returns a relation-capable object for an arbitrary object. If the input object is already capable of holding relations it is returned unchanged. Otherwise the associated relatable object is cached so that subsequent invocations return the same object \(including any relations that have already been set on it\). The association are stored in week references so that they are automatically removed when the original object no longer exists.

{% hint style="info" %}
#### Footprint

For readers concerned that subclassing might have an adverse effect on the size of instances: `RelatedObject` contains only a single additional field. This is an object reference which initially points to a default object. Only when relations are created on such an object will more memory be used \(for the relations\).
{% endhint %}

#### The `Relatable` Interface

The main implementation class for objects that can hold relations is `RelatedObject`. However, the functionality of objects that can have relations to other objects is defined in the `Relatable` interface which is also implemented by `RelatedObject`. All general relation-specific methods and functions in the framework expect or return an instance of `Relatable`. In most cases the default implementation should be sufficient but if need arises it is also possible to create a different implementation of `Relatable`. Most of it's methods have default implementations, only a handful of abstract methods need to be implemented.

#### Accessing Relations

Accessing relations in a relatable object is simple and straightforward. Most methods expect a relation type as their first \(or only\) argument. For example, setting and retrieving the name of an object would look like this:

```java
import static org.obrel.type.StandardTypes.NAME;

Relatable aObject = new RelatedObject();

aObject.set(NAME, "HelloRelations");
String sName = aObject.get(NAME);

```

This shows that accessing relations is not much different than accessing normal Java properties, especially when using static imports. Maybe a bit uncommon is the uppercase notation of the relation type, but that is due to the fact that the static final relation type declarations follow the Java convention for the naming of constants. On the other hand this notation makes the use of relations easily distinguishable from other code.

This example also shows that instances of `RelatedObject` can be used directly without subclassing. Other than instances of `java.lang.Object` a basic related object can be very useful this way because it can accept arbitrary relations right away. Because a typical usage of this is to create objects to carry parameters between parts of an application the framework contains the subclass `Params` for this purpose. It implements additional interfaces that make using relations for this purpose even simpler.

Besides setting and querying relations relatable objects provide additional methods to delete or test for the existence of relations. It is also possible to iterate over or some relations of an object. The latter can also be done with the modern stream API of Java by invoking `streamRelations()`. Furthermore each relatable object has two additional set methods that allow to define relations that are processed by functions:

* `<T, I> set(RelationType<T> rType, Function<I,T> fResolver, I rIntermediateValue)`

  This sets the relation of type T to an intermediate value or type I. Only if the relation is queried the intermediate value will be resolved to the actual target type by invoking the resolver function. One possible application would be to provide a URL string as the intermediate value and a function that retrieves the contents of that URL.

* `<T, D> transform( RelationType<T> rType, InvertibleFunction<T,D> fTransformation)`

  A transformation is similar to an intermediate relation, only that the value is always stored in a different format than that of the actual target value. For that purpose the argument must be an `InvertibleFunction`, which is a function that can be invoked in both directions. This could be used to always store a relation in an encrypted format, for example.

{% hint style="info" %}
#### Functional Programming

As the previous paragraphs showed, functional programming is used in several parts of the framework. As the framework development started before Java 8 which introduced functional elements in the standard libraries it contains it's own Function classes. With the advent of Java 8 they have been made fully compatible so that they are interoperable with the Java classes. But they still are more extensive as the example of `InvertibleFunction` shows.

Speaking of functions: the `RelationType` class also implements the `Function` interface. The expected input value is a `Relatable` object and the result will be a value of their target type T. This allows to use relation types directly in function chains when processing relatable objects, without the need to wrap the calls into lambda expressions. And a predicate that can be used to test relatable objects can be created with `RelationType.is(Predicate<T>)`.

Because the function classes are not specific to the ObjectRelations concept they reside in the `de.esoco.lib` package mentioned earlier.
{% endhint %}

### Relations

The third building block in the ObjectRelations framework are the relation objects themselves. These are instance of the `Relation` class but is normally neither necessary nor advisable to create a subclass of it. Relations serve mainly as containers for the relation data \(type and target object\) and meta-informations. Instances are created automatically when relations are set on objects or queried with a relation type that produces an initial value. They are also managed automatically by the `Relatable` implementation \(which typically is `RelatedObject`\). They are in most cases accessed indirectly through the methods of `Relatable`.

As mentioned before relations are also relatable objects. That means that they can have \(meta-\) relations as well. When a relation is set on an object the set method returns the relation. This allows to immediately set a meta-relation like in this extension of the previous example:

```java
aObject.set(NAME, "HelloRelations").annotate(MAX_LENGTH, 40);
```

This sets the name of the object and then annotates the `NAME` relation with a meta-information that the maximum length of the attribute is 40 characters. Such an information could then be used by user interface code, for example, to limit the maximum size of an input field. By the way, the `annotate()` method is only an synonym for the standard set method and exists only \(in relations and relation types\) as a semantic alternative to indicate the setting of metadata.

#### Aliasing

One advanced but interesting feature of relations is the possibility to define aliases of relations with a different relation type. This can be done by invoking one of the `aliasAs()` or `viewAs()` methods. They create new relations that are backed by the original relation they have been invoked on. In the case of views the cross-reference is read-only but aliases can be updated through all relations. There are even variants of the methods that accept a conversion function, thus allowing aliases to have different datatypes.

### Relation Listeners

The ObjectRelations framework also provides an event listener mechanism that allows applications to be notified of relation changes. The notification is done by sending instances of `RelationEvent` to implementations of the `EventHandler` interface. This is a functional interface which can therefore be implemented with lambda expressions. The event object contains an event type that describes the relation modification as well as information about the involved objects. The listener mechanism is itself implemented with relations and registering a listener would be done as in this example:

```java
aObject.get(RELATION_LISTENERS).add(e -> processRelationEvent(e));
```

The target datatype of the relation type `RELATION_LISTENERS` is the class `EventDispatcher`. This is a universal event handling class in the framework which manages an arbitrary number of event handlers. The relation type is defined as a type with an initial value. That means if the relation is queried but doesn't exist yet a new relation with an new target will be created. In this case the value will be a new and empty event dispatcher. This allows to call the `add()` method directly on the returned value because it will always be a valid event dispatcher. A listener can be removed later by calling `remove()`.

Relation listeners can be set on any relatable object. The registered listener\(s\) will then be notified of any change in the relations of the object, regardless of the relation type. But sometimes it may be of more interest to be notified of changes of relations of a certain type in arbitrary objects. That can be achieved by adding the listener to the `RELATION_TYPE_LISTENERS` of a relation type instance instead. The relation `RELATION_LISTENERS` may also be set on a relation type but such a listener will only be notified of changes of the relations in that particular type instance \(remember, relation types are relatable themselves\).

And finally it is possible to set `RELATION_UPDATE_LISTENERS` on relation object. These will then be notified only of changes to that particular relation \(again, RELATION\_LISTENERS would be notified of changes in the relations of that relations\). If `RELATION_TYPE_LISTENERS` or `RELATION_UPDATE_LISTENERS` are set on other objects than their designated target types they will be ignored.

### Working with ObjectRelations

Originally the driving force behind the development of ObjectRelations was the idea of modelling complex relations. Such relations have their own functionality, like the listener mechanism in the previous chapter. And although this is a scenario where the generic character of relations can be very useful the first applications built on top of the framework showed that the principle is at least as useful even for simple relations that are basically nothing else than simple object attributes.

Take the previously shown relation type `StandardTypes.NAME` as an example. This relation references a simple string value and the first tendency would be to model such a property as a simple field in a class. But upon further consideration it becomes clear the the `NAME` relation is much more than just an attribute. To begin with, many classes can have a name that is used to identify and/or display their objects. With typical code each class would need to implement that field and the corresponding access methods. With relations the `NAME` type only needs to be defined once and can then be applied where it is needed. Each object can have a name if desired, with no additional coding effort at all.

Furthermore it is simple to check whether a certain relation exists on an arbitrary object, by simply querying the relatable object with `hasRelation(RelationType)`. And if not it can just be applied by setting it. It is also possible to iterate over the relations of any relatable object. With the traditional method it would be necessary to resort to reflection to search for a matching access methods, with all the exception risk, type-safety problems, and inefficiency that implies.

Another important aspect is that relations can easily be annotated with [listeners](objectrelations.md#relation-listeners), making it simple to react to changes even of such simple properties. This is not possible with object fields without additional code, not even by using reflection. Being able to listen to attribute changes makes solutions possible that could otherwise only be achieved with much additional implementation effort.

And finally, relations and relation types can carry meta-information, making it possible to control application behavior by simply annotating relations or relations types with the appropriate metadata. A previous example mentioned to annotate the `NAME` property with a maximum length which could then be checked by user interface code to limit the maximum input length. Similar examples would be to set display patterns or formatting information. The `MetaTypes` class contains several generic metadata relation types that can be used for such purposes. Such annotations could also be set on relation types as defaults and on relations as overridden values to modify them for specific purposes.

So, all these factors make relations very helpful even for simple object attributes. The other frameworks that are built upon ObjectRelations make use of these features to simplify common software developments tasks without the need to use reflection or separate configuration files. One example is the _esoco-business_ framework which performs object-relational mapping with entities that are relatable objects. The relation types in the Entity classes define the persistent attributes of the entities and the framework uses the mechanisms described above to synchronize these entities with relation databases. Another example from the same project is the modelling of business processes with an automatic mapping to web application user interfaces based on the process relations.

{% hint style="info" %}
#### Performance and Memory Impact

Some readers may think that all these positive features may come with a disadvantage  in performance and memory usage because of the more complex structure of relations compared with simple object fields. While this is true to some degree it is in most cases not a noticeable factor. Relations are internally stored in a map and looking up a relation obviously takes a bit longer than accessing a field. But compared to the other processing that is done by an application this is not a real factor. For example, the mentioned entity framework has been used for mass data processing and the handling of the relations had no measurable effect on the program execution.

On the other hand relations can even reduce processing time and memory usage if applied wisely. Because relations only exists if they have been created they can reduce these factors in dynamic implementations where only a few objects have certain properties. This is of even more importance if more complex properties like collections or maps are used. When using objects with field declarations all fields will consume memory, even if they are not used. And if such objects need to be searched for properties with reflection all fields need to be iterated through while an empty relation map can be detected immediately.

And as always the classic rule stated by Donald Knuth should be applied: _Premature optimization is the root of all evil_.
{% endhint %}

#### Dependency Injection

Another very useful application of ObjectRelations is dependency injection. Many frameworks that are explicitly built for that purpose require either special code structures or even separate configuration files to configure and provide resources to an application. With relations this is basically a by-product of the generic properties of relation-based development. Just define relation types for the dependency objects to inject and query them from the code that needs to access these object. If necessary these relation types can have default or initial values and they may contain metadata with additional information about the resource. Then during application startup, initialize the corresponding relation \(or only the relation types\) from the application configuration so that the resources are available to all other parts of the application. And because relations types can easily be re-used, the same resources can be accessed in multiple projects without additional code.

#### Advanced Usage

While relations are already very useful in their basic form they can also be used in their intermediate or transformed variants or with derived relation types that provide additional functionality. For example, the entity persistence framework uses intermediate relations to store references to other entities with the primary of the referenced object. Only if such a reference is accessed will a database query be performed for the full entity data.

A demonstration of an "active" relation type with a specific implementation can be found in the `MetaTypes` class. It contains a flag meta-relation type called `IMMUTABLE` that is backed by a special sub-type of `RelationType`. After that type is set on an arbitrary relatable object the object's relation cannot be modified anymore \(removing the flag isn't possible as well\). The immutability includes all references to collections and maps which will be wrapped into their unmodifiable counterparts. It also includes all references to other relatables recursively. This shows that a single relation implementation can be used for a multitude of applications without any additional code.

Instead of statically defined relation types it is also possible to create new relation types at runtime. This can be used to create dynamic systems that store values in relations that are only known when the application executes. The business process framework uses such dynamic relation types when it generates individual user interfaces for the users of a web application.

#### Limitations

The main limitation of the current implementation is that it isn't a real language extension but only a library. Therefore it is not possible for existing Java objects to \(directly\) have relations, only subclasses of the relatable base classes can have them. Furthermore the declaration of relation types is more verbose than is desirable because it needs to adhere to the Java specification. The resulting usage of uppercase relation type names based on the Java naming conventions may also seem uncommon. And these static types requires a static initializer with an initialization call which probably causes the most common usage error of the library when it is not present.

The concept itself is quite limitless, although its full possibilities still need to be explored. But that versatility comes with it's own side effects. Applying relations to a project requires careful planning to still have a well-structured result. Because relations can be so ubiquitous it is easy to apply them without a defined structure. And as relations can be set on any relatable object it needs to be avoided to set them on the wrong object\(s\), for example when using them for dependency injection. But good design is necessary in almost any development environment, especially if it is so dynamic and generic.

