# Scala controller constraints

Controller-level constraints can be added in two different ways - through the use of action builders and through action composition.  The resulting behaviour is identical, so choose whichever suites your style best.

## Controller constraints with the action builder

To get started, inject `ActionBuilders` into your controller.

    class ExampleController @Inject() (actionBuilder: ActionBuilders) extends Controller

You now have builders for all the constraint types, which we'll take a quick look at in a minute. In the following examples I'm using the default handler, i.e. `.defaultHandler()` but it's also possible to use a different handler with `.key(HandlerKey)` or pass in a handler directly using `.withHandler(DeadboltHandler)`.

## Controller constraints with action composition

Using the `DeadboltActions` class, you can compose constrained functions. To get started, inject `DeadboltActions` into your controller.

    class ExampleController @Inject() (deadbolt: DeadboltActions) extends Controller

You now have functions equivalent to those of the builders mentioned above. In the following examples I'm using the default handler, i.e. no handler is specified, but it's also possible to use a different handler with `handler = <some handler, possibly from the handler cache>`.

{pagebreak}

## SubjectPresent

Sometimes, you don't need fine-grained checks - you just need to see if there **is** a user present.

### Action builder

{title="DeadboltHandler#getSubject must result in a Some for access to be granted", lang=scala}
~~~~~~~
def someFunctionA = actionBuilder.SubjectPresentAction().defaultHandler() { Ok(accessOk()) }
~~~~~~~

### Action composition

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | HandlerCache.apply()          | The DeadboltHandler instance to use.             |


