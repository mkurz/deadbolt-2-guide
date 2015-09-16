# Java controller constraints
If you like annotations in Java code, you're in for a treat.  If you don't, this may be a good time to consider the Scala version.

One very important point to bear in mind is the order in which Play evaluates annotations.  Annotations applied to a method are applied to annotations applied to a class.  This can lead to situations where Deadbolt method constraints deny access because information from a class constraint.  See the section on deferring method-level interceptors for a solution to this.

As with the previous chapter, here is a a breakdown of all the Java annotation-driven interceptors available in Deadbolt Java, with parameters, usages and tips and tricks.

#### Static constraints
Static constraints, are implemented entirely within Deadbolt because it can finds all the information needed to determine authorization automatically.  For example, if a constraint requires two roles, "foo" and "bar" to be present, the logic behind the `Restrict` constraint knows it just needs to check the roles of the current subject.

The static constraints available are

- SubjectPresent
- SubjectNotPresent
- Restrict
- Pattern - when using EQUALITY or REGEX

#### Dynamic constraints
Dynamic constraints are, as far as Deadbolt is concerned, completely arbitrary; they're handled by implementations of `DynamicResourceHandler`.

The dynamic constraints available are

- Dynamic
- Pattern - when using CUSTOM

{pagebreak} 

### SubjectPresent
`SubjectPresent` is one of the simplest constraints offered by Deadbolt.  It checks if there is a subject present, by invoking `DeadboltHandler#getSubject` and allows access if the result is an `Optional` containing a value.

`@SubjectPresent` can be used at the class or method level.

|Parameter                |Type                              |Default             |Notes                                             |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| content                 | String                           |""                  | A hint to indicate the content  expected in      |
|                         |                                  |                    | in the response.  This value will be passed      |
|                         |                                  |                    | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  |                    | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| handlerKey              | String                           |"defaultHandler"    | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| deferred                | boolean                          |false               | If true, the interceptor will not be applied     |
|                         |                                  |                    | until a `DeadboltDeferred` annotation is         |
|                         |                                  |                    | encountered.                                     |

**Example 1**

    @SubjectPresent
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked unless there is a subject present
            ...
        }

        public F.Promise<Result> search()
        {
            // this method will not be invoked unless there is a subject present
            ...
        }
    }


**Example 2**

    // Deny access to a single method of a controller unless there is a user present
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method is accessible to anyone
            ...
        }

        @SubjectPresent
        public F.Promise<Result> search()
        {
            // this method will not be invoked unless there is a subject present
            ...
        }
    }

{pagebreak} 

### SubjectNotPresent
`SubjectNotPresent` is the opposite in functionality of `SubjectPresent`.  It checks if there is a subject present, by invoking `DeadboltHandler#getSubject` and allows access only if the result is an empty `Optional`.

`@SubjectNotPresent` can be used at the class or method level.

|Parameter                |Type                              |Default             |Notes                                             |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| content                 | String                           |""                  | A hint to indicate the content  expected in      |
|                         |                                  |                    | in the response.  This value will be passed      |
|                         |                                  |                    | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  |                    | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| handlerKey              | String                           |"defaultHandler"    | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|--------------------|--------------------------------------------------|
| deferred                | boolean                          |false               | If true, the interceptor will not be applied     |
|                         |                                  |                    | until a `DeadboltDeferred` annotation is         |
|                         |                                  |                    | encountered.                                     |

**Example 1**

    @SubjectNotPresent
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked if there is a subject present
            ...
        }

        public F.Promise<Result> search()
        {
            // this method will not be invoked if there is a subject present
            ...
        }
    }


**Example 2**

    // Deny access to a single method of a controller if there is a user present
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method is accessible to anyone
            ...
        }

        @SubjectNotPresent
        public F.Promise<Result> search()
        {
            // this method will not be invoked unless there is not a subject present
            ...
        }
    }

{pagebreak}

### Restrict
The Restrict constraint requires that a) there is a subject present, and b) the subject has ALL the roles specified in the at least one of the `Group`s in the constraint.  The key thing to remember about `Restrict` is that it ANDs together the role names within a group and ORs between groups.

#### Notation
The role names specified in the annotation can take two forms.

