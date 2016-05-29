# Using Deadbolt 2 with Play 2 Java projects

Deadbolt 2 for Java provides an idiomatic API for dealing with Java controllers and templates rendered from Java controllers in Play applications.  It takes advantage of the features such as access to the HTTP context to give access to the current request and session, and the annotation-driven interceptor support.

## The Deadbolt Handler
For any module - or framework - to be useable, it must provide a mechanism by which it can be hooked into your application.  For Deadbolt, the central hook is the `be.objectify.deadbolt.java.DeadboltHandler` interface.  The four methods defined by this interface are crucial to Deadbolt - for example, `DeadboltHandler#getSubject` gets the current subject (if any), whereas `DeadboltHandler#onAccessFailure` is used to generate a response when authorization fails.

DeadboltHandler implementations should be stateless.

For each method, a `CompletionStage` is returned - this is Java 8 interface for a future.  If the future may complete to have an empty value, e.g. calling `getSubject` when no subject is present, the return type is `CompletionStage<Optional>`.

An application may have one or more handler implementations, which will be explored when we discuss the `HandlerCache
Despite the use of the definite article in the section title, you can have as many Deadbolt handlers in your app as you wish.  These can be specified on a per-constraint basis, allowing tight contol over 

### Performing pre-constraint tests
Before a constraint is applied, the `F.Promise<Optional<Result>> beforeAuthCheck(Http.Context context)` method of the current handler is invoked.  If the resulting promise completes to a non-empty `Optional`, the target action is not invoked and instead the result of `beforeAuthCheck` is used for the HTTP response; if the resulting promise completes to an empty `Optional` the action is invoked with the Deadbolt constraint applied to it.


### Obtaining the subject
To get the current subject, the `F.Promise<Optional<Subject>> getSubject(Http.Context context)` method is invoked.  Returning an empty `Optional` indicates there is no subject present - this is a valid scenario.


### Dealing with authorization failure
When authorization fails, the `F.Promise<Result> onAccessFailure(Http.Context context, String content)` method is used to obtain a result for the HTTP response.  The result required from the `F.Promise` returned from this method is a regular `play.mvc.Result`, so it can be anything you chose.  You might want to return a 403 forbidden, redirect to a location accessible to everyone, etc.


### Dealing with dynamic constraints
Dynamic constraints, which are `Dynamic` and `Pattern.CUSTOM` constraints, are dealt with by implementations of `DynamicResourceHandler`; this will be explored in a later chapter.  For now, it's enough to say `F.Promise<Optional<DynamicResourceHandler>> getDynamicResourceHandler(Http.Context context)` is invoked when a dynamic constraint it used.


## Expose your DeadboltHandlers with a HandlerCache
Unlike earlier versions of Deadbolt, in which handlers were declared in `application.conf` and created reflectively, Deadbolt now uses dependency injection to achieve the same functionality in a type-safe and more flexible manner.  Various components of Deadbolt, which will be explored in later chapters, require an instance of `be.objectify.deadbolt.java.cache.HandlerCache` - however, no such implementations are provided.


Instead, you need to implement your own version.  This has two requirements:


* You have a get() method which returns the application-wide default handler
* You have an apply(String handlerKey) method which returns a named handler

**NB** One interesting (and annoying) quirk is the way in which template and controller constraints obtain the default handler.  The template constraints are written in Scala, so the `HandlerCache#get()` method can be used.  Controllers, on the other hand, are configured via annotations and it's not possible to have a default value of `null` for an annotation value and so the standard handler name defined by `be.objectify.deadbolt.java.ConfigKeys.DEFAULT_HANDLER_KEY;` is used.  In the example below, `HandlerKeys` is an class that defines the handler names used in the application.  To make sure we use the necessary default handler key, we re-use the `ConfigKeys` constant here.

{title="Centralising the names of handlers", lang=java}
~~~~~~~
public enum HandlerKeys
{
    DEFAULT(ConfigKeys.DEFAULT_HANDLER_KEY),
    ALT("altHandler");

