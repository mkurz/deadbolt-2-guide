# Deadbolt Java Templates

Before you get your hopes up that Play templates can be written in Java, I'm talking about Scala templates rendered from Java controllers.


Using template constraints, you can exclude portions of templates from being generated on the server-side.  Template constraints have the same possibilities as controller constraints.


This is not a client-side DOM manipulation, but rather the exclusion of content when templates are rendered.  This also means that any logic inside the constrained content will not execute if authorization fails.


    @subjectPresent {
        <!-- paragraphs will not be rendered, satellites will not be repositioned -->
        <!-- and light speed will not be engaged if no subject is present -->
        <p>Let's see what this thing can do...</p>
        @repositionSatellite
        @engageLightSpeed
    }


One important thing to note here is that templates are blocking, so any Futures used need to be completed for the resuly to be used in the template constraints.  As a result, each constraint can take a function that expresses a Long, which is the millisecond value of the timeout.  It defaults to 1000 milliseconds, but you can change this globally by setting the `deadbolt.java.view-timeout` value in your `application.conf`.

### Handlers

By default, template constraints use the default Deadbolt handler, as obtained via `<YourDeadboltHandlerImpl>#get()` but as with controller constraints you can pass in a specific handler.  The cleanest way to do this is to pass the handler into the template and then pass it into the constraints.

### Fallback content

Each constraint has an `xOr` variant, which allows you to render content in place of the unauthorized content.  This takes the form `<constraint>Or`, for example `subjectPresentOr`


    @subjectPresentOr {
        <!-- paragraphs will not be rendered, satellites will not be repositioned -->
        <!-- and light speed will not be engaged if no subject is present -->
        <p>Let's see what this thing can do...</p>
        @repositionSatellite
        @engageLightSpeed
    } {
        <marquee>Sorry, you are not authorised to perform clichéd actions from B movies.  Suffer the marquee!</marquee>
    }


In each case, the fallback content is defined as a second `Content` block following the primary body.

### Timeouts

Because templates use blocking calls when rendering, the promises returned from the Deadbolt handler, etc, need to be completed during the rendering process.  A timeout, with a default value of 1000ms, is used to wait for the completion but you may want to change this.  You can do this in two ways.

##### Set a global timeout

If you want to change the default timeout, define `deadbolt.java.view-timeout` in your configuration and give it a millisecond value, e.g.

    deadbolt {
      java {
        view-timeout=1500
      }
    }

##### Use a supplier to provide a timeout

All Deadbolt templates have a `timeout` parameter which defaults to returning the app-wide value - 1000L if nothing else if defined, otherwise whatever `deadbolt.java.view-timeout` is set to.  But - and here's the nice part - the `timeout` parameter is not a `Long` but rather a `Supplier<Long>`.  This means you can use a timeout that fluctuates based on some metric - say, the number of timeouts that occur during template rendering.

**How do I know if timeouts are occurring?**

That's a good question.  And the answer is - you need to implement `be.objectify.deadbolt.java.TemplateFailureListener` and bind it using a module; see "Expose your DeadboltHandlers with a HandlerCache" section in chapter 4 for more details on this.  If you re-use that chapter 4 module, the binding will look something like this.

    public Seq<Binding<?>> bindings(final Environment environment,
                                    final Configuration configuration) {
        return seq(bind(HandlerCache.class).to(MyHandlerCache.class).in(Singleton.class),
                   bind(TemplateFailureListener.class).to(MyTemplateFailureListener.class).in(Singleton.class));
    }

Making it a singleton allows you to keep a running count of the failure level;  if you're using it for other purposes, then scope it accordingly.

{pagebreak}

## SubjectPresent

Sometimes, you don't need fine-grained checked - you just need to see if there **is a** user present


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A function returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


**Example 1**

The default Deadbolt handler is used to obtain the subject.


    @subjectPresent() {
        This content will be present if handler#getSubject results in a Some
    }

    @subjectPresentOr() {
        This content will be present if handler#getSubject results in a Some
    } {
    	fallback content
    }


**Example 2**

A specific Deadbolt handler is used to obtain the subject.


    @(handler: DeadboltHandler)
    @subjectPresent(handler = handler) {
        This content will be present if handler#getSubject results in a Some
    }

    @subjectPresentOr(handler = handler) {
        This content will be present if handler#getSubject results in a Some
    } {
    	fallback content
    }

{pagebreak}

## SubjectNotPresent

Sometimes, you don't need fine-grained checked - you just need to see if there **is no** user present


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A function returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


**Example 1**

The default Deadbolt handler is used to obtain the subject.

    @subjectNotPresent() {
        This content will be present if handler#getSubject results in a None
    }

    @subjectNotPresentOr() {
        This content will be present if handler#getSubject results in a None
    } {
    	fallback content
    }


**Example 2**

