---
description: >-
  The ObjectRelations Framework provides an extension of object-oriented
  programming for the modelling of relations between objects.
---

# ObjectRelations

### Introduction

Several years ago, after working on multiple Java projects I noticed that a considerable amount of the work for each project consisted of re-creating very similar code. After further contemplation I realized that although object-orientation allowed to model the single components of a problem domain \(the objects\), it did not provide a generic way to model the relations of these components with each other. But building the relations between objects is a major and maybe even the most important part of the software development process.

The ObjectRelations concept and framework that is described here addresses this issue. It does so by providing a generic mechanism for the modelling of the relations between objects. It is built on the foundation of object-orientation and consists of only a few new elements. That makes it easy to understand and it's application simple, even in existing projects.

Due to the generality of the idea the obvious way to implement this concept would be as an extension of an object-oriented programming language. Such a solution would need additional specifications, keywords, and tools like compilers, IDE support, etc. But the problem with such an approach is that it would require fundamental changes to the development process. And similar projects in the past have shown that it is difficult if not impossible to convince developers to switch to such new technologies if the tools lack maturity. And that which will almost certainly be the case with every new technology.

For that reason the ObjectRelations framework is implemented as a standard Java library that can be easily applied to any project, new or existing. Using relations therefore doesn't require any special tools or changes to the development process, only a slightly different approach to solve the problems at hand. The restrictions imposed by implementing the concept as a library compared to a language feature are actually quite minor and can be circumvented without too much effort.

Although the current implementation is for the Java programming language, the concept can be applied to any object-oriented language. Some more recent languages may even support an easier integration than Java. For example, in Kotlin it would be possible to use extension functions to provide native relation support for any object \(which is not possible in Java\). And because Java code can be used directly from Kotlin code \(and in the most JVM languages\) the ObjectRelations Java library can be used directly in a Kotlin project. 

### The Framework

As described above, the idea behind the framework is to model the relations between objects. So, what exactly are relations? Essentially, a relation is a reference from one object to another. This is the same as any other object reference in Java. But with the ObjectRelations concept, this reference is a distinctive object of it's own, not just an object pointer managed by the runtime environment. And the reference is now called a _relation_. 

Like the objects in object-orientation relations need to be created from a template. In the case of objects the template is a class. In the case of relations, the template is called a _relation type_. A relation type defines the properties and behavior that a certain relation of that type has. Relation types are defined during development and are applied during the execution of an application to create or modify relations.

{% hint style="info" %}
**This is the key point of the concept:** relations between objects are modeled as objects themselves, with their own state and behavior. This allows to abstract typical properties of inter-object relations into relation types which can be re-applied in different areas and projects without the repetition of code.
{% endhint %}

An interesting and very useful "side-effect" of relations being objects is that relation objects and relation types can also have relations. This allows to easily apply meta-information to any relation-enabled \(= relatable\) object. And because these meta-data is based on relations it can easily be re-used for different purposes and projects.