{title="DeadboltHandler#getSubject must result in a Some for access to be granted", lang=scala}
~~~~~~~
def someFunctionA = deadbolt.SubjectPresent() {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{pagebreak}

## SubjectNotPresent

Sometimes, you don't need fine-grained checks - you just need to see if there **is no** user present.

### Action builder    

{title="DeadboltHandler#getSubject must result in a None for access to be granted", lang=scala}
~~~~~~~
def someFunctionB = actionBuilder.SubjectNotPresentAction().defaultHandler() { Ok(accessOk()) }
~~~~~~~

### Action composition

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | HandlerCache.apply()          | The DeadboltHandler instance to use.             |


{title="DeadboltHandler#getSubject must result in a None for access to be granted", lang=scala}
~~~~~~~
def someFunctionB = deadbolt.SubjectNotPresent() {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{pagebreak}

## Restrict

Restrict uses a `Subject`'s `Role`s to perform AND/OR/NOT checks. The values given to the builder must match the `Role.name` of the subject's roles.

AND is defined as an `Array[String]` (or more correctly, `String*`), OR is a `List[Array[String]]`, and NOT is a rolename with a `!` preceding it.

### Action builder

|Parameter                |Type                              |Default             |Notes                                             |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| roles                   | List[Array[String]]              |                    | Allows the definition of OR'd constraints by     |
|                         |                                  |                    | having multiple arrays in the list.              |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| roles                   | String*                          |                    | A short-hand of defining a single role or AND    |
|                         |                                  |                    | constraint.                                      |


{title="The subject must have the foo role", lang=scala}
~~~~~~~
def restrictedFunctionA = actionBuilder.RestrictAction("foo")
                                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~
    
{title="The subject must have the foo AND bar roles", lang=scala}
~~~~~~~
def restrictedFunctionB = actionBuilder.RestrictAction("foo", "bar")
                                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~
    
{title="The subject must have the foo OR bar roles", lang=scala}
~~~~~~~
def restrictedFunctionC = actionBuilder.RestrictAction(List(Array("foo"), Array("bar")))
                                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~


### Action composition

|Parameter                |Type                              |Default                |Notes                                             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| roleGroups              | List[Array[String]]              |                       | Allows the definition of OR'd constraints by     |
|                         |                                  |                       | having multiple arrays in the list.              |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| handler                 | DeadboltHandler                  | HandlerCache.apply()  | The DeadboltHandler instance to use.             |


{title="The subject must have the foo role", lang=scala}
~~~~~~~
def restrictedFunctionA = deadbolt.Restrict(List(Array("foo")) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~


{title="The subject must have the foo AND bar roles", lang=scala}
~~~~~~~
def restrictedFunctionB = deadbolt.Restrict(List(Array("foo", "bar")) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{title="The subject must have the foo OR bar roles", lang=scala}
~~~~~~~
def restrictedFunctionB = deadbolt.Restrict(List(Array("foo"), Array("bar")) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{pagebreak}

## Pattern

Pattern uses a `Subject`'s `Permission`s to perform a variety of checks.  The check depends on the pattern type.

- EQUALITY - the subject must have a permission whose value is exactly the same as the `value` parameter
- REGEX - the subject must have a permission which matches the regular expression given in the `value` parameter
- CUSTOM - the `DynamicResourceHandler#checkPermission` function is used to determine access 

It's possible to invert the constraint by setting the `invert` parameter to true.  This changes the meaning of the constraint in the following way.

- EQUALITY - the subject must NOT have a permission whose value is exactly the same as the `value` parameter
- REGEX - the subject must have NO permissions that match the regular expression given in the `value` parameter
- CUSTOM - the `DynamicResourceHandler#checkPermission` function, where the OPPOSITE of the Boolean resolved from the function is used to determine access

### Action builder

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| value                   | String                 |                               | The value of the permission.                     |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| patternType             | PatternType            |                               | One of EQUALITY, REGEX or CUSTOM.                |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| invert                  | Boolean                | false                         | Invert the result of the test                    |


{title="subject must have a permission with the exact value 'admin.printer'", lang=scala}
~~~~~~~
def permittedFunctionA = actionBuilders.PatternAction(value = "admin.printer",
                                                      patternType = PatternType.EQUALITY)
                                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~


{title="subject must have a permission that matches the regular expression (without quotes) '(.)*\.printer'", lang=scala}
~~~~~~~
def permittedFunctionB = actionBuilders.PatternAction(value = "(.)*\.printer", 
                                                      patternType = PatternType.REGEX)
                                      .defaultHandler() { Ok(accessOk()) }
~~~~~~~

{title="checkPermission is used to determine access", lang=scala}
~~~~~~~
// the checkPermssion function of the current handler's DynamicResourceHandler
// will be used.  This is a user-defined test
def permittedFunctionC = actionBuilders.PatternAction(value = "something arbitrary", 
                                                      patternType = PatternType.CUSTOM)
                                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~

{title="subject must have no permissions that end in .printer", lang=scala}
~~~~~~~
def permittedFunctionB = actionBuilders.PatternAction(value = "(.)*\.printer", 
                                                      patternType = PatternType.REGEX,
                                                      invert = true)
                                      .defaultHandler() { Ok(accessOk()) }
~~~~~~~

### Action composition

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| value                   | String                 |                               | The value of the permission.                     |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| patternType             | PatternType            |                               | One of EQUALITY, REGEX or CUSTOM.                |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | HandlerCache.apply()          | The DeadboltHandler instance to use.             |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| invert                  | Boolean                | false                         | Invert the result of the test                    |


{title="subject must have a permission with the exact value 'admin.printer'", lang=scala}
~~~~~~~
def permittedFunctionA = deadbolt.Pattern(value = "admin.printer", 
                                          patternType = PatternType.EQUALITY) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{title="subject must have a permission that matches the regular expression (without quotes) '(.)*\.printer'", lang=scala}
~~~~~~~
def permittedFunctionB = deadbolt.Pattern(value = "(.)*\.printer", 
                                          patternType = PatternType.REGEX) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~


{title="checkPermission is used to determine access", lang=scala}
~~~~~~~
// the checkPermssion function of the current handler's DynamicResourceHandler
// will be used.
def permittedFunctionC = deadbolt.Pattern(value = "something arbitrary", 
                                          patternType = PatternType.CUSTOM) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~


{title="subject must have no permissions that end in .printer", lang=scala}
~~~~~~~
def permittedFunctionB = deadbolt.Pattern(value = "(.)*\.printer", 
                                          patternType = PatternType.REGEX,
                                          invert = true) {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~

{pagebreak}

## Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses `DynamicResourceHandler#isAllowed` to determine access.

### Action builder

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| name                    | String                 |                               | The name of the constraint.                      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| meta                    | String                 |                               | Additional information for the constraint        |
|                         |                        |                               | implementation to use.                           |


{title="use the constraint associated with the name 'someClassifier' to control access", lang=scala}
~~~~~~~
def foo = actionBuilder.DynamicAction(name = "someClassifier")
                       .defaultHandler() { Ok(accessOk()) }
~~~~~~~


### Action composition

|Parameter                |Type                    | Default                       | Notes                                            |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| name                    | String                 |                               | The name of the constraint.                      |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| meta                    | String                 |                               | Additional information for the constraint        |
|                         |                        |                               | implementation to use.                           |
|-------------------------|------------------------|-------------------------------|--------------------------------------------------|
| handler                 | DeadboltHandler        | HandlerCache.apply()          | The DeadboltHandler instance to use.             |


{title="use the constraint associated with the name 'someClassifier' to control access", lang=scala}
~~~~~~~
def foo = deadbolt.Dynamic(name = "someClassifier") {
  Action {
    Ok(accessOk())
  }
}
~~~~~~~
