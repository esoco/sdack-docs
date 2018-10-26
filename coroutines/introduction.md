# Introduction to Coroutines

After some experimenting with goroutines and the coroutine library of Kotlin I was looking for something similar in Java. But all solutions I could find are not pure Java by using non-Java libraries via JNI or rely on special tools, like bytecode manipulation by special Java Agents that need to run in parallel with an application.

So the question is, is it possible to implement coroutines in pure Java? To answer that it was necessary to look at the two core properties of coroutines:

1. Coroutines are often called “light-weight” threads, meaning they don’t run on their own dedicated system thread but instead share existing threads cooperatively —hence _co_-routines. _Cooperatively_ means that it is the responsibility of the coroutine developer to ensure that it doesn’t occupy a shared thread more than necessary. Explicit support for shared thread execution had been added to Java 7 with _ForkJoinTask_ and enhanced in Java 8 with _CompletableFuture_.
2. Coroutines may be suspended for indeterminate time, e.g. when waiting for I/O or to await other coroutines. During suspension coroutines don’t occupy any thread at all and resume execution only when the resource waited upon becomes available. Suspension is the part for which languages with coroutine features have special keywords and compiler support and where the frameworks mentioned above modify the bytecode of the executing program. But is it possible in pure Java without such tricks?

The short answer to this question is: Yes. This article presents a pure Java implementation of coroutines, available as [Open Source on GitHub](https://github.com/esoco/coroutines) under the Apache 2.0 license. It uses features available since Java 8 to make the declaration and execution of coroutines as simple as possible. 

The current version is 0.9.0 to indicate that the code is still considered experimental and hasn’t been used in production yet. It should be expected to still have some bugs. Any feedback on the API and functionality are welcome and would help to have a 1.0 release soon. The library has only a single dependency to another of our projects which provides some generic base functionality.

The coroutine execution part of this framework follows the pattern of _structured concurrency_ and has been much inspired by the noteworthy essay [_Notes on structured concurrency, or: Go statement considered harmful_](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) by Nathaniel J. Smith. I highly recommend this to anyone who is involved in the programming of concurrent code and/or wants to use this framework. It has also [influenced the latest iteration of Kotlin coroutines](https://medium.com/@elizarov/structured-concurrency-722d765aa952). As a quick summary, it states that concurrent code that runs on it’s own without any connection to the application code is dangerous and must be avoided. But read it, it’s really worth the time!

#### Declaring Coroutines

As Java lacks native mechanisms to declare suspensions the only way to implement the declaration of suspending coroutines is with an API. Fortunately, the functional programming features introduced with Java 8, most notably lambda expressions and method references, allow a concise declaration of executable code fragments. Together with the option of static imports the presented framework allows a simple declaration of coroutines.

Coroutines are defined as instances of the class `Coroutine`. A new coroutine is created either by invoking the constructor or by calling the static factory method `Coroutine.first()`. The latter is recommended because it is part of the [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface) for the building of coroutines. The complement to it is the instance method `Coroutine.then()` which adds an additional execution step.

These methods expect an instance of the abstract class `CoroutineStep` as their argument. This class needs to be extended to provide new coroutine functionality. Fortunately the framework already comes with several coroutine step implementations that should cover the most common uses. The most generic of these steps is probably `CodeExecution` which does as it’s name says: execute \(arbitrary\) code. It provides a supplemental fluent API based the functional interfaces of Java 8 for the creation of new step. Here’s a simple example of a coroutine declaration:

{% embed url="https://gist.github.com/esocode/af34423b449ecfcaac1842cda972650e" caption="A simple coroutine declaration" %}

This reveals that coroutines are generically typed with an input and and output type, similar to the Java `Function` interface. It means that coroutines can be invoked like such functions by providing an input value to process and receiving a result of the computation. 

The example also shows a slight limitation of nested generic invocations in Java. The input type for the call to _apply_ inside the call to _first_ cannot be determined automatically and therefore makes it necessary to provides the input type explicitly \(by using `(String s)` instead of _s_ only\).

An important property of coroutines is that they are immutable. Once defined the sequence of steps cannot be modified anymore. The instance method _then_ always returns a new coroutine instance that contains the additional step. This allows to create coroutine templates that can be used as building blocks for new coroutines, either as a starting point for added functionality or by using them as subroutines \(described later\). Because of their immutability they may also be declared as static final constants.

Back to the example: the method _apply_ \(a static import from `CodeExecution`\) expects a function argument that can either be provided as a lambda expression like above or as a method reference if an implementation with a matching signature is already available. In fact, in the example above the first call could also be written as `first(apply(String::trim))`. And in this case that would also eliminate the problem with the resolving of the nested generically typed invocations.

CodeExecution contains static factory method like _apply_ for all standard functional interfaces. They are named after the original interface methods: _consume_ \(for `Consumer`\), _supply_ \(`Supplier`\), _run_ \(`Runnable`\). These method are generically typed so that `Coroutine.then()` will only accept code with an input type that matches the current output of the coroutine and creates a new coroutine with the same output type as the code. This allows to chain arbitrary steps into larger coroutines with a defined in- and output.

The framework contains several other coroutine step implementations that provide additional functionality, especially ones that implement suspension which could be seen as the most attractive feature of coroutines. But before we come to this let’s first have a look at how coroutines are executed.

#### Executing Coroutines

In the first iteration of the framework, coroutines where simply started to run in some pooled background thread. But then I came across the [_Notes on structured concurrency_](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) mentioned before and recognized the insight behind it. Just starting some code so that it runs without being supervised by the application is a bad idea. Especially if it consists of a lot of parallel executions which is a typical scenario for coroutines. 

As long as code behaves as expected that may not even be a big deal. But if such code fails due to arbitrary error conditions the problems begin. Such background code often fails silently, in the best case leaving only some trace in log files or similar. Read the essay, it’s reasoning explains this in detail.

**CoroutineScope**

Hence, in the current version of the framework, coroutines cannot be executed in some fire-and-forget manner. Instead, they must always be executed inside a `CoroutineScope`. The main property of a scope is that it will block the invoking thread unless all coroutines that have been started in it have terminated, either successfully or with an error. And if an error occurred in any coroutine the scope will also throw an exception instead of just continuing the current thread. This makes it impossible to lose track of the executions of coroutines and their error conditions.

Furthermore the scope serves as an object that gives access to configuration data to it’s child coroutines. It also provides them with a way to share state if needed. Though that is an advanced topic because it may require synchronized access as the coroutines run on different threads. 

A coroutine scope cannot be instantiated directly. Instead, the class provides static methods that create a new scope and executes some application-provided code that in turn starts coroutines in the scope. The scope then blocks the invoking thread until the code and all coroutines that have been started by it have finished. If an error occurs during an scope execution the call will throw an exception. 

The following example shows a \(very simple\) scope execution:

{% embed url="https://gist.github.com/esocode/b98baef2ab5a8b7b73f8b90a5d42a1ae" caption="Launching a million coroutines" %}

If there were any code after the call to _launch_ \(i.e. after the closing `});`\) it would not run before all coroutines have finished because as explained, the scope will await the termination of all it’s children.

The example runs one million instances of a coroutine in parallel. By default they are executed in the common thread pool of the Java runtime. If you would try to start that number of distinct threads instead there’s a good chance that it will cause an out of memory error. With lower counts threads will work but be much slower than the coroutine solution \(on my machine by a factor of around 20\).

The single-step coroutine in this example has no deeper purpose than to occupy a few processing cycles by calculating the square roots of the values 1 to 10 \(the _Range_ class is a member of the base library the _coroutines_ project depends on\). The coroutine variable is declared with wildcard generic types because the in- and outputs are not needed in this example. It could also be declared as _&lt;Void,Void&gt;_ because that is the actual type of the above declaration.

The _launch_ method performs the scope-execution described above. It’s argument is a functional interface with a method that expects a coroutine scope as it’s argument and returns nothing. Other than the typical Java functional interfaces, this one is allowed to throw arbitrary exceptions. This allows the scope code to leave error handling to the code outside the scope because the scope will re-throw any exceptions that may occur.

In most cases the code for the scope will probably be provided as a lambda expression like in the example. A coroutine can then be run from within by invoking the _runAsync_ method with the scope as the argument and optionally a coroutine input value \(which is not used in the example above\). As the method name says, the coroutine is executed asynchronously in a different thread than the launching code is running on.

**Continuation**

When a coroutine is started an instance of `Continuation` is returned as shown in the example code below. This object has the same generic type as the output type of the associated coroutine and serves two purposes:

1. Provide a shared state for all steps that are executed in a coroutine.
2. Allow the starting code to control the coroutine execution and to access it’s the result after finishing.

Getting the coroutine result from the continuation

All steps that are executed in a coroutine receive the current continuation as their argument. For example, the `CodeExecution` class mentioned before has alternative factory methods that accept binary functions with an additional continuation argument. The step can then use the continuation to get or set state values, query configuration data, or access the scope it is running in.

Continuations are always local to a single execution and can therefore be accessed by steps without synchronization. The steps of a coroutine are always executed sequentially and therefore cannot access the continuation concurrently.

The application code can use the returned continuation object to query the execution state, await the termination of the coroutine, and access the result after a successful termination or process errors that may have occurred. It should not modify any state of the continuation while it runs because that could cause concurrent modification issues.

The easiest way to access the result of a coroutine is to query the _getResult_ method of the continuation as shown in the example above. This method will block the calling thread until the result is available or throw an exception if the coroutine failed or has been cancelled.

To only wait for the coroutine to finish the method _await_ can be used instead. This will neither return a result nor throw an exception on errors. Any error handling must then be performed by the invoking code. If such explicit error handling is done the application must notify the continuation by calling it’s method _errorHandled_. That will clear the error in the scope \(which would otherwise throw an exception on termination\).

The continuation can also be used to cancel the execution if it hasn’t finished yet by invoking it’s _cancel_ method. Cancellation will be automatically checked between step executions but it can also be queried from within a step implementation by querying _isCancelled_ on the current continuation.

**CoroutineContext**

As mentioned before, by default coroutines are executed in the common thread pool of the Java runtime \(`ForkJoinPool.commonPool()`\). But the thread pool and other parameters of the execution are actually provided by the `CoroutineContext`. The purpose of the context is to provide coroutines in one or more scopes with an execution environment. It can also be used to share configuration and synchronized resources between scopes.

The framework has a default context which can be queried and changed through static methods of the global `Coroutines` class. But it is also possible to instantiate and configure contexts only for certain executions if necessary. A non-default context can be provided when launching a new scope which will then use the context for all coroutines it launches. But for many use cases the default context should be sufficient.

**Blocking Execution**

Testing and debugging asynchronous code is often problematic because it is difficult to track the different executions, especially with shared thread pools. To make the development of single coroutines easier the coroutine class also has a _runBlocking_ method which can be used instead of _runAsync_. As the name indicates, this will simply execute the coroutine synchronously on the current thread and block until the coroutine completed. It still needs to be run inside a scope because that provides the runtime environment of coroutines \(state, configuration, context\).

Running coroutines synchronously can be especially useful when creating unit tests of algorithms implemented as coroutines. It can also be used to run certain coroutines either asynchronously or block, depending on certain conditions. Of course, if the purpose of a coroutine is concurrent communication with other coroutines or to wait for external resources running it in blocking mode won’t help without running it in a separate thread.

#### Suspension and Channels

The examples shown so far have only executed some simple code concurrently which could also be achieved with regular threads or thread pools. But as stated at the beginning, an important feature of coroutines is the ability to suspend the execution completely while waiting for some condition. In suspended state a coroutine won’t consume or block any processing resources \(i.e. threads\) and simply lie dormant until the condition is met.

A typical case of suspension is the communication between coroutines through so-called _Channels_ \(not to be confused with the `java.nio.Channel` Interface\). Channels are queue-like data structures with a fixed capacity that data can be written to or read from. A coroutine that tries to read from an empty channel or write to a full channel will be suspended until data or capacity becomes availability. 

The coroutines framework contains support for suspending steps in general and also for channels in particular. Channels are identified by a `ChannelId` which has the main purpose of defining the channel data type with it’s generic type parameter. The following code shows a simple example of channel communication. It again uses static imports for all key elements for a concise notation.Suspending coroutine communication through a channel

This code first defines a channel ID and two coroutines, one for sending a string into the channel and one to receive data from it. It achieves that by adding the coroutine steps `ChannelSend` and `ChannelReceive` \(through their static factory methods\) with the channel ID as their configuration parameter.

If a channel doesn’t exist when it is accessed for the first time it will be created automatically with a capacity of 1. If a higher capacity is needed the channel needs to be created in advance with _createChannel_, e.g. by the scope code before launching the first coroutine.

The coroutines are then executed in a launched scope. The call of `Threads.sleep(1000)` in this example only serves to demonstrate the suspension of the receiving coroutine \(_Threads_ is a class from the base library and in this case skips the otherwise necessary exception handling\). When the _receive_ coroutine is started the channel with the ID _testChannel_ will be created automatically with a capacity of 1 and initially empty. Therefore the coroutine is suspended until data becomes available in the channel.

The code then sleeps 1000 milliseconds and subsequently starts the _send_ coroutine with an input value. As capacity is available in the channel the send will succeed immediately \(otherwise _send_ would be suspended\). And now data has become available, so the _receive_ coroutine will be resumed and process that data. Finally the result of the processing can be retrieved from the continuation.

By default channels are created in the coroutine scope. But it is also possible to create channels in the context, thereby allowing coroutines in different contexts to communicate through channels. To do so the channel must be created in advance by calling _createChannel_ on the context with the desired channel ID and capacity.

Channels are a very powerful mechanism for communication between coroutines. For example, simple Boolean channels with the default capacity of 1 can be used as semaphore to exchange signals between coroutines. And larger channels with a complex datatype may be used to transfer data between different processing stages that run concurrently. In neither case is it necessary to perform complex synchronization like in traditional concurrent code. Sending to or receiving from a channel within a coroutine is sufficient to achieve the synchronization while also only consuming as few resources as required.

#### Selection

Selection is a coroutine concept that is often associated with \(and sometimes constrained to\) receiving from channels. Languages with coroutine support typically have a _select_ statement for this purpose. The statement suspends on multiple channels to receive from and resumes as soon as data becomes available in one of the channels.

In this framework selection is implemented as a suspending coroutine step. It extends the concept by allowing to select from an arbitrary number of coroutines, not only from channels. The example below shows a \(contrived\) example where the select consists of one channel receive and two hypothetical coroutines that also suspend until data is available. If executed the _selectFirst_ coroutine would resume execution as soon as one of the coroutines selected from you resume.

Normally selection will be performed on a set of suspending coroutines. If the set would contain a non-suspending coroutine the select would never suspend because that coroutine would not suspend and therefore be selected immediately. But this property can also be used to add default behavior to a selecting coroutine: select a suspending value or, if not available, use the default. In such a case it would typically be necessary to have some surrounding control structure, e.g. a loop to repeat the selection. Another way would be to precede the default path with a suspending delay, thus converting it to a suspending coroutine itself. See below for information about such additional steps.

**Collection**

The framework also extends the selection concept to collection. If you replace _select_ with _collect_ and _or_ with _and_ in the example above, you get the equivalent code for the collection of multiple coroutines. While selection continues if the first value becomes available, collection will only resume when all values have become available. Consequentially, whereas selection returns a single value, collection returns a collection of all values for further processing by subsequent steps.

#### NIO

An obvious use case for suspension is waiting for I/O. The _coroutines_ library contains a couple of suspending steps that are built upon the asynchronous API of the Java New IO package. They allow to send and receive to and from sockets, to read or write files, and to listen on server socket. Their usage is analog to the channel steps. These steps can also serve as examples for how to create new suspending steps based on existing asynchronous APIs.

#### Subroutines, Conditions, Loops, and More

Because coroutines need to be declared through the coroutine API, existing Java constructs like conditional expressions and loops cannot be used for suspensions. Instead, the framework provides special coroutine steps for these \(and some other\) purposes. Besides, it is always easily possible to subclass `CoroutineStep` to create new steps if the existing implementations don’t suffice.

**Subroutines**

Any existing coroutine can be invoked as a subroutine of another coroutine, by using the step `CallSubroutine`. This allows to compose new coroutines from existing ones and therefore to build libraries of coroutine functionalities as a base for such compositions.

**Conditional Execution**

Sometimes it may be necessary to execute different steps \(including subroutines\) based on the result of a certain condition. This can be achieved with the step `Condition` which provides different options to make the execution of steps depend on the evaluation of a predicate, as shown in this example.Conditional coroutine execution

**Loops**

Instead of looping inside the lambda expression that is executed by a step it can make more sense to repeat the execution of a certain step or subroutine based on a Boolean condition. This permits the framework to suspend on each loop cycle if necessary.Looping in a coroutine

**Iteration**

For the common case of looping over a collection or another object that implements the `Iterable` interface the framework provides the `Iteration` step. It applies another coroutine step to each element of a collection and can optionally collect the results into a new collection.

**Delayed Execution**

When using dedicated threads an application can wait for a certain amount of time by calling `Thread.sleep`. But that method must be avoided when working with thread pools because letting a pooled thread sleep will block it for any other use. Furthermore, with coroutines it is desirable to suspend the execution while a coroutine is delayed.

Therefore the framework provides the special step `Delay` that has a builder method _sleep_ that suspends a coroutine for a certain time. This can be used as any other step and the implementation behind it should be sufficient for most use cases. But it’s inner workings are a bit different than that of other steps and may need to be considered under special circumstances, e.g. if a lot of delayed executions are performed.

The reason is that delayed executions are not performed in the standard thread pool but in an instance of a `ScheduledExecutorService` that is queried from the context with it’s _getScheduler_ method. By default that is initialized with a scheduled thread pool with a size of 1. So, if a lot of scheduling should take place it may be adequate to configure a different scheduler. Furthermore, the scheduler may compete with the standard thread pool which typically uses all available system threads. In such case alternative thread pool configurations can be set in the coroutine context, either in the default context or a dedicated one for the affected execution\(s\).

#### ObjectRelations

As mentioned the coroutines project has one dependency, and that is to the _ObjectRelations_ project. Apart from some base functionality \(like the _Range_ and _Threads_ classes used in the previous examples\) this library implements a new development concept for the generic modeling of the relations between objects in object-oriented programming. Very curtly, you can think of relations as typed and intelligent object references that can be set on arbitrary objects. And which can have relations themselves. A detailed introduction can be found [on the documentation site](https://esoco.gitbook.io/sdack/frameworks/objectrelations).

This base is important because it adds some generic functionality to coroutines that make the framework even more versatile. Each class in the framework implements the _Relatable_ interface which allows them to have arbitrary relations. And these relations can then be used to inject configuration to the framework objects or to carry state through a coroutine execution by setting relations on the continuation. Such state can consist of an arbitrary number of \(type-safe\) relations which would be difficult to achieve otherwise.

One application of relations can be found in the `Coroutines` class which contains global definitions. Among those are several event listener relation types. These listeners can then be set on either coroutines, continuations, scopes, or contexts and will be notified by the framework when the associated events happen. Such listeners can be very helpful for debugging or the management of coroutine executions.

Another possible use of relations can be found in the coroutine scope. Sometimes it may be desirable to have a scope that produces a result when all it’s coroutines have finished. This can be achieved by starting the scope with the _produce_ method instead of _launch_. While the latter has a void result and blocks, _produce_ returns directly with an instance of Java’s _Future_ interface. This instance can then be used to await the scope’s termination and query it’s result. 

To determine the result, _produce_ receives a function that will be applied to the finished scope to retrieve the result. And because relation types \(which define relations\) are also functions a relation type can be directly used to access the result of a scope. Let’s assume _someCoroutine_ produces a text result and stores it in the relation `TEXT` of the scope. Then the following code would initiate the asynchronous production and create a future with access to that result:Producing a value from a coroutine scope

Another generic application of relations is the automatic closing of resource that have been opened by coroutine steps. Any relation in a continuation or scope that refers to a closable object and has the meta-relation `MANAGED` set on it will be automatically closed when the coroutine or scope finishes in any state. This behavior is similar to Java’s try-with-resources blocks and is used by the NIO steps mentioned above to automatically close the socket or file channels they operate on.

#### Conclusion

This article hopefully provides an appropriate introduction to the coroutines framework and it’s basic use. Please be reminded that it’s a first release, so it may still contain some bugs and inconsistencies. Any feedback as well as questions are welcome, as are contributions to the project.