To implement this concept the ObjectRelations framework defines three core elements: relation types, objects that contain relations \(to other objects\), and the relations to other objects. The following sections explain these elements in detail. All classes of the framework are placed in children of the `org.obrel` package. The [source code is available on GitHub](https://github.com/esoco/objectrelations) and the library can be accessed as a dependency from _Gradle_ or _Maven_.

### Relation Types

Relation types are to relation-oriented software development what classes are to object-orientation. They represent the abstraction of a relation between objects as described above. The abstract base class of all relation types is org.obrel.core.RelationType. To create a new kind of relation between objects a developer only needs to subclass RelationType and implement or override the necessary methods to control the behavior of relations with the new type.

The class RelationType is a generic type. It uses the generics feature of Java 5 to declare the type of the objects that will be referenced by relations of this type. By using generic types relation-based operations will be as type-safe as statically typed code, reducing the need of type casts and therefore enhancing the readability of relation-based code.

The generic declaration of the class is not so obvious at first glance: RelationType\. Instead of a single generic parameter defining the referenced type there are two different parameter T and R. The idea behind this is that not all relations will reference an object directly, especially when defining relations to objects in a different context \(like another application, on the network, in a persistent storage etc.\).

To support such scenarios the referenced type of a relation is split in two parts. First there is the relation target, defined by the generic type T. This is the type of the value that is actually stored by a relation of the respective type. Second is the resolved type R, which is the type that will be returned when the relation is queried. A typical example is a relation that accesses remote objects on a network. Instead of storing a heavyweight remote object reference, the relation target will instead contain a URL \(or a similar identifier\). Only if the relation is actually accessed will the target value be resolved and returned as a value of type R. All this is handled by the relation type subclass and is completely transparent to application code.

To implement a new relation type the minimum a subclass of RelationType must do is to implement the single abstract method resolve\(\). The purpose of this method is to take the target value of a particular relation and to convert it to the resolved value of the type. All other methods have default implementations that may also be overridden to modify the behavior of a relation type or to add additional functionality. Please see the framework documentation for details about these methods.

While the separation of target and resolved types facilitates the implementation of sophisticated relation types, many relation types will be used in a local context where the target and resolved values will be the same \(i.e., T and R will be identical\). To support this the framework contains the base class PropertyType\. It's resolve\(\) method has been implemented to simply return the target value. The name expresses that relations of such a type are very similar to the properties of objects in object-oriented applications because such properties are nothing else than references to other objects.

### Related Objects

As a relation is a reference to a certain object there must also be an object from which the relation originates. The class org.obrel.core.RelatedObject serves as this starting point for relations. Code that wants to use relations should subclass it in as many classes as possible. RelatedObject manages the relations associated with an instance and provides methods to access and modify them. It is to the ObjectRelations framework what the class Object is to the standard class library of Java.

One of the few disadvantages of ObjectRelations being implemented as a library instead of a native language features is that objects of existing classes cannot contain relations because they are obviously not derived from RelatedObject. But the framework provides a way to manage relations for such objects too. The class ObjectRelations contains several static methods that support working with relations. One of these is getRelatable\(Object\)which returns a related object that can be used to manage the relations for an arbitrary Java object.

The returned object will be an instance of the org.obrel.core.Relatable interface which defines the methods to access and modify the relations of an object. It is implemented by the RelatedObject class and is not intended to be implemented by classes outside of the ObjectRelations framework. But it should be extended by interfaces that want to signal that their implementations will \(and must\) be instances of RelatedObject.

As a rule of thumb it can be said that each class in a relation-oriented project should be a subclass of RelatedObject, like each class in a Java project is a subclass of Object. The overhead compared to normal objects is very low so it should probably never harm to subclass it.

### Relations

Relations are basically the instances of relation types, in the same way as objects are the instances of classes in object-oriented code. When a relation is set on a related object a new instance of the class org.obrel.core.Relation \(or a subclass\) will be created. The main difference to objects is that relations are created automatically and that they are typically accessed and managed through the methods of the RelatedObject class \(which are defined in the Relatable interface\). In most cases application code doesn't need to access relation instances directly.

But it is possible to query the relations from related objects and for some advanced scenarios it makes sense to access relations directly. One very interesting property of relations is that they are also related objects \(like almost all classes in the ObjectRelations framework\). This means that it is possible to set relations on relations \(and on relation types as well\). Such relations can seen as meta-relations or, as is the ObjectRelations terminology, annotations. The Relation class contains additional methods for accessing annotations on both the relation and it's type.

Meta-relations provide an additional dimension when working with relations. For example, a user interface library could evaluate the annotations of the relations of an object that is to be visualized and determine certain attributes from the meta-relations, like mandatory fields, default values, value ranges, data formats, hierarchy information, etc. Indirect meta-relations could even retrieve such attributes over the network or from some configuration storage. And all this in a fully generic way that will work with any related object and can be used transparently by all frameworks that is enabled to handle relations.

#### Working with relations

The ObjectRelations framework already contains several common relation types in the package org.obrel.type. Let's have a look at a declaration from the class StandardProperties which contains standard property types:

```java
public static final PropertyType\<String\> NAME;
```

This declaration defines a property type constant with a string datatype that is called NAME. Please note that because ObjectRelations is a standard Java library and not a language extension it follows the standard naming convention for Java code. According to this convention the names of constants must be all upper case, which is why relation types constant are named that way. The following code shows the typical usage of a relation of the above type:

```java
import static org.obrel.type.StandardProperties.NAME;

RelatedObject aObject = new RelatedObject();

aObject.set(NAME, "HelloWorld");

String sName = aObject.get(NAME);

aObject.deleteRelation(NAME);
```

Please note from this example that RelatedObject is not an abstract class and can be instantiated directly like java.lang.Object. But unlike Object it often even makes sense to use it directly because by setting relations on an instance arbitrary informations or even functionality can be added.

The example makes use of another new feature from Java 5: static imports. This allows to use relation types almost like object-oriented attributes as is shown above for the property NAME. Apart from the slightly different naming the main difference to accessing attributes in object-oriented code is that the property name is not part of the method name but rather a method argument. And this illustrates one big advantage of using relations even for simple attributes: relations are dynamic compared to the static attributes of object-oriented code. While the attributes of classes must be defined at development time relations can be added \(or removed\) even at runtime as the example shows. This allows to create dynamic applications that are not easily possible in object-oriented code.

As explained above, properties are the most simple form of relations. And for many applications it may be sufficient to mainly declare additional property types without subclassing. But let's have a look at a slightly more complex example of subclassing a relation type for the common case of implementing a listener mechanism.

To support a certain listener interface it is typically necessary to implement the code to manage and notify the listeners again in every class that has to support them. Sometimes it may be possible to build the code into a base class, but due to the lack of multiple inheritance this often is not possible, especially if multiple listener types need to be supported. Extracting the listener management into a separate class still requires to implement delegating code. With ObjectRelations you only need to implement the code once in a relation type. And it is so simple that it can be done in an inner class declaration, as an example for the ActionListener interface shows:

```java
public static final 
   ListenerType\<ActionListener,ActionEvent\> ACTION\_LISTENERS = 
      new
ListenerType\<ActionListener,ActionEvent\>("ACTION\_LISTENERS")
      {
         protected void notifyListener(ActionListener rListener,
                                       ActionEvent   rEvent)
         {
            rListener.actionPerformed(rEvent);
         }
      };
```

Again this short examples demonstrates several aspects of the ObjectRelations framework. First it shows that relation types support inheritance like all other classes. The class ListenerType is provided by the framework and implements the base functionality for typical event listener relations. The action listener type only needs to subclass it, define the listener and event classes as the generic types, and implement a single method which performs the actual listener notification. Because of the generic types used the code is completely type-safe and does not need any type casting.

Furthermore, the example illustrates that relations can refer to arbitrary complex objects. The base class declaration of ListenerType is actually PropertyType\&gt; where L stands for the type of the listener interface \(ActionListener in the example\). The result is that a relation of type ACTION\_LISTENERS refers to a list of action listeners. By convention relation types that refer to multiple objects are named in plural.

Finally, this example indicates that all relation types have a name \(the string argument to the constructor\) that should be identical to the name of the corresponding constant declaration. The framework contains tools to enforce the identity of declaration and instance names.

The following code shows how the ACTION\_LISTENERS relation type is used to set an action listener on an object as an instance of an anonymous inner class:

```java
aActionSource.get(ACTION_LISTENERS).add(new ActionListener()
   {
      public void actionPerformed(ActionEvent rEvent)
      {
         // handle action event here
      }
   });
```

In this example aActionSource would be a related object that is capable of generating action events. How exactly this is done depends on the respective implementation and is not the concern of the ACTION\_LISTENERS relation type. But it would also be possible to create relation types which perform the listener notification automatically. For example, another relation type referencing a button could register itself as an action listener on the button and notify action listeners through the listener type if an event occurs. Notifying all listeners of a ListenerType subclass is very simple:

```java
ACTION_LISTENERS.notifyListeners(this, rEvent);
```

Where this stands for the object that wants to notify it's listeners and rEvent is the action event that occurred. If another relation type wants to notify the listeners of a relation's parent object it would use that parent object as the first argument instead.

This concludes the introduction to the main elements of the ObjectRelations framework. The abstraction and reusability it provides can reduce the complexity and size of typical object-oriented code considerably. And because the framework itself is very lightweight it can be added to almost any project, even in constrained environments.