    public final String key;

    private HandlerKeys(final String key)
    {
        this.key = key;
    }
}
~~~~~~~

Here's one possible implementation, using hard-coded handlers.  `defaultHandler` is kept separate to make the `get` method more efficient, instead up looking it up in the ´handlers` map every time. 

{title="Defining hard-coded handlers", lang=java}
~~~~~~~
@Singleton
public class MyHandlerCache implements HandlerCache {

    private final Map<String, DeadboltHandler> handlers = new HashMap<>();

    private final DeadboltHandler defaultHandler = new MyDeadboltHandler();

    // handler keys is an application-specific enum
    public MyHandlerCache() {
        // See below regarding the default handler
        handlers.put(HandlerKeys.DEFAULT.key, defaultHandler);
        handlers.put(HandlerKeys.ALT.key, new MyAlternativeDeadboltHandler());
    }

    @Override
    public DeadboltHandler apply(final String key) {
        return handlers.get(key);
    }

    @Override
    public DeadboltHandler get() {
        return defaultHandler;
    }
}
~~~~~~~

Ideally, handler implementations should be self-sufficient.  To this end, the `DeadboltHandler` interface contains a default method called `handlerName`.  By default, this returns the class name.  To make it more useful, implement the method and specific the handler name in it.

{title="Over-riding the handler name", lang=java}
~~~~~~~
public class MyDeadboltHandler implements DeadboltHandler {
    
    // ...

    public String handlerName() {
        return HandlerKeys.DEFAULT.key;
    }
}
~~~~~~~

This means we can change the previous implementation to the following.

{title="Using the handlerName method to map handlers", lang=java}
~~~~~~~
@Singleton
public class MyHandlerCache implements HandlerCache {

    private final Map<String, DeadboltHandler> handlers = new HashMap<>();

    private final DeadboltHandler defaultHandler = new MyDeadboltHandler();

    // handler keys is an application-specific enum
    public MyHandlerCache() {
        handlers.put(defaultHandler.handlerName(), defaultHandler);
        final DeadboltHandler altHandler = new MyAlternativeDeadboltHandler();
        handlers.put(altHandler.handlerName(), altHandler);
    }

    // ...
}
~~~~~~~

Finally, create a small module which binds your implementation.

{title="Binding your HandlerCache implementation", lang=java}
~~~~~~~
package com.example.modules

import be.objectify.deadbolt.java.cache.HandlerCache;
import play.api.Configuration;
import play.api.Environment;
import play.api.inject.Binding;
import play.api.inject.Module;
import scala.collection.Seq;
import security.MyHandlerCache;

import javax.inject.Singleton;

public class CustomDeadboltHook extends Module {
    @Override
    public Seq<Binding<?>> bindings(final Environment environment,
                                    final Configuration configuration) {
        return seq(bind(HandlerCache.class).to(MyHandlerCache.class).in(Singleton.class));
    }
}
~~~~~~~

### Using dependency injection with handlers
Both of these examples would work fine, but ignore the wide-spread and recommended use of dependency injection in Play 2.5.  You may find it more useful to inject handlers into the handler cache, especially if those handlers themselves rely on injected components.  As we've already seen, it's possible to have multiple handler implementations so we also need a way to distinguish between implementations during injection.

The trick here is to qualify the handlers.  This example will use the Guice approach; if you're using a different DI framework, you need to check if it has support for a similar mechanism.  If your application has a single handler implementation, you can skip this step.

{title="Binding annotations for multiple handlers", lang=java}
~~~~~~~
import com.google.inject.BindingAnnotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

public class HandlerQualifiers
{
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.PARAMETER})
    @BindingAnnotation
    public @interface MainHandler {}

    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.PARAMETER})
    @BindingAnnotation
    public @interface AltHandler {}
}
~~~~~~~