1. Exact form - the subject must have a role whose name matches the required role exactly.  For example, for a constraint `@Restrict("foo")` the Subject *must* have a `Role` whose name is "foo".
2. Negated form - if the required role starts starts with a !, the constraint is negated.  For example, for a constraint `@Restrict("!foo")` the Subject *must not* have a `Role` whose name is "foo".

`@Restrict` can be used at the class or method level.

|Parameter                |Type                              |Default                |Notes                                             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| value                   | Group[]                          |                       | For each `Group`, the roles that must (or in the |
|                         |                                  |                       | case of negation, must not) be held by the       |
|                         |                                  |                       | `Subject`.  When the restriction is applied, the |
|                         |                                  |                       | `Group` instances are OR'd together.             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| content                 | String                           | ""                    | A hint to indicate the content  expected in      |
|                         |                                  |                       | in the response.  This value will be passed      |
|                         |                                  |                       | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  |                       | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| handlerKey              | String                           | "defaultHandler"      | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| deferred                | boolean                          | false                 | If true, the interceptor will not be applied     |
|                         |                                  |                       | until a `DeadboltDeferred` annotation is         |
|                         |                                  |                       | encountered.                                     |

**Example 1**

    // For every action in a controller, require the `Subject` to have *editor* and *viewer* roles
    @Restrict(@Group{"editor", "viewer"})
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked unless the subject has editor and viewer roles
            ...
        }

        public F.Promise<Result> search()
        {
            // this method will not be invoked unless the subject has editor and viewer roles
            ...
        }
    }

**Example 2**

    // Each action has different role requirements
    public class MyController extends Controller
    {
        @Restrict(@Group("editor"))
        public F.Promise<Result> edit()
        {
            // this method will not be invoked unless the subject has editor role
            ...
        }

        @Restrict(@Group("view"))
        public F.Promise<Result> view()
        {
            // this method will not be invoked unless the subject has viewer role
            ...
        }
    }

**Example 3**

    @Restrict(@Group({"editor", "!viewer"}))
    public class MyController extends Controller
    {
        public F.Promise<Result> edit()
        {
            // this method will not be invoked unless the subject has editor role AND does NOT have the viewer role
            ...
        }

        public F.Promise<Result> view()
        {
            // this method will not be invoked unless the subject has editor role AND does NOT have the viewer role
            ...
        }
    }

**Example 4**

    @Restrict({@Group("editor"), @Group("viewer")})
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked unless the subject has editor or viewer roles
            ...
        }

        public F.Promise<Result> search()
        {
            // this method will not be invoked unless the subject has editor or viewer roles
            ...
        }
    }

**Example 5**

    // Require the subject to have (customer AND viewer) OR (support AND viewer) roles
    @Restrict({@Group({"customer", "viewer"}), @Group({"support", "viewer"})})
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked unless the subject has customer and viewer,
            // or support and viewer roles
            ...
        }
    }

**Example 6**

    // Require the subject to have (customer AND NOT viewer) OR (support" AND NOT *viewer* roles
    @Restrict({@Group("customer", "!viewer"), @Group("support", "!viewer")})
    public class MyController extends Controller
    {
        public F.Promise<Result> index()
        {
            // this method will not be invoked unless the subject has customer but not viewer roles,
            // or support but not viewer roles
            ...
        }
    }

{pagebreak} 

### Dynamic

The most flexible constraint - this is a completely user-defined constraint that uses DynamicResourceHandler#isAllowed to determine access.

`@Dynamic` can be used at the class or method level.

|Parameter                |Type                              |Default                |Notes                                             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| value                   | String                           |                       | The name of the constraint.                      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| meta                    | String                           |                       | Additional information passed into `isAllowed`.  |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| content                 | String                           | ""                    | A hint to indicate the content  expected in      |
|                         |                                  |                       | in the response.  This value will be passed      |
|                         |                                  |                       | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  |                       | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| handlerKey              | String                           | "defaultHandler"      | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| deferred                | boolean                          | false                 | If true, the interceptor will not be applied     |
|                         |                                  |                       | until a `DeadboltDeferred` annotation is         |
|                         |                                  |                       | encountered.                                     |

**Example 1**

    @Dynamic(name = "name of the test")
    public F.Promise<Result> someMethod() 
    {
        // the method will execute if the user-defined test returns true
    }

{pagebreak} 

### Pattern

This uses the Subjects Permissions to perform a variety of checks.

`@Pattern` uses a `Subject`'s `Permission`s to perform a variety of checks.  The check depends on the pattern type.

