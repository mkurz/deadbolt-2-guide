# Deadbolt Java Templates

Deadbolt's templates allow  you to customise your Twirl templates.  Unlike controller-level constraints, where a HTTP request is either allowed or denied, template constraints 
are applied during the rendering of the template and may occur multiple times within a single template.  This also means that any logic inside the constrained content will not execute if authorization fails.

{title="A subject is required for the content to be rendered", lang=scala}
~~~~~~~
@subjectPresent {
    <!-- paragraphs will not be rendered, satellites will not be repositioned -->
    <!-- and light speed will not be engaged if no subject is present -->
    <p>Let's see what this thing can do...</p>
    @repositionSatellite
    @engageLightSpeed
}
~~~~~~~

This is not a client-side DOM manipulation!

Consider a menu that needs to change based on both the presence and security level of a user.

{title="An unconstrained menu", lang=text}
~~~~~~~
Log in
Log out
Create account
My account
News
Explore
Administer something
~~~~~~~

Clearly, there are mutually exclusive items in this menu - having both "Log out" and "Log in" is strange, as is "My account" if a subject is not present.  "Administer something" should only be visible to administrators.  Using Deadbolt's template constraints, the menu can customised to reflect a more logical arrangement, and is shown here in pseudo-code.

{title="An unconstrained menu", lang=text}
~~~~~~~
if subject is present
  My account
  Log out
else
  Create account
  Log in

News
Explore
if subject is an administrator
  Administer something
~~~~~~~

- The menu for an unknown user will contain "Create account", "Log in", "News" and "Explore".
- The menu for a regular known user will contain "My account", "Log out", "News" and "Explore"
- An administrator would see "My account", "Log out", "News", "Explore" and "Administer something"

### Handlers

Template constraints use the default Deadbolt handler, as obtained via `HandlerCache#get()` but as with controller constraints you can pass in a specific handler.  The cleanest way to do this is to pass the handler into the template and then pass it into the constraints.

### Fallback content

Each constraint has an `xOr` variant, which allows you to render content in place of the unauthorized content.  This takes the form `<constraint>Or`, for example `subjectPresentOr`


{title="Providing fallback content when a subject is not present", lang=scala}
~~~~~~~
@subjectPresentOr {
    <button>Log out</button>
} {
    <button>Create an accoun</button>
    <button>Log in</button>
}
~~~~~~~


In each case, the fallback content is defined as a second `Content` block following the primary body.

### Timeouts

Because templates use blocking calls when rendering, the promises returned from the Deadbolt handler, etc, need to be completed during the rendering process.  A timeout, with a default value of 1000ms, is used to wait for the completion but you may want to change this.  You can do this in two ways.

##### Set a global timeout

If you want to change the default timeout, define `deadbolt.java.view-timeout` in your configuration and give it a millisecond value, e.g.

{title="Define timeouts in milliseconds", lang=javascript}
~~~~~~~
deadbolt {
  java {
    view-timeout=1500
  }
}
~~~~~~~

##### Use a supplier to provide a timeout

All Deadbolt templates have a `timeout` parameter which defaults to returning the app-wide value - 1000L if nothing else if defined, otherwise whatever `deadbolt.java.view-timeout` is set to.  But - and here's the nice part - the `timeout` parameter is not a `Long` but rather a `Supplier<Long>`.  This means you can use a timeout that fluctuates based on some metric - say, the number of timeouts that occur during template rendering.

##### How do I know if timeouts are occurring?

That's a good question.  And the answer is - you need to implement `be.objectify.deadbolt.java.TemplateFailureListener`.  When timeouts occur, the listener is invoked with the timeout and the exception that occurred.  The template listener should be exposed via a module - see "Expose your DeadboltHandlers with a HandlerCache" section in chapter 4 for more details on this.  If you re-use that chapter 4 module, the binding will look something like this.

{title="Declaring a template failure listener", lang=scala}
~~~~~~~
public Seq<Binding<?>> bindings(final Environment environment,
                                final Configuration configuration) {
    return seq(bind(HandlerCache.class).to(MyHandlerCache.class).in(Singleton.class),
               bind(TemplateFailureListener.class).to(MyTemplateFailureListener.class).in(Singleton.class));
}
~~~~~~~

Making it a singleton allows you to keep a running count of the failure level;  if you're using it for other purposes, then scope it accordingly.

{pagebreak}

## subjectPresent

Sometimes, you don't need fine-grained checked - you just need to see if there **is a** user present.

- be.objectify.deadbolt.java.views.html.subjectPresent
- be.objectify.deadbolt.java.views.html.subjectPresentOr


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A supplier returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


### Examples