These annotations can then be used to tag specific implementations in a way that can be accessed during the injection phase.  For example, `MyDeadboltHandler` now has the following declaration; `MyAlternativeDeadboltHandler` has a similar declaration with the appropriate annotation.

{title="Use annotations to label implementations", lang=java}
~~~~~~~
@HandlerQualifiers.MainHandler
public class MyDeadboltHandler implements DeadboltHandler
~~~~~~~

The ´bindings´ method of the module needs to be updated to be aware of the handler implementations.

{title="Use annotations to label implementations", lang=java}
~~~~~~~
public Seq<Binding<?>> bindings(final Environment environment,
                                    final Configuration configuration) {
    return seq(bind(DeadboltHandler.class).qualifiedWith(HandlerQualifiers.MainHandler.class).to(MyDeadboltHandler.class).in(Singleton.class),
               bind(DeadboltHandler.class).qualifiedWith(HandlerQualifiers.AltHandler.class).to(MyAlternativeDeadboltHandler.class).in(Singleton.class),
               bind(HandlerCache.class).to(MyHandlerCache.class).in(Singleton.class));
}
~~~~~~~

Now, it's just a question of injecting the qualified handlers into the `HandlerCache`.

{title="Injecting handlers into the cache", lang=java}
~~~~~~~
@Singleton
public class MyHandlerCache implements HandlerCache {

    private final Map<String, DeadboltHandler> handlers = new HashMap<>();

    private final DeadboltHandler defaultHandler;

    @Inject
    public MyHandlerCache(@HandlerQualifiers.MainHandler final DeadboltHandler defaultHandler,
                          @HandlerQualifiers.AltHandler final DeadboltHandler altHandler) {
        this.defaultHandler = defaultHandler;

        handlers.put(defaultHandler.handlerName(), defaultHandler);
        handlers.put(altHandler.handlerName(), altHandler);
    }

    // ...
}
~~~~~~~


## application.conf

### Declare the necessary modules
Both `be.objectify.deadbolt.java.DeadboltModule` and your custom bindings module must be declared in the configuration.

{title="Binding your HandlerCache implementation", lang=javascript}
~~~~~~~
play {
  modules {
    enabled += be.objectify.deadbolt.java.DeadboltModule
    enabled += com.example.modules.CustomDeadboltHook
  }
}
~~~~~~~

### Tweaking Deadbolt
Deadbolt Java-specific configuration lives in the `deadbolt.java` namespace.

* `deadbolt.java.view-timeout`
  * The millisecond timeout applied to blocking calls when rendering templates.  Defaults to 1000ms.
* `deadbolt.java.cache-user`
  * A flag to indicate if the subject should be cached on a per-request basis.  This defaults to false, but setting it to true may result in significant performance gains if you're using template constraints.


Personally, I prefer the HOCON (Human-Optimized Config Object Notation) syntax supported by Play, so I would recommend the following:

{title="Tweaking the default settings", lang=javascript}
~~~~~~~
deadbolt {
  java {
    cache-user=true
    view-timeout=500
  }
}
~~~~~~~

### JPA
After all the effort I made to ensure Deadbolt is as non-blocking as possible, JPA emerged from the mist to bite me on the ass; entity managers, it seems, do not like the kind of multi-threaded usage implicit in Play's asynchronous behaviour.

Luckily, a solution is at hand.  Less luckily, it's blocking.

To address this, you can put Deadbolt into blocking mode - this ensures all DB calls made in the Deadbolt layer are made from the same thread; this has performance implications, but it's unavoidable with JPA.

To switch to blocking mode, set `deadbolt.java.blocking` to true in your configuration.

The default timeout is 1000 milliseconds - to change this, use `deadbolt.java.blocking-timeout` in your configuration.

This example configuration puts Deadbolt in blocking mode, with a timeout of 2500 milliseconds:

{title="Surrendering to the whims of JPA", lang=javascript}
~~~~~~~
deadbolt {
  java {
    blocking=true
    blocking-timeout=2500
  }
}
~~~~~~~