- EQUALITY - the subject must have a permission whose value is exactly the same as the `value` parameter
- REGEX - the subject must have a permission which matches the regular expression given in the `value` parameter
- CUSTOM - the `DynamicResourceHandler#checkPermission` function is used to determine access 

It's possible to invert the constraint by setting the `invert` parameter to true.  This changes the meaning of the constraint in the following way.

- EQUALITY - the subject must NOT have a permission whose value is exactly the same as the `value` parameter
- REGEX - the subject must have NO permissions that match the regular expression given in the `value` parameter
- CUSTOM - the `DynamicResourceHandler#checkPermission` function, where the OPPOSITE of the Boolean resolved from the function is used to determine access

`@Pattern` can be used at the class or method level.

**A note on inverted custom constraints**

When using `invert` and `CUSTOM`, care must be taken to ensure the desired result is achieved.  For example, consider an implementation of `checkPermission` where a subject is required and that subject must have a certain attribute; access is denied if there is no subject present OR that attribute is not present.  When inverting the result, access would be allowed if the subject is not present OR that attribute is not present.  This is because when `invert` is true, the boolean resolved from `checkPermission` is negated.

If you only mean to allow access if a subject is present but does not have the attribute, you will need to engage in some annoying double negation.

Just before `checkPermission` is invoked, the value of `invert` is stored in the arguments of the HTTP context using the `ConfigKeys.PATTERN_INVERT` key; you can use this to determine what to return.

In the following example, one of four things can happen:
- A subject is present, and it satisfies the arbitrary test.
- A subject is present, and it does not satisfy the arbitrary test
- A subject is not present, and invert is true
- A subject is not present, and invert is false
- There is also a fallback assumption that invert is false if `ConfigKeys.PATTERN_INVERT` is not in the context; Deadbolt guarantees this will not happen, but it doesn't hurt to make sure

    @Override
    public F.Promise<Boolean> checkPermission(final String permissionValue,
                                              final DeadboltHandler deadboltHandler,
                                              final Http.Context ctx)
    {
        // just checking for zombies...just like I do every night before I go to bed
        return deadboltHandler.getSubject(ctx)
                              .map(option -> option.map(subject -> subject.getPermissions()
                                                                          .stream()
                                                                          .filter(perm -> perm.getValue().contains("zombie"))
                                                                          .count() > 0)
                                                   .orElseGet(() -> (Boolean) ctx.args.getOrDefault(ConfigKeys.PATTERN_INVERT,
                                                                                                    false)));
    }

So, in cases where we have a subject we just test like usual; the negation of the result, if required by `invert`, will be handled by Deadbolt.  In cases where there is no subject, we return the value of `invert` itself - if it's false, no negation will be internally applied, and if it's true it will be negated to false and access denied.  Thus, the test for `perm.getValue().contains("zombie")` is separated from the requirement to have a subject.

Double negation sucks.

|Parameter                |Type                              |Default                |Notes                                                         |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| value                   | String                           |                       | The pattern value.  Its context depends on the pattern type. |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| patternType             | PatternType                      | EQUALITY              | Additional information passed into `isAllowed`.              |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| content                 | String                           | ""                    | A hint to indicate the content  expected in                  |
|                         |                                  |                       | in the response.  This value will be passed                  |
|                         |                                  |                       | to `DeadboltHandler#onAccessFailure`.   The                  |
|                         |                                  |                       | value of this parameter is completely arbitrary.             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| handlerKey              | String                           | "defaultHandler"      | The name of a handler in the `HandlerCache`                  |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| deferred                | boolean                          | false                 | If true, the interceptor will not be applied                 |
|                         |                                  |                       | until a `DeadboltDeferred` annotation is                     |
|                         |                                  |                       | encountered.                                                 |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------------------|
| invert                  | boolean                          | false                 | Invert the result of the test.                               |


**Example 1**

    @Pattern("admin.printer")
    public F.Promise<Result> someMethodA()
    {
        // subject must have a permission with the exact value "admin.printer"
    }


**Example 2**

    @Pattern(value = "(.)*\.printer", patternType = PatternType.REGEX)
    public F.Promise<Result> someMethodB() 
    {
        // subject must have a permission that matches the regular expression (without quotes) "(.)*\.printer"
    }


