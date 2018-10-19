# Introduction

The **SDACK** project consists of a stack of multiple framework projects. Each of the stack levels defines an abstraction of a particular domain like process flow, user interface, or persistence. The purpose of these abstractions is to hide the complexity of the underlying technologies. This provides a foundation for the fast and easy creation of applications that are connected to existing or new business data.

The following illustration displays the full framework stack:

![SDACK](../.gitbook/assets/esoco-framework-stack.png)

A typical SDACK application is defined by designing processes, entities, and user interfaces. Processes encapsulate the business logic and entities define the persistent state of the application. The foundation of SDACK is the **ObjectRelations** framework. It allows the generic modeling of relations between objects, similar to the way object orientation defines the modeling of objects. This abstraction of object interdependencies makes many of the framework features possible.

SDACK provides all components that are necessary to write applications for different purposes without the need for almost no additional dependencies. This covers areas like persistence, networking, logging, user interface, and more. User interfaces are defined with the GEWT framework which provides an abstraction of component-based user interface toolkits. For web applications it uses the open source [GWT library](https://www.gwtproject.org) for the actual UI rendering, but this can be replaced or extended with other UI libraries if necessary. Apart from this external dependencies are only needed for an application's runtime environment, e.g. JDBC database drivers.

The main goal of SDACK is to make application development as easy as possible while still providing the full functionality of all underlying APIs. A major design principle of all frameworks is to hide as much non-essential code from the developer as possible. It provides simple to use APIs that need no configuration for default behavior. They also use modern software design patterns like Generics, Functional Programming, and Fluent Interfaces. And the full framework stack is quite compact, in code size as well as considering the number of API methods to learn.

The following sections provide an overview of the single framework layers. They also link to detailed documentation for the respective layer.

## esoco-common

This library contains a small set of essential functionality like core interfaces and fundamental data structures. It is the foundation of the ObjectRelations framework and uses only a small set of standard Java APIs. That allows to use it in constrained Java environments like GWT where only a subset of these APIs is available.

## ObjectRelations

The Object Relations library implements a new development concept. The idea is to enhance object-oriented software development by modeling the relations between objects. The other frameworks make extensive use of generic relations to achieve their purpose because by using relations code becomes typically much more efficient. In addition the ObjectRelations project also contains packages for functional programming which is also used in many of the dependent frameworks. These packages are quite simliar to the new functional programming code introduced in Java 8.

It is recommended to study the ObjectRelations documentation before the other frameworks to get an understanding of the underlying principles.

## esoco-lib

This library contains several packages with generic code that will be helpful for almost any development project. It defines fundamental data structures and interfaces as well as utility functions for several areas like I/O, reflection, string handling and more. It builds upon the **esoco-common** project but is not compatible with the Javascript generation of GWT because it requires a full Java Runtime Environment.

## Storage Framework

The storage framework \(in the project esoco-storage\) provides a simple but powerful abstraction to implement object persistence. It's generic and simple API can be adapted to different persistence concepts like object oriented databases or key-value stores \(aka NoSQL databases\). It already comes with JDBC-based implementation for the common case of SQL databases. It provides direct persistence for arbitrary Java objects \(POJOs\) without the need for configuration beside the actual database connection. It is built on the ObjectRelations framework and the functional programming framework therein. Predicates from the latter define the query criteria in a fluent way.

## Business Framework

The business framework \(esoco-business\) is the foundation for business-related applications. The term _business_ in this context stands for higher-level abstractions, not as a limititation to commercial purposes. This framework provides the means to model persistent data entities and to define processes that modify such data and interact with users or external parties. Processes can display and query data to users through application-specific user interfaces. Entities can also be used to integrate existing databases into a new software project.

## GEWT

Most applications need some kind of user interface to display and collect data. The GEWT framework is a generic user interface abstraction that uses the open source Google Web Toolkit \(GWT\) as the actual user interface implementation. But GEWT has a generic and simple API that can be mapped on other component-based UI Toolkits if necessary. The EWT project does so for desktop applications based on Swing or SWT.

## GWT Web Application Framework

The Web Application Framework \(esoco-gwt\) is the bridge between the business framework and GEWT. It combines the business back-end \(processes and persistent entities\) with a GWT user interface to build client-server web applications. The user interface is generated from process state and input from the user is persisted automatically.

The Webapp framework also contains the base classes and core elements necessary to create an application that can be deployed into an application server. For persistence it uses a connection to a standard relational database like PostgreSQL or MySQL. Such applications already support the automatic historization of changes in persistent data, logging, and background processing. It is only necessary to implement the business-specific entities and processes to create a full application.

