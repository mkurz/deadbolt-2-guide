# Deadbolt Scala Templates


This is not a client-side DOM manipulation, but rather the exclusion of content when templates are rendered.  This also means that any logic inside the constrained content will not execute if authorization fails.


    @subjectPresent {
      <!-- paragraphs will not be rendered, satellites will not be repositioned -->
      <!-- and light speed will not be engaged if no subject is present -->
      <p>Let's see what this thing can do...</p>
      @repositionSatellite
      @engageLightSpeed
    }


One important thing to note here is that templates are blocking, so any Futures used need to be completed for the resuly to be used in the template constraints.  As a result, each constraint can take a function that expresses a Long, which is the millisecond value of the timeout.  It defaults to 1000 milliseconds, but you can change this globally by setting the `deadbolt.scala.view-timeout` value in your `application.conf`.


## Handlers


By default, template constraints use the default Deadbolt handler but as with controller constraints you can pass in a specific handler. The cleanest way to do this is to pass the handler into the template and then pass it into the constraints. Another advantage of this approach is you can pass in a wrapped version of the handler that will cache the subject; if you have a lot of constraints in a template, this can yield a significant gain.


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

{pagebreak}

## SubjectPresent


Sometimes, you don't need fine-grained checked - you just need to see if there **is a** user present.

**Parameters**

To do - detail the parameters of the constraint.

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

Sometimes, you don't need fine-grained checked - you just need to see if there **is no** user present.

**Parameters**

To do - detail the parameters of the constraint.

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


Use `Subject`s `Role`s to perform AND/OR/NOT checks.  The values given to the constraint must match the `Role.name` of the subject's roles.


AND is defined as an `Array[String]`, OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.

**Parameters**

To do - detail the parameters of the constraint.

**Example 1**

The subject must have the "foo" role.

    @restrict(roles = List(Array("foo"))) {
        Subject requires the foo role for this to be visible
    }


**Example 2**

The subject must have the "foo" AND "bar" roles.

    @restrict(List(Array("foo", "bar")) {
         Subject requires the foo AND bar roles for this to be visible
    }

**Example 3**

The subject must have the "foo" OR "bar" roles.

    @restrict(List(Array("foo"), Array("bar"))) {
         Subject requires the foo OR bar role for this to be visible
    }


**Example 3**

The subject must have the "foo" OR "bar" roles, or fallback content will be displayed.

    @restrictOr(List(Array("foo", "bar")) {
         Subject requires the foo AND bar roles for this to be visible
    } {
    	Subject does not have the necessary roles
    }

{pagebreak}

## Pattern

Use the `Subject`s `Permission`s to perform a variety of checks.

The default pattern type is `PatternType.EQUALITY`.

**Parameters**

To do - detail the parameters of the constraint.

**Example 1**

The subject and `DynamicResourceHandler` are obtained from the default handler, and must have a permission with the exact value "admin.printer".

    @pattern("admin.printer") {
        Subject must have a permission with the exact value "admin.printer" for this to be visible
    }
    
**Example 2**

The subject and `DynamicResourceHandler` are obtained from the default handler, and must have a permission that matches the specified regular expression.

    @pattern("(.)*\.printer", PatternType.REGEX) {
    	Subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer" for this to be visible
    }
    
**Example 3**

The `DynamicResourceHandler` is obtained from the default handler and used to apply the custom test

    @pattern("something arbitrary", PatternType.CUSTOM) {
    	DynamicResourceHandler#checkPermission must result in true for this to be visible
    }
    
**Example 4**

Fallback content is displayed if the user does not have a permission exactly matching "admin.printer".

    @patternOr("admin.printer") {
        Subject must have a permission with the exact value "admin.printer" for this to be visible
    } {
    	Subject did not have necessary permissions
    }

## Dynamic


The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.

**Parameters**

To do - detail the parameters of the constraint.

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