**Example 3**

    @Pattern(value = "something arbitrary", patternType = PatternType.CUSTOM)
    public F.Promise<Result> someMethodC() 
    {
        // the checkPermssion method of the current handler's DynamicResourceHandler will be used.  This is a user-defined test
    }

**Example 4**

    @Pattern(value = "(.)*\.printer", patternType = PatternType.REGEX, invert = true)
    public F.Promise<Result> someMethodB() 
    {
        // subject must have no permissions that end in .printer
    }

{pagebreak}

### Unrestricted
`Unrestricted` allows you to over-ride more general constraints, i.e. if a controller is annotated with `@SubjectPresent` but you want an action in there to be accessible to everyone.

    @SubjectPresent
    public class MyController extends Controller
    {
        public F.Promise<Result> foo()
        {
            // a subject must be present for this to be accessible
        }
        
        @Unrestricted
        public F.Promise<Result> bar()
        {
            // anyone can access this action
        }
    }

You can also flip this on its head, and use it to show that a controller is explicitly unrestricted - used in this way, it's a marker of intent rather than something functional.  Because method-level constraints are evaluated first, you can have still protected actions within an `@Unrestricted` controller.

    @Unrestricted
    public class MyController extends Controller
    {
        @SubjectPresent
        public F.Promise<Result> foo()
        {
            // a subject must be present for this to be accessible
        }
        
        public F.Promise<Result> bar()
        {
            // anyone can access this action
        }
    }

`@Unrestricted` can be used at the class or method level.

|Parameter                |Type                              |Default                |Notes                                             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| content                 | String                           | ""                    | A hint to indicate the content  expected in      |
|                         |                                  |                       | in the response.  This value will be passed      |
|                         |                                  |                       | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  |                       | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| handlerKey              | String                           | "defaultHandler"      | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| deferred                | boolean                          | false                 | If true, the interceptor will not be applied     |
|                         |                                  |                       | until a `DeadboltDeferred` annotation is         |
|                         |                                  |                       | encountered.                                     |

{pagebreak}

## Deferring method-level annotation-driven interceptors
Play executes method-level annotations before controller-level annotations. This can cause issues when, for example, you want a particular action to be applied for method and before the method annotations. A good example is `Security.Authenticated(Secured.class)`, which sets a user’s user name for `request().username()`. Combining this with method-level annotations that require a user would fail, because the user would not be present at the time the method interceptor is invoked.

One way around this is to apply `Security.Authenticated` on every method, which violates DRY and causes bloat.

    public class DeferredController extends Controller
    {
        @Security.Authenticated(Secured.class)
        @Restrict(value="admin")
        public F.Promise<Result> someAdminFunction()
        {
            return F.Promise.promise(accessOk::render)
                            .map(Results::ok);
        }

        @Security.Authenticated(Secured.class)
        @Restrict(value="editor")
        public F.Promise<Result> someEditorFunction()
        {
            return F.Promise.promise(accessOk::render)
                            .map(Results::ok);
        }
    }

A better way is to set the `deferred` parameter of the Deadbolt annotation to `true`, and then use `@DeferredDeadbolt` at the controller level to execute the method-level annotations at controller-annotation time. Since annotations are processed in order of declaration, you can specify `@DeferredDeadbolt` after `@Security.Authenticated` and so achieve the desired effect.

    @Security.Authenticated(Secured.class)
    @DeferredDeadbolt
    public class DeferredController extends Controller
    {
        @Restrict(value="admin", deferred=true)
        public F.Promise<Result> someAdminFunction()
        {
            return F.Promise.promise(accessOk::render)
                            .map(Results::ok);
        }

        @Restrict(value="editor", deferred=true)
        public F.Promise<Result> someEditorFunction()
        {
            return F.Promise.promise(accessOk::render)
                            .map(Results::ok);
        }
    }

Specifying a controller-level restriction as `deferred` will work, if the annotation is higher in the annotation list than `@DeferredDeadbolt` annotation, but this is essentially pointless.  If your constraint is already at the class level, there's no need to defer it.  Just ensure it appears below any other annotations you wish to have processed first.

{pagebreak}

