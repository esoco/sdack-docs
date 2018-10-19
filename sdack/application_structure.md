# Application Structure

This chapter describes the basic structure of a SDACK web application. As any typical web application it is built with a client-server archictecture. That means it has a server part that executes the process \(or business\) logic and a client part that consists of the user interface, running in a web browser. The good news is that by using SDACK a developer can concentrate on the server code. Apart from some bootstrap code the user interface is mostly generated automatically by the framework from process parameters.

## Package Structure

The recommended package structure is similar to a typical GWT application. It consists of the following packages:

* `client`: Contains the code for the client part of the application, i.e. mainly user interface code. Because the foundation of the web application framework is GWT the client code will be translated from Java into JavaScript code. Therefore it must comply with the constraints of the JRE emulation of GWT. Please see the [GWT documentation](http://gwtproject.org) for details.
* `server`: The server package \(and it's sub-packages\) comprises the code that runs on the application server. It can contain arbitrary Java code that may use any kind of external libraries. This package contains the main part of a SDACK application.
* `shared`: The client and server need to communicate with each other and exchange data. The `shared` package of an application \(and of framework projects\) contains the code and data structures for this purpose. Because it is also used on the client it must conform to the same restrictions as the `client` package.

## The Application

The execution of the application always begins on the client when a user visits the start URL. The application code needs to be implemented in a subclass of `GwtApplication`.