{title="Using the default Deadbolt handler", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.{subjectPresent, subjectPresentOr}

@subjectPresent() {
   This content will be present if handler#getSubject results in a non-empty Optional
}

@subjectPresentOr() {
  This content will be present if handler#getSubject results in a non-empty Optional
} {
  This content will be present if handler#getSubject results in an empty Optional
}
~~~~~~~

{title="Using a specific Deadbolt handler", lang=scala}
~~~~~~~
@(handler: be.objectify.deadbolt.java.DeadboltHandler)
@import be.objectify.deadbolt.java.views.html.{subjectPresent, subjectPresentOr}

@subjectPresent(handler = handler) {
    This content will be present if handler#getSubject results in a non-empty Optional
}

@subjectPresentOr(handler = handler) {
  This content will be present if handler#getSubject results in a non-empty Optional
} {
  This content will be present if handler#getSubject results in an empty Optional
}
~~~~~~~

{pagebreak}

## subjectNotPresent

Just like `subjectPresent` requires a subject be present, `subjectNotPresent` requires **no** subject to be present.

- be.objectify.deadbolt.java.views.html.subjectNotPresent
- be.objectify.deadbolt.java.views.html.subjectNotPresentOr


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A supplier returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


### Examples

{title="Using the default Deadbolt handler", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.{subjectNotPresent, subjectNotPresentOr}

@subjectNotPresent() {
  This content will be present if handler#getSubject results in an empty Optional
}

@subjectNotPresentOr() {
  This content will be present if handler#getSubject results in an empty Optional
} {
  This content will be present if handler#getSubject results in a non-empty Optional
}
~~~~~~~


{title="Using a specific Deadbolt handler", lang=scala}
~~~~~~~
@(handler: be.objectify.deadbolt.java.DeadboltHandler)
@import be.objectify.deadbolt.java.views.html.{subjectNotPresent, subjectNotPresentOr}

@subjectNotPresent(handler = handler) {
  This content will be present if handler#getSubject results in an empty Optional
}

@subjectNotPresentOr(handler = handler) {
  This content will be present if handler#getSubject results in an empty Optional
} {
  This content will be present if handler#getSubject results in a non-empty Optional
}
~~~~~~~

{pagebreak}

## Restrict
Use `Subject`s `Role`s to perform AND/OR/NOT checks.  The values given to the builder must match the `Role.name` of the subject's roles.

- be.objectify.deadbolt.java.views.html.restrict
- be.objectify.deadbolt.java.views.html.restrictOr

AND is defined as an `Array[String]`, OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.

`anyOf` and `allOf` are convenience methods for creating a `List[Array[String]]` and an `Array[String]`.  There is also `allOfGroup`, which 
allows `anyOf(allOf("foo"))` to be written as `allOfGroup("foo")`.  They can be imported using `@import be.objectify.deadbolt.core.utils.TemplateUtils.{anyOf, allOf, allOfGroup}`.  

I> In earlier versions of Deadbolt, `anyOf` and `allOf` were called the far shorter and less succinct `la` (list of arrays) and `as` (array of strings).  These methods are still available, but are deprecated.


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| roles                   | List[Array[String]]    |                               | The AND/OR/NOT restrictions. One array defines   |
|                         |                        |                               | an AND, multiple arrays define OR.   See notes   |
|                         |                        |                               | on `anyOf` and `allOf` above.                    |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A supplier returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |

### Examples

{title="The subject is obtained from the default handler, and must have the foo role", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.restrict

@restrict(roles = allOfGroup("foo")) {
  Subject requires the foo role for this to be visible
}
~~~~~~~


{title="The subject is obtained from the default handler, and must have the foo AND bar role", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.restrict

@restrict(roles = allOfGroup("foo", "bar")) {
  Subject requires the foo AND bar roles for this to be visible
}
~~~~~~~


{title="The subject is obtained from the default handler, and must have the foo OR bar role", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.restrict

@restrict(roles = anyOf(allOf("foo"), allOf("bar", "hurdy"))) {
  Subject requires the foo OR (bar and hurdy) roles for this to be visible
}
~~~~~~~


{title="Providing fallback content", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.restrictOr

@restrictOr(roles = allOfGroup("foo", "bar")) {
  Subject requires the foo AND bar roles for this to be visible
} {
  Subject does not have the necessary roles
}
~~~~~~~


{title="Getting the subject via a specific Deadbolt handler", lang=scala}
~~~~~~~
@(handler: be.objectify.deadbolt.java.DeadboltHandler)
@import be.objectify.deadbolt.java.views.html.restrict

@restrict(roles = allOfGroup("foo"), handler = handler) {
  Subject requires the foo role for this to be visible
}
~~~~~~~

{pagebreak}

## Pattern

Use the `Subject`s `Permission`s to perform a variety of checks.  This may be a regular expression match, an exact match or
some custom constraint using `DynamicResourceHandler#checkPermission`.

- be.objectify.deadbolt.java.views.html.pattern
- be.objectify.deadbolt.java.views.html.patternOr


|Parameter                |Type                    | Default                       | Notes                                                |
|-------------------------|------------------------|-------------------------------|------------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.          |
|-------------------------|------------------------|-------------------------------|------------------------------------------------------|
| value                   | String                 |                               | The value of the pattern, e.g. a regex or a          |
|                         |                        |                               | precise match.                                       |
|-------------------------|------------------------|-------------------------------|------------------------------------------------------|
| patternType             | PatternType            | PatternType.EQUALITY          |                                                      |
|                         |                        |                               |                                                      |
|-------------------------|------------------------|-------------------------------|------------------------------------------------------|
| invert                  | Boolean                | false                         | If true, do **NOT** render content if the subject's  |
|                         |                        |                               | permissions satisfy the pattern.                     |
|-------------------------|------------------------|-------------------------------|------------------------------------------------------|
| timeout                 | () -> Long             | A supplier returning          | The timeout applied to blocking calls.               |
|                         |                        | `deadbolt.java.view-timeout`  |                                                      |
|                         |                        | if it's defined, otherwise    |                                                      |
|                         |                        | 1000L                         |                                                      |


### Examples

{title="The subject must have a permission with the exact value 'admin.printer'", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.pattern

@pattern(value = "admin.printer") {
  Subject must have a permission with the exact value "admin.printer" for this to be visible
}
~~~~~~~


{title="The subject must have a permission that matches the specified regular expression", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.pattern

@pattern(value = "(.)*\.printer", patternType = PatternType.REGEX) {
  Subject must have a permission that matches the regular expression (.)*\.printer for this to be visible
}
~~~~~~~


{title="A custom test is applied to determine access", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.pattern

@pattern(value = "something arbitrary", patternType = PatternType.CUSTOM) {
  DynamicResourceHandler#checkPermission must result in true for this to be visible
}
~~~~~~~


{title="Providing fallback content", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.patternOr

@patternOr(value = "something arbitrary", patternType = PatternType.CUSTOM) {
  DynamicResourceHandler#checkPermission must result in true for this to be visible
} {
  Tough luck
}
~~~~~~~


{title="Using a specific Deadbolt handler", lang=scala}
~~~~~~~
@(handler: be.objectify.deadbolt.java.DeadboltHandler)
@import be.objectify.deadbolt.java.views.html.pattern

@pattern(handler = handler, value = "(.)*\.printer", patternType = PatternType.REGEX) {
  Subject must have a permission that matches the regular expression (.)*\.printer for this to be visible
}
~~~~~~~


{title="Inverting the constraint", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.pattern

@pattern(value = "(.)*\.printer", patternType = PatternType.REGEX, invert = true) {
  Subject must have no permissions that end with '.printer'
}
~~~~~~~

{pagebreak}

## Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.

- be.objectify.deadbolt.java.views.html.dynamic
- be.objectify.deadbolt.java.views.html.dynamicOr


|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | handlerCache.get()            | The handler to use to apply the constraint.      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| name                    | String                 |                               | The name of the constraint, passed into the      |
|                         |                        |                               | `DynamicResourceHandler`.                        |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| meta                    | String                 | null                          |                                                  |
|                         |                        |                               |                                                  |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| timeout                 | () -> Long             | A supplier returning          | The timeout applied to blocking calls.           |
|                         |                        | `deadbolt.java.view-timeout`  |                                                  |
|                         |                        | if it's defined, otherwise    |                                                  |
|                         |                        | 1000L                         |                                                  |


### Examples

{title="The `DynamicResourceHandler` is obtained from the default handler and is used to apply a named constraint to the content", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.dynamic

@dynamic(name = "someName") {
  DynamicResourceHandler#isAllowed must result in true for this to be visible
}
~~~~~~~


{title="Providing meta data to the constraint", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.dynamic

@dynamic(name = "someName", meta = "foo") {
  DynamicResourceHandler#isAllowed must result in true for this to be visible
}
~~~~~~~


{title="Meta data does not have to be hard-coded", lang=scala}
~~~~~~~
@(someMetaValue: String)
@import be.objectify.deadbolt.java.views.html.dynamic

@dynamic(name = "someName", meta = someMetaValue) {
  DynamicResourceHandler#isAllowed must result in true for this to be visible
}
~~~~~~~


{title="Providing fallback content", lang=scala}
~~~~~~~
@import be.objectify.deadbolt.java.views.html.dynamicOr

@dynamicOr(name = "someName") {
  DynamicResourceHandler#isAllowed must result in true for this to be visible
} {
  Better luck next time
}
~~~~~~~


{title="Using a specific Deadbolt handler", lang=scala}
~~~~~~~
@(handler: be.objectify.deadbolt.java.DeadboltHandler)
@import be.objectify.deadbolt.java.views.html.dynamic

@dynamic(handler = handler, name = "someName") {
  DynamicResourceHandler#isAllowed must result in true for this to be visible
}
~~~~~~~
