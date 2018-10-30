# Coroutines in pure Java

This article presents a pure Java implementation of coroutines, available as [Open Source on GitHub](https://github.com/esoco/coroutines) under the Apache 2.0 license. It uses features available since Java 8 to make the declaration and execution of coroutines as simple as possible. The library has only a single dependency to our [ObjectRelations framework](https://github.com/esoco/objectrelations) which provides generic base functionality.

The coroutine execution part of this framework follows the pattern of _structured concurrency_ and has been much inspired by the noteworthy essay [_Notes on structured concurrency, or: Go statement considered harmful_](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) by Nathaniel J. Smith. I highly recommend this to anyone who is involved in the programming of concurrent code and/or wants to use this framework. It has also [influenced the latest iteration of Kotlin coroutines](https://medium.com/@elizarov/structured-concurrency-722d765aa952). As a quick summary, it states that concurrent code that runs on it’s own without any connection to the application code is dangerous and must be avoided.

#### Declaring Coroutines

As Java lacks native mechanisms to declare suspensions the only way to implement suspending coroutines is with an API. Fortunately, the functional programming features introduced with Java 8, most notably lambda expressions and method references, allow a concise declaration of executable code fragments. Together with the option of static imports the presented framework allows a simple declaration of coroutines.

Coroutines are defined as instances of the class `Coroutine`. A new coroutine is created either by invoking the constructor or by calling the static factory method `Coroutine.first()`. The latter is recommended because it is part of the [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface) for the declaration of coroutines. It's complement is the instance method `Coroutine.then()` which extends a coroutine with additional functionality.

These methods expect an instance of the abstract class `CoroutineStep` as their argument. The framework already contains several step implementations that cover the most common uses. The most generic step steps is `CodeExecution`. As all framework steps it provides static factory methods that supplement the fluent coroutine API. By invoking these methods with lambda expressions or method references a coroutine can be extended with arbitrary code. Here’s a simple example of such a declaration:

{% embed url="https://gist.github.com/esocode/af34423b449ecfcaac1842cda972650e" caption="A simple coroutine declaration" %}

This reveals that coroutines are generically typed with an input and and output type, similar to the Java `Function` interface. Hence coroutines can be invoked with a specific input value to process and may return the result of a computation.

An important trait of coroutines is that they are immutable. Once defined the sequence of steps cannot be modified anymore. The instance method _then_ always returns a new coroutine instance that contains the additional step. This allows to create coroutine templates that can be used as building blocks for new coroutines, either as a starting point for added functionality or by using them as subroutines \(described later\). Because of their immutability they can also be used in constant declarations.

`CodeExecution` contains factory methods for the standard functional interfaces of Java. These method are generically typed so that `Coroutine.then()` will only accept code with an input type that matches the current output of the coroutine and creates a new coroutine with the same output type as the code. This allows to chain arbitrary steps into larger coroutines with a defined in- and output.

A coroutine can have an arbitrary number of steps. The more steps there are, the more suspension points a coroutine has. Each of these points shortly interrupts the execution, giving other concurrent code a chance to run. On the other hand each suspension has some management overhead. It needs to be determined which granularity is the right one for a particular solution.

The framework includes several other coroutine steps that provide additional functionality, especially steps that implement coroutine suspension. But before we look into this let’s first see how coroutines are executed.

#### Executing Coroutines

In the first iteration of the framework, coroutines where simply started to run in some pooled background thread. But after reading the [_Notes on structured concurrency_](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) the code was changed to follow that pattern and always execute coroutines in a well-defined environment.

As long as code behaves as expected executing it unsupervised in the background may not be a big deal. But if such code fails due to arbitrary error conditions the problems begin. Such background code often fails silently, in the best case leaving only some trace in log files or similar.

**CoroutineScope**

The environment in which all coroutines need to be run is `CoroutineScope`. The main property of a scope is that it will block the invoking thread unless all coroutines that have been started in it have terminated, either successfully, by cancellation, or with an error. And if an error occurred in any coroutine the scope will also throw an exception instead of just continuing the current thread. This prevents losing track of coroutine executions and their error conditions.

Furthermore, the scope gives it’s coroutines access to configuration data and provides them with a common place to share state through. Though the latter is an advanced topic because it may require synchronized access as different coroutines will normally run on different threads.

A coroutine scope cannot be instantiated directly. Instead, it provides static methods which create a new scope and execute some application-provided code that in turn starts coroutines in the scope. The scope then blocks the invoking thread until the code and all coroutines that have been started by it have finished. If an error occurs during the executions the scope will throw an exception. 

The following example shows a \(very simple\) scope execution:

{% embed url="https://gist.github.com/esocode/b98baef2ab5a8b7b73f8b90a5d42a1ae" caption="Launching a million coroutines" %}

If there were any code after the call to _launch_ \(i.e. after the closing `});`\) it would not run before all coroutines have finished because as explained, the scope will await the termination of all it’s children.

The example runs one million instances of a coroutine in parallel. By default they are executed in the common thread pool of the Java runtime. If you would try to start that number of distinct threads instead there’s a good chance that it will cause an out of memory error. With lower counts threads will work but be much slower than the coroutine solution \(on my machine by a factor of around 20\).

The single-step coroutine in this example has no deeper purpose than to occupy a few processing cycles by calculating the square roots of the values 1 to 10 \(the _Range_ class is a member of the base library the _coroutines_ project depends on\). The coroutine variable is declared with wildcard generic types because the in- and outputs are not needed in this example. It could also be declared as _&lt;Void,Void&gt;_ because that is the actual type of the above declaration.

The _launch_ method performs the scope-execution. It’s argument is a functional interface with a method that expects a coroutine scope as it’s argument and returns nothing. Other than the typical Java functional interfaces, this one is allowed to throw arbitrary exceptions. This allows the application’s scope code to leave error handling to the code outside the scope because the scope will re-throw any exceptions that may occur.

In most cases the code for the scope will probably be provided as a lambda expression like in the example. Coroutines can then be run from within by invoking their _runAsync_ method with the scope as the argument and optionally an input value \(which is not used in the example above\). As the method name says, the coroutine is executed asynchronously in a different thread than the launching scope code is running on.

Sometimes it may be desirable to have a scope that produces a result when all it’s coroutines have finished. This can be achieved by starting the scope with the _produce_ method instead of _launch_. While the latter has a void result and blocks, _produce_ returns directly with an instance of Java’s _Future_ interface. This instance can then be used to await the scope’s termination and query it’s result. 

To retrieve the result from the scope a corresponding function must be provided as the first argument to the _produce_ factory method. In the following example this function is called _`TEXT`_ which  is actually a relation type from the [ObjectRelations framework](../frameworks/objectrelations.md). The usage of relations in coroutines is explained at the end of this article.

{% embed url="https://gist.github.com/esocode/2e7b8ee6ec9759c62a90a1852b622db7" caption="Producing a value from a coroutine scope" %}



**Continuation**

When a coroutine is started, an instance of `Continuation` is returned as shown in the example code below. The continuation has the same generic type as the output type of the associated coroutine and serves two purposes:

1. Provide a shared state for all steps that are executed in a coroutine.
2. Allow the starting code to control the coroutine execution and to access it’s the result after finishing.

{% embed url="https://gist.github.com/esocode/c6da3c83574980c27ab7fbabf5fe38e6" caption="Getting the coroutine result from a continuation" %}

All steps that are executed in a coroutine receive the current continuation as their argument. For example, the `CodeExecution` class mentioned before has alternative factory methods that accept binary functions with an additional continuation argument. The step can then use the continuation to get or set state values, query configuration data, or access the scope it is running in.

Continuations are always local to a single execution and can therefore be accessed by steps without synchronization. The steps of a coroutine are always executed sequentially and therefore cannot access the continuation concurrently.

The application code can use the returned continuation object to query the execution state of the coroutine, await it’s termination, or cancel it. The continuation also provides access to the result after a successful termination or to errors that may have occurred. The starting code should not modify any state of the continuation while the coroutine is still running because that could cause concurrent modification issues.

The easiest way to access the result of a coroutine is to query the _getResult_ method of the continuation as shown in the example above. This method will block the calling thread until the result is available or throw an exception if the coroutine failed or has been cancelled.

To wait for the coroutine to finish the method _await_ can be used instead. This will neither return a result nor throw an exception on errors. Any error handling must then be performed by the invoking code. If such explicit error handling is done the application must notify the continuation by calling it’s method _errorHandled_. That will clear the error in the scope, which would otherwise throw an exception on termination.

The continuation can also be used to cancel the execution if it hasn’t finished yet by invoking it’s _cancel_ method. Cancellation will be automatically checked between step executions but it can also be queried from within a step implementation by querying _isCancelled_ on the current continuation.

**CoroutineContext**

As mentioned before, by default coroutines are executed in the common thread pool of the Java runtime \(`ForkJoinPool.commonPool()`\). But the thread pool and other parameters of the execution are actually provided by the `CoroutineContext`. The purpose of the context is to provide coroutines in one or more scopes with an execution environment. It can also be used to share configuration and synchronized resources between scopes.

The framework has a default context which can be queried and changed through static methods of the global `Coroutines` class. But it is also possible to instantiate and configure contexts only for certain executions if necessary. A non-default context can be provided when launching a new scope which will then use the context for all coroutines it launches. But for many use cases the default context should be sufficient.

**Blocking Execution**

Testing and debugging asynchronous code is often problematic because it is difficult to track the different executions, especially with shared thread pools. To make the development of single coroutines easier the coroutine class also has a _runBlocking_ method which can be used instead of _runAsync_. As the name indicates, this will simply execute the coroutine synchronously on the current thread and block until the coroutine completed. It still needs to be run inside a scope because that provides the runtime environment of coroutines \(state, configuration, context\).

Running coroutines synchronously can be especially useful when creating unit tests of algorithms implemented as coroutines. It can also be used to run certain coroutines either asynchronously or blocking, depending on certain conditions.  Of course, if the purpose of a coroutine is concurrent communication with other coroutines or other types of suspensions, running it in blocking mode would only work with running it in a separate thread.

#### Suspension and Channels

The examples shown so far have only executed some simple code concurrently which could also be achieved with regular threads. But an important feature of coroutines is the ability to suspend their execution completely while waiting for some condition. In suspended state a coroutine won’t consume or block any processing resources \(like threads\) and simply lie dormant until the condition is met.

A typical case of suspension is the communication between coroutines through so-called _Channels_ \(not to be confused with the `java.nio.Channel` Interface\). Channels are queue-like data structures with a fixed capacity that data can be written to or read from. A coroutine that tries to read from an empty channel or write to a full channel will be suspended until data or capacity becomes availability. 

The coroutines framework contains support for suspending steps in general and also for channels in particular. Channels are identified by a `ChannelId` which has the main purpose of defining the channel data type with it’s generic type parameter. The following code shows an example of channel communication. It again uses static imports for all key elements for a concise notation.

{% embed url="https://gist.github.com/esocode/9cf4c32e9d0407b8dbd8f9fafb5f54b8" caption="Suspending coroutine communication through a channel" %}

This code first defines a channel ID and two coroutines, one for sending a string into the channel and one to receive data from it. It achieves that by adding the coroutine steps `ChannelSend` and `ChannelReceive` \(through their static factory methods\) with the channel ID as their configuration parameter.

If a channel doesn’t exist when it is accessed for the first time it will be created automatically with a capacity of 1. If a higher capacity is needed the channel needs to be created in advance with _createChannel_, either in the scope or the context..

The coroutines are then executed in a launched scope. The call to `Threads.sleep(1000)` in this example only serves to demonstrate the suspension of the receiving coroutine \(_Threads_ is a class from the dependency that also performs exception handling\). When the _receive_ coroutine is started the channel with the ID _testChannel_ will be created automatically with a capacity of 1 and is initially empty. Therefore the coroutine is suspended until data becomes available in the channel.

The code then sleeps 1000 milliseconds before starting the _send_ coroutine. Since capacity is available in the channel the send will succeed immediately \(otherwise _send_ would be suspended\). And now data has become available in the channel, so the _receive_ coroutine will be resumed and process that data. Finally the result of the processing can be retrieved from the continuation.

By default channels are created in the coroutine scope. Alternatively channels can be created in the context, thereby allowing coroutines in different scopes to communicate through channels. In that case the channel must be created in advance by calling _createChannel_ on the context with the desired channel ID and capacity.

Channels are a very powerful mechanism for communication between coroutines. For example, simple Boolean channels with the default capacity of 1 can be used as semaphore to exchange signals between coroutines. And larger channels with a complex datatype may be used to transfer data between different processing stages that run concurrently. In neither case is it necessary to perform complex synchronization like in traditional concurrent code. Sending to or receiving from a channel within a coroutine is sufficient to achieve synchronization.

#### Selection

Selection is a coroutine concept that is often associated with \(and sometimes constrained to\) receiving from channels. Languages with coroutine support typically have a _select_ statement for this purpose. The statement suspends on multiple channels to receive from and resumes as soon as data becomes available in one of the channels.

In this framework selection is implemented as a suspending coroutine step. It extends the concept by allowing to select from an arbitrary number of coroutines, not only from channels. The example below shows a \(contrived\) example where the select consists of one channel receive and two hypothetical coroutines that also suspend until data is available. If executed, the _selectFirst_ coroutine would resume execution as soon as one of the coroutines selected from resumes. The other coroutines will then be cancelled.

{% embed url="https://gist.github.com/esocode/6cd7bbfda15d24a3698df1254aa032bb" caption="Selection from multiple coroutines" %}

Normally, selection will be performed on a set of suspending coroutines. If the set would contain a non-suspending coroutine the select would never suspend because that coroutine would be selected immediately. But this trait can also be used to add default behavior to a selecting coroutine: select a suspending value or, if not available, use the default. In such a case it would typically be necessary to have some surrounding control structure, e.g. a loop to repeat the selection until a suspension resumes. Another option would be to precede the default path with a suspending delay, thereby converting it to a suspending coroutine itself.

**Collection**

The framework also extends the selection concept to collection. If you replace _select_ with _collect_ and _or_ with _and_ in the example above, you get the equivalent code for the collection of multiple coroutines. While selection continues if the first suspension becomes available, collection will only resume when all suspensions have resumed. Consequentially, whereas selection continues with a single value, collection produces a collection of all values for further processing by subsequent steps.

#### NIO

An obvious use case for suspension is waiting for I/O. The _coroutines_ library contains a couple of suspending steps that are built upon the asynchronous APIs of the Java New IO package. They allow to send and receive to and from sockets, to read or write files, and to listen on server sockets. Their usage is analog to the channel steps. These steps can also serve as examples for how to create new suspending steps based on existing asynchronous APIs.

#### Subroutines, Conditions, Loops, and more

Because coroutines need to be declared through the coroutine API, existing Java constructs like conditional and loop expressions cannot be used in a suspending way. The framework provides special coroutine steps for these and some other common purposes. Besides, it is always easily possible to subclass `CoroutineStep` to create new steps if the existing implementations don’t suffice.

**Subroutines**

Any existing coroutine can be invoked as a subroutine of another coroutine, by using the step `CallSubroutine`. This allows to compose new coroutines from existing ones and therefore to build libraries of coroutine functionalities as a base for such compositions. The static factory method is _call_.

**Conditional Execution**

Sometimes it may be necessary to execute different steps \(including subroutines\) based on the result of a certain condition. This can be achieved with the step `Condition` which provides multiple variants to execute different steps, depending on the evaluation of a predicate:

{% embed url="https://gist.github.com/esocode/88e9563e3fc5442975b8b4b597483543" caption="Conditional coroutine execution" %}

**Loops**

Instead of looping in the code that is executed by a step it may be a better choice to repeat the execution of a certain step or subroutine based on a Boolean condition, because this allows the framework to suspend on each loop cycle:

{% embed url="https://gist.github.com/esocode/c914dba21158af8807e94aaf7912292b" caption="Looping in a coroutine" %}

**Iteration**

For the common case of looping over a collection or another object that implements the `java.lang.Iterable` interface the `Iteration` step can be used. It applies another coroutine step to each element of a collection and can optionally collect the results into a new collection. As with loops the coroutine is suspended between each iteration round. The following example revisits the first coroutine execution example and uses the iteration step to process each element in a range:

{% embed url="https://gist.github.com/esocode/c5ed805d61ef796473ba878973d18eb5" caption="Iteration in a coroutine" %}

If you ****would profile this example you may notice that it is slower than the non-suspending variant shown earlier. This is because the suspension and re-launch of the code has some overhead which becomes visible in such simple examples.

**Delayed Execution**

When using dedicated threads an application can wait for a certain amount of time by calling `Thread.sleep`. But that method must be avoided when working with thread pools because letting a pooled thread sleep will block it for any other use. Furthermore, with coroutines it is desirable to suspend the execution while a coroutine is delayed.

Therefore the framework provides the special step `Delay` which has several factory methods that suspend a coroutine for a certain time. This can be used as any other step and the implementation behind it should be sufficient for most use cases. But it’s inner workings are a bit different than that of other steps and may need to be considered under special circumstances, e.g. if a lot of delayed executions are performed.

The reason is that delayed executions are not performed in the standard thread pool but in an instance of a `ScheduledExecutorService` which is queried from the context through the _getScheduler_ method. By default it is initialized with a scheduled thread pool with a size of 1. So, if a lot of scheduling should take place it may be adequate to configure a different scheduler. Furthermore, the scheduler may compete with the standard thread pool which typically uses all available system threads. In such case alternative thread pool configurations can be set in the coroutine context, either in the default context or a dedicated one for the affected execution\(s\).

#### ObjectRelations

As mentioned, the coroutines project has one dependency, and that is to the [ObjectRelations](../frameworks/objectrelations.md) project. Apart from some base functionality \(like the _Range_ and _Threads_ classes used in the previous examples\) this library implements a new development concept. This concept allows the generic modeling of the relations between objects in object-oriented programming. Very shortly, you can think of relations as typed and intelligent object references that can be set on arbitrary objects. And they can have relations themselves, providing an easy and generic way to apply meta-data.

This base is important because it adds some generic functionality to coroutines that make the framework even more versatile. Each class in the framework implements the _Relatable_ interface which allows them to have arbitrary relations. And these relations can then be used to inject configuration to the framework objects or to carry state through a coroutine execution by setting relations on the continuation. Such state can consist of an arbitrary number of \(type-safe\) relations which would be difficult to achieve otherwise.

One application of relations can be found in the `Coroutines` class which contains global definitions. Among those are several event listener relation types. These listeners can then be set on either coroutines, continuations, scopes, or contexts and will be notified by the framework when the associated events happen. Such listeners can be very helpful for debugging or the management of coroutine executions.

Another generic application of relations is the automatic closing of resource that have been opened by coroutine steps. Any relation in a continuation or scope that refers to a closable object and has the meta-relation `MANAGED` set on it will be automatically closed when the coroutine or scope finishes in any state. This behavior is similar to Java’s try-with-resources blocks and is used by the NIO steps mentioned above to automatically close the socket or file channels they operate on.