## Invoking DeadboltHandler#beforeAuthCheck independently
`DeadboltHandler#beforeAuthCheck` is invoked by each interceptor prior to running the actual constraint tests.  If the method call returns an empty `Optional`, the constraint is applied, otherwise the contained in the `Optional` result is returned.  The same logic can be invoked independently, using the `@BeforeAccess` annotation, in which case the call to `beforeRoleCheck` itself becomes the constraint.  Or, to say it in code,

    result = preAuth(true, ctx, deadboltHandler)
              .flatMap(preAuthResult -> preAuthResult.map(F.Promise::pure)
                                                     .orElseGet(() -> delegate.call(ctx))

#### Scope

`@BeforeAccess` can be used at the class or method level.

#### Parameters

|Parameter                |Type                              | Default               |Notes                                             |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| handlerKey              | String                           | "defaultHandler"      | The name of a handler in the `HandlerCache`      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| alwaysExecute           | boolean                          | true                  | By default, if another Deadbolt action has       |
|                         |                                  |                       | already been executed in the same request and has|
|                         |                                  |                       | allowed access, `beforeAuthCheck` will not be    |
|                         |                                  |                       | executed again.  Set this to true if you want it |
|                         |                                  |                       | to execute unconditionally.                      |
|-------------------------|----------------------------------|-----------------------|--------------------------------------------------|
| deferred                | boolean                          | false                 | If true, the interceptor will not be applied     |
|                         |                                  |                       | until a DeadboltDeferred annotation is           |
|                         |                                  |                       | encountered.                                     |

{pagebreak}

## Customising the inputs of annotation-driven actions
One of the problems with Deadbolt's annotations is they require strings to specify, for example, role names or pattern values.  It would be far safer to use enums, but this is not possible for a module - it would completely kill the generic applicability of the annotations.  If Deadbolt shipped with an enum containing roles, how would you extend it?  You would be stuck with whatever was specified, or forced to fork the codebase and customise it.  Similarly, annotations can neither implement interfaces or be extended.

To address this situation, Deadbolt has three constraints whose inputs can be customised to some degree.  The trick lies, not with inheritence, but delegation and wrapping.  The constraints are

 * Restrict
 * Dynamic
 * Pattern

Here, I'll explain how to customise the Restrict constraint to use enums as annotation parameters, but the principle is the same for each constraint.

To start, create an enum that represents your roles.

    public enum MyRoles implements Role
    {
        foo,
        bar,
        hurdy;

        @Override
        public String getRoleName()
        {
            return name();
        }
    }

To allow the AND/OR syntax that `Restrict` uses, another annotation to group them together is required.

    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface MyRolesGroup
    {
        MyRoles[] value();
    }

Next, create a new annotation to drive your custom version of Restrict.  Note that an array of `MyRoles` values can be placed in the annotation.  The standard `Restrict` annotation is also present to provide further configuration.  This means your customisations are minimised. 

    @With(CustomRestrictAction.class)
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD, ElementType.TYPE})
    @Documented
    @Inherited
    public @interface CustomRestrict
    {
        MyRolesGroup[] value();

        Restrict config();
    }

The code above contains `@With(CustomRestrictAction.class)` in order to specify the action that should be triggered by the annotation.  This action can be implemented as follows. 

    @Override
    public Result call(Http.Context context) throws Throwable
    {
        final CustomRestrict outerConfig = configuration;
        final RestrictAction restrictAction = new RestrictAction(configuration.config(),
                                                                 this.delegate)
        {
            @Override
            public List<String[]> getRoleGroups()
            {
                final List<String[]> roleGroups = new ArrayList<>();
                for (MyRolesGroup mrg : outerConfig.value())
                {
                    final List<String> group = new ArrayList<>();
                    for (MyRoles role : mrg.value())
                    {
                        group.add(role.getName());
                    }
                    roleGroups.add(group.toArray(group.size()));
                }
                return roleGroups;
            }
        };
        return restrictAction.call(context);
    }

To use your custom annotation, you apply it as you would any other Deadbolt annotation.

    @CustomRestrict(value = {MyRoles.foo, MyRoles.bar}, config = @Restrict(""))
    public static F.Promise<Result> customRestrictOne() 
    {
        return F.Promise.promise(accessOk::render)
                        .map(Results::ok);
    }

Each customisable action has one or more extension points.  These are

|Action class       |Extension points                  |
|-------------------|----------------------------------|
|RestrictAction     | * List<String[]> getRoleGroups() |
|-------------------|----------------------------------|
|DynamicAction      | * String getValue()              |
|                   | * String getMeta()               | 
|-------------------|----------------------------------|
|PatternAction      | * String getValue()              |
