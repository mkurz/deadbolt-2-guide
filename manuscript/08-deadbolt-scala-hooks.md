# Using Deadbolt 2 with Play 2 Scala projects

Deadbolt for Scala provides an idiomatic API for dealing with Scala controllers and templates rendered from Scala controllers in Play applications.

## The Deadbolt Handler
For any module - or framework - to be useable, it must provide a mechanism by which it can be hooked into your application.  For Deadbolt, the central hook is the `be.objectify.deadbolt.scala.DeadboltHandler` trait.  The four functions defined by this trait are crucial to Deadbolt - for example, `DeadboltHandler#getSubject` gets the current user (or subject, to use the correct security terminology), whereas `DeadboltHandler#onAccessFailure` is used to generate a response when authorization fails.

DeadboltHandler implementations should be stateless.

For each method, a `Future` is returned.  If the future may complete to have an empty value, e.g. calling `getSubject` when no subject is present, the return type is `Future[Option]`.

Despite the use of the definite article in the section title, you can have as many Deadbolt handlers in your app as you wish.

### Performing pre-constraint tests
Before a constraint is applied, the `Future[Option[Result]] beforeAuthCheck[A](request: Request[A])` function of the current handler is invoked.  If the resulting future completes `Some`, the target action is not invoked and instead the result of `beforeAuthCheck` is used for the HTTP response; if the resulting future completes to `None` the action is invoked with the Deadbolt constraint applied to it.

### Obtaining the subject
To get the current subject, the `Future[Option[Subject]] getSubject[A](request: Request[A])` function is invoked.  Returning a `None` indicates there is no subject present - this is a valid scenario.

### Dealing with authorization failure
When authorization fails, the `Future[Result] onAccessFailure[A](request: Request[A])` function is used to obtain a result for the HTTP response.  The result required from the `Future` returned from this method is a regular `play.api.mvc.Result`, so it can be anything you chose.  You might want to return a 403 forbidden, redirect to a location accessible to everyone, etc.

### Dealing with dynamic constraints
Dynamic constraints, which are `Dynamic` and `Pattern.CUSTOM` constraints, are dealt with by implementations of `DynamicResourceHandler`; this will be explored in a later chapter.  For now, it's enough to say `Future[Optional[DynamicResourceHandler]] getDynamicResourceHandler[A](request: Request[A])` is invoked when a dynamic constraint it used.

## Expose your DeadboltHandlers with a HandlerCache
Deadbolt uses dependency injection to expose handlers in a type-safe manner.  Various components of Deadbolt, which will be explored in later chapters, require an instance of `be.objectify.deadbolt.scala.cache.HandlerCache` - however, no such implementations are provided.

Instead, you need to implement your own version.  This trait extends `Function[HandlerKey, DeadboltHandler]` and `Function0[DeadboltHandler]` and uses them as follows

* `handlers()` will express the default handler
* handlers(key: HandlerKey) will express a named handler

Here's one possible implementation, using hard-coded handlers.

{title="An example handler cache", lang=scala}
~~~~~~~
@Singleton
class MyHandlerCache extends HandlerCache {
    val defaultHandler: DeadboltHandler = new MyDeadboltHandler

    // HandlerKeys is an user-defined object, containing instances 
    // of a case class that extends HandlerKey
    val handlers: Map[Any, DeadboltHandler] = 
        Map(HandlerKeys.defaultHandler -> defaultHandler,
            HandlerKeys.altHandler -> 
                  new MyDeadboltHandler(Some(MyAlternativeDynamicResourceHandler)),
            HandlerKeys.userlessHandler -> new MyUserlessDeadboltHandler)

    // Get the default handler.
    override def apply(): DeadboltHandler = defaultHandler

    // Get a named handler
    override def apply(handlerKey: HandlerKey): DeadboltHandler = handlers(handlerKey)
}
~~~~~~~

Finally, create a small module which binds your implementation.

{title="Binding the handler cache", lang=scala}
~~~~~~~
package com.example.modules

import be.objectify.deadbolt.scala.cache.HandlerCache
import play.api.inject.{Binding, Module}
import play.api.{Configuration, Environment}
import com.example.security.MyHandlerCache

class CustomDeadboltHook extends Module {
    override def bindings(environment: Environment, 
                          configuration: Configuration): Seq[Binding[_]] = Seq(
        bind[HandlerCache].to[MyHandlerCache]
    )
}
~~~~~~~

## application.conf

