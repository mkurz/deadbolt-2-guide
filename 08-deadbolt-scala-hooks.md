# Using Deadbolt 2 with Play 2 Scala projects

Deadbolt 2 for Scala provides an idiomatic API for dealing with Scala controllers and templates rendered from Scala controllers in Play 2 applications.

## The Deadbolt Handler
For any module - or framework - to be useable, it must provide a mechanism by which it can be hooked into your application.  For Deadbolt, the central hook is the `be.objectify.deadbolt.scala.DeadboltHandler` trait.  The four functions defined by this trait are crucial to Deadbolt - for example, `DeadboltHandler#getSubject` gets the current user (or subject, to use the correct security terminology), whereas `DeadboltHandler#onAccessFailure` is used to generate a response when authorisation fails.

DeadboltHandler implementations should be stateless.

For each method, a `Future` is returned.  If the future may complete to have an empty value, e.g. calling `getSubject` when no subject is present, the return type is `Future[Option]`.

Despite the use of the definite article in the section title, you can have as many Deadbolt handlers in your app as you wish.

### Performing pre-constraint tests
Before a constraint is applied, the `Future[Option[Result]] beforeAuthCheck[A](request: Request[A])` function of the current handler is invoked.  If the resulting future completes `Some`, the target action is not invoked and instead the result of `beforeAuthCheck` is used for the HTTP response; if the resulting future completes to `None` the action is invoked with the Deadbolt constraint applied to it.

### Obtaining the subject
To get the current subject, the `Future[Option[Subject]] getSubject[A](request: Request[A])` function is invoked.  Returning a `None` indicates there is no subject present - this is a valid scenario.

### Dealing with authorization failure
When authorisation fails, the `Future[Result] onAccessFailure[A](request: Request[A])` function is used to obtain a result for the HTTP response.  The result required from the `Future` returned from this method is a regular `play.api.mvc.Result`, so it can be anything you chose.  You might want to return a 403 forbidden, redirect to a location accessible to everyone, etc.

### Dealing with dynamic constraints
Dynamic constraints, which are `Dynamic` and `Pattern.CUSTOM` constraints, are dealt with by implementations of `DynamicResourceHandler`; this will be explored in a later chapter.  For now, it's enough to say `Future[Optional[DynamicResourceHandler]] getDynamicResourceHandler[A](request: Request[A])` is invoked when a dynamic constraint it used.

## Expose your DeadboltHandlers with a HandlerCache
Deadbolt uses dependency injection to expose handlers in a type-safe manner.  Various components of Deadbolt, which will be explored in later chapters, require an instance of `be.objectify.deadbolt.scala.cache.HandlerCache` - however, no such implementations are provided.

Instead, you need to implement your own version.  This trait extends `Function[HandlerKey, DeadboltHandler]` and `Function0[DeadboltHandler]` and uses them as follows

* `handlers()` will express the default handler
* handlers(key: HandlerKey) will express a named handler

Here's one possible implementation, using hard-coded handlers.

    @Singleton
    class MyHandlerCache extends HandlerCache {
        val defaultHandler: DeadboltHandler = new MyDeadboltHandler

        // HandlerKeys is an user-defined object, containing instances of a case class that extends HandlerKey
        val handlers: Map[Any, DeadboltHandler] = Map(HandlerKeys.defaultHandler -> defaultHandler,
                                                      HandlerKeys.altHandler -> new MyDeadboltHandler(Some(MyAlternativeDynamicResourceHandler)),
                                                      HandlerKeys.userlessHandler -> new MyUserlessDeadboltHandler)

        // Get the default handler.
        override def apply(): DeadboltHandler = defaultHandler

        // Get a named handler
        override def apply(handlerKey: HandlerKey): DeadboltHandler = handlers(handlerKey)
    }

Finally, create a small module which binds your implementation.

    package com.example.modules

    import be.objectify.deadbolt.scala.cache.HandlerCache
    import play.api.inject.{Binding, Module}
    import play.api.{Configuration, Environment}
    import com.example.security.MyHandlerCache

    class CustomDeadboltHook extends Module {
        override def bindings(environment: Environment, configuration: Configuration): Seq[Binding[_]] = Seq(
            bind[HandlerCache].to[MyHandlerCache]
        )
    }

## application.conf

### Declare the necessary modules
Both `be.objectify.deadbolt.scala.DeadboltModule` and your custom bindings module must be declared in the configuration.

    play {
      modules {
        enabled += be.objectify.deadbolt.scala.DeadboltModule
        enabled += com.example.modules.CustomDeadboltHook
      }
    }

### Tweaking Deadbolt
Deadbolt Scala-specific configuration lives in the `deadbolt.scala` namespace.

There is one setting, which is millisecond timeout applied to blocking calls when rendering templates.  This defaults to 1000ms.

Personally, I prefer the HOCON (Human-Optimized Config Object Notation) syntax supported by Play, so I would recommend the following:

    deadbolt {
        java {
            view-timeout=500
        }
    }