A specific Deadbolt handler is used to obtain the subject.

    @(handler: DeadboltHandler)
    @subjectNotPresent(handler = handler) {
        This content will be present if handler#getSubject results in a None
    }

    @subjectNotPresentOr(handler = handler) {
        This content will be present if handler#getSubject results in a None
    } {
    	fallback content
    }

{pagebreak}

## Restrict
Use `Subject`s `Role`s to perform AND/OR/NOT checks.  The values given to the builder must match the `Role.name` of the subject's roles.


`la` and `as` are convenience functions for creating a `List<Array<String>>` and an `Array<String>`.  You can import both of them using


    @import be.objectify.deadbolt.core.utils.TemplateUtils.{la, as}


AND is defined as an `Array[String]`, OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| roles                   | List[Array[String]]    |                               | The AND/OR/NOT restrictions. One array defines   |
|                         |                        |                               | an AND, multiple arrays define OR.   See notes   |
|                         |                        |                               | on `la` and `as` above.                          |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A function returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


**Example 1**

The subject is obtained from the default handler, and must have the foo role.


    @restrict(roles = la(as("foo"))) {
        Subject requires the foo role for this to be visible
    }


**Example 2**

The subject is obtained from the default handler, and must have the foo AND bar role.


    @restrict(roles = la(as("foo", "bar")) {
         Subject requires the foo AND bar roles for this to be visible
    }


**Example 3**

The subject is obtained from the default handler, and must have the foo OR bar role.


    @restrict(roles = la(as("foo"), as("bar"))) {
         Subject requires the foo OR bar role for this to be visible
    }


**Example 4**

The subject is obtained from the default handler, and must have the foo AND bar role; otherwise, fallback content will be rendered.


    @restrictOr(roles = la(as("foo", "bar"))) {
         Subject requires the foo AND bar roles for this to be visible
    } {
    	Subject does not have the necessary roles
    }


**Example 5**

The subject is obtained from a specific handler, and must have the foo OR bar role.

    @(handler: DeadboltHandler)
    @restrict(roles = la(as("foo"), as("bar")), handler = handler) {
         Subject requires the foo OR bar role for this to be visible
    }

{pagebreak}

## Pattern

Use the `Subject`s `Permission`s to perform a variety of checks.


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| value                   | String                 |                               | The value of the pattern, e.g. a regex or a      |
|                         |                        |                               | precise match.                                   |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| patternType             | PatternType            | PatternType.EQUALITY          |                                                  |
|                         |                        |                               |                                                  |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A function returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


**Example 1**

The subject and `DynamicResourceHandler` are obtained from the default handler, and must have a permission with the exact value "admin.printer".


    @pattern(value = "admin.printer") {
        Subject must have a permission with the exact value "admin.printer" for this to be visible
    }


**Example 2**

The subject and `DynamicResourceHandler` are obtained from the default handler, and must have a permission that matches the specified regular expression.


    @pattern(value = "(.)*\.printer", patternType = PatternType.REGEX) {
    	Subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer" for this to be visible
    }


**Example 3**

The `DynamicResourceHandler` is obtained from the default handler and used to apply the custom test


    @pattern(value = "something arbitrary", patternType = PatternType.CUSTOM) {
    	DynamicResourceHandler#checkPermission must result in true for this to be visible
    }


**Example 4**

The subject and `DynamicResourceHandler` are obtained from a specific handler, and must have a permission that matches the specified regular expression.


    @(handler: DeadboltHandler)
    @pattern(handler = handler, value = "(.)*\.printer", patternType = PatternType.REGEX) {
    	Subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer" for this to be visible
    }

{pagebreak}

## Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| name                    | String                 |                               | The name of the constraint, passed into the      |
|                         |                        |                               | `DynamicResourceHandler`.                        |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| meta                    | PatternType            | null                          |                                                  |
|                         |                        |                               |                                                  |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A function returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


**Example 1**

The `DynamicResourceHandler` is obtained from the default handler and is used to apply a named constraint to the content.


    @dynamic(name = "someName") {
        DynamicResourceHandler#isAllowed must result in true for this to be visible
    }


**Example 2**

The `DynamicResourceHandler` is obtained from the default handler and is used to apply a named constraint to the content with some hard-coded meta data.


    @dynamic(name = "someName", meta = "foo") {
        DynamicResourceHandler#isAllowed must result in true for this to be visible
    }


**Example 3**

The `DynamicResourceHandler` is obtained from the default handler and is used to apply a named constraint to the content with some dynamically-defined meta data.


    @(someMetaValue: String)
    @dynamic(name = "someName", meta = someMetaValue) {
        DynamicResourceHandler#isAllowed must result in true for this to be visible
    }


**Example 4**

The `DynamicResourceHandler` is obtained from a specific handler and is used to apply a named constraint.


    @(handler: DeadboltHandler)
    @dynamic(handler = handler, name = "someName") {
        DynamicResourceHandler#isAllowed must result in true for this to be visible
    }