### Declare the necessary modules
Both `be.objectify.deadbolt.scala.DeadboltModule` and your custom bindings module must be declared in the configuration.

{title="Enable your module", lang=javascript}
~~~~~~~
play {
  modules {
    enabled += be.objectify.deadbolt.scala.DeadboltModule
    enabled += com.example.modules.CustomDeadboltHook
  }
}
~~~~~~~


## Using compile-time dependency injection

If you prefer to wire everything together with compile-time dependency, you don't need to create a custom module or add `DeadboltModule` to `play.modules`. 

Instead, dependencies are handled using a custom `ApplicationLoader`.  To make things easier, various Deadbolt components are made available via the `be.objectify.deadbolt.scala.DeadboltComponents` trait.  You will still need to provide a couple of things, such as your `HandlerCache` implementation, and you'll then have access to all the usual pieces of Deadbolt.  

{title="An example ApplicationLoader for compile-time DI", lang=scala}
~~~~~~~
class CompileTimeDiApplicationLoader extends ApplicationLoader  {
  override def load(context: Context): Application 
             = new ApplicationComponents(context).application
}

class ApplicationComponents(context: Context) 
                            extends BuiltInComponentsFromContext(context) 
                            with DeadboltComponents
                            with EhCacheComponents {

  // Define a pattern cache implementation
  // defaultCacheApi is a component from EhCacheComponents
  override lazy val patternCache: PatternCache = new DefaultPatternCache(defaultCacheApi)

  // Declare something required by MyHandlerCache
  lazy val subjectDao: SubjectDao = new TestSubjectDao

  // Specify the DeadboltHandler implementation to use
  override lazy val handlers: HandlerCache = new MyHandlerCache(subjectDao) 

  // everything from here down is application-level
  // configuration, unrelated to Deadbolt, such as controllers, routers, etc
  // ...
}
~~~~~~~

The components provided by Deadbolt are

* `scalaAnalyzer` - constraint logic
* `deadboltActions` - for composing actions
* `actionBuilders` - for building actions
* `viewSupport` - for template constraints
* `patternCache` - for caching regular expressions.  You need to define this yourself in the application loader, but as in the example above it's easy to use the default implementation
* `handlers` - the implementation of `HandlerCache` that you provide
* `configuration` - the application configuration
* `ecContextProvider` - the execution context for concurrent operations.  Defaults to `scala.concurrent.ExecutionContext.global`
* `templateFailureListenerProvider` - for listening to Deadbolt-related errors that occur when rendering templates.  Defaults to a no-operation implementation

Once you've defined your `ApplicationLoader`, you need to add it to your `application.conf`.

{title="Enable your ApplicationLoader", lang=javascript}
~~~~~~~
play {
  application {
    loader=com.example.myapp.CompileTimeDiApplicationLoader
  }
}
~~~~~~~

## Tweaking Deadbolt
Deadbolt Scala-specific configuration lives in the `deadbolt.scala` namespace.


There is one setting, `deadbolt.scala.view-timeout`, which is millisecond timeout applied to blocking calls when rendering templates.  This defaults to 1000ms.


Personally, I prefer the HOCON (Human-Optimized Config Object Notation) syntax supported by Play, so I would recommend the following:

{title="Example configuration", lang=javascript}
~~~~~~~
deadbolt {
  scala {
    view-timeout=500
  }
}
~~~~~~~

### Execution context

By default, all futures are executed in the scala.concurrent.ExecutionContext.global context. If you want to provide a separate execution context, you can plug it into Deadbolt by implementing the DeadboltExecutionContextProvider trait.

{title="Providing a custom execution context", lang=javascript}
~~~~~~~
import be.objectify.deadbolt.scala.DeadboltExecutionContextProvider

class CustomDeadboltExecutionContextProvider extends DeadboltExecutionContextProvider {
    override def get(): ExecutionContext = ???
}
~~~~~~~

**NB:** This provider is invoked twice, once in `DeadboltActions` and once in `ViewSupport`.  Make sure you take this into account when you implement the `get()` function.

Once you've implemented the provider, you need to declare it in your custom module (see `CustomDeadboltHook` above for further information).

{title="Binding the custom execution context provider", lang=javascript}
~~~~~~~
class CustomDeadboltHook extends Module {
    override def bindings(environment: Environment, 
                          configuration: Configuration): Seq[Binding[_]] = Seq(
        bind[HandlerCache].to[MyHandlerCache],
        bind[DeadboltExecutionContextProvider].to[CustomDeadboltExecutionContextProvider]
    )
}
~~~~~~~
