# Deadbolt Scala Controllers

Controller-level constraints can be added in two different ways - through the use of action builders and through action composition.  The resulting behaviour is identical, so choose whichever suites your style best.

## Controller constraints with the action builder

To get started, inject `ActionBuilders` into your controller.

    class ExampleController @Inject() (actionBuilder: ActionBuilders) extends Controller

You now have builders for all the constraint types, which we'll take a quick look at in a minute. In the following examples I'm using the default handler, i.e. `.defaultHandler()` but it's also possible to use a different handler with `.key(HandlerKey)` or pass in a handler directly using `.withHandler(DeadboltHandler)`.

### SubjectPresent and SubjectNotPresent

Sometimes, you don't need fine-grained checks - you just need to see if there is a user present (or not present).

    // DeadboltHandler#getSubject must result in a Some for access to be granted
    def someFunctionA = actionBuilder.SubjectPresentAction().defaultHandler() { Ok(accessOk()) }
    
    // DeadboltHandler#getSubject must result in a None for access to be granted
    def someFunctionB = actionBuilder.SubjectNotPresentAction().defaultHandler() { Ok(accessOk()) }

### Restrict

Restrict uses a `Subject`'s `Role`s to perform AND/OR/NOT checks. The values given to the builder must match the `Role.name` of the subject's roles.

AND is defined as an `Array[String]` (or more correctly, `String*`), OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.

    // subject must have the "foo" role 
    def restrictedFunctionA = actionBuilder.RestrictAction("foo")
                                           .defaultHandler() { Ok(accessOk()) }
    
    // subject must have the "foo" AND "bar" roles 
    def restrictedFunctionB = actionBuilder.RestrictAction("foo", "bar")
                                          .defaultHandler() { Ok(accessOk()) }
    
    // subject must have the "foo" OR "bar" roles 
    def restrictedFunctionC = actionBuilder.RestrictAction(List(Array("foo"), Array("bar")))
                                           .defaultHandler() { Ok(accessOk()) }

### Pattern

Pattern uses a `Subject`'s `Permission`s to perform a variety of checks.

    // subject must have a permission with the exact value "admin.printer" 
    def permittedFunctionA = actionBuilders.PatternAction(value = "admin.printer",
                                                          patternType = PatternType.EQUALITY)
                                           .defaultHandler() { Ok(accessOk()) }
    
    // subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer" 
    def permittedFunctionB = actionBuilders.PatternAction(value = "(.)*\.printer", 
                                                          patternType = PatternType.REGEX)
                                          .defaultHandler() { Ok(accessOk()) }
    
    // the checkPermssion function of the current handler's DynamicResourceHandler will be used.  This is a user-defined test     def permittedFunctionC = actionBuilders.PatternAction(value = "something arbitrary", 
                                                          patternType = PatternType.CUSTOM)
                                           .defaultHandler() { Ok(accessOk()) }

### Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.

    // use the constraint associated with the name "someClassifier" to control access
    def foo = actionBuilder.DynamicAction(name = "someClassifier")
                           .defaultHandler() { Ok(accessOk()) }


## Controller constraints with action composition

Using the `DeadboltActions` class, you can compose constrained functions. To get started, inject `DeadboltActions` into your controller.

    class ExampleController @Inject() (deadbolt: DeadboltActions) extends Controller

You now have functions equivalent to those of the builders mentioned above. In the following examples I'm using the default handler, i.e. no handler is specified, but it's also possible to use a different handler with `handler = <some handler, possibly from the handler cache>`.

### SubjectPresent and SubjectNotPresent

Sometimes, you don't need fine-grained checks - you just need to see if there is a user present (or not present).

And yes, that line was copy and pasted from the action builder section for the same constraints to underline one thing: the constraint behaviour is identical, no matter how you prefer to define them.

    // DeadboltHandler#getSubject must result in a Some for access to be granted
    def someFunctionA = deadbolt.SubjectPresent() {
      Action {
        Ok(accessOk())
      }
    }

    // DeadboltHandler#getSubject must result in a None for access to be granted
    def someFunctionB = deadbolt.SubjectNotPresent() {
      Action {
        Ok(accessOk())
      }
    }
    
### Restrict
    
Restrict uses a `Subject`'s `Role`s to perform AND/OR/NOT checks. The values given to the builder must match the `Role.name` of the subject's roles.

AND is defined as an `Array[String]` (or more correctly, `String*`), OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.

    // subject must have the "foo" role 
    def restrictedFunctionA = deadbolt.Restrict(List(Array("foo")) {
      Action {
        Ok(accessOk())
      }
    }

    // subject must have the "foo" AND "bar" roles 
    def restrictedFunctionB = deadbolt.Restrict(List(Array("foo", "bar")) {
      Action {
        Ok(accessOk())
      }
    }

    // subject must have the "foo" OR "bar" roles 
    def restrictedFunctionC = deadbolt.Restrict(List(Array("foo"), Array("bar"))) {
      Action {
        Ok(accessOk())
      }
    }

## Pattern

Pattern uses a `Subject`'s `Permission`s to perform a variety of checks.

    // subject must have a permission with the exact value "admin.printer" 
    def permittedFunctionA = deadbolt.Pattern("admin.printer", PatternType.EQUALITY) {
      Action {
        Ok(accessOk())
      }
    }

    // subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer" 
    def permittedFunctionB = deadbolt.Pattern("(.)*\.printer", PatternType.REGEX) {
      Action {
        Ok(accessOk())
      }
    }
    
    // the checkPermssion function of the current handler's DynamicResourceHandler will be used.  This is a 
    // user-defined test in DynamicResourceHandler#checkPermission.
    def permittedFunctionC = deadbolt.Pattern("something arbitrary", PatternType.CUSTOM) {
      Action {
        Ok(accessOk())
      }
    }

### Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.

    // use the constraint associated with the name "someClassifier" to control access
    def foo = deadbolt.Dynamic(name = "someClassifier") {
      Action {
        Ok(accessOk())
      }
    }