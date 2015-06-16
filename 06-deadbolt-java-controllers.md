# Java controller constraints
If you like annotations in Java code, you're in for a treat.  If you don't, this may be a good time to consider the Scala version.

One very important point to bear in mind is the order in which Play! 2 evaluates annotations.  Annotations applied to a method are applied to annotations applied to a class.  This can lead to situations where D2 method constraints deny access because information from a class constraint.  See the section on deferring method-level interceptors for a solution to this.

As with previous chapter, here is a a breakdown of all the Java annotation-driven interceptors available in D2 Java, with parameters, usages and tips and tricks.

## Static constraints
Static constraints, such as Restrict, are implemented entirely within D2 because it can finds all the information needed to determine authorisation automatically.  For example, if a constraint requires two roles, "foo" and "bar" to be present, the logic behind the Restrict constraint knows it just needs to check the roles of the current roles holder.

#### SubjectPresent
`SubjectPresent` is one of the simplest constraints offered by D2.  It checks if there is a subject present, by invoking `DeadboltHandler#getSubject` and allows access if the result is not null.

##### Scope
`@SubjectPresent` can be used at the class or method level.

##### Parameters

|Parameter                |Type                              |Notes                                             |
|-------------------------|----------------------------------|--------------------------------------------------|
| content                 | String                           | A hint to indicate the content  expected in      | 
|                         |                                  | in the response.  This value will be passed      |
|                         |                                  | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------------------------------------|
| handler                 | Class<? extends DeadboltHandler> | The **class** of a `DeadboltHandler` to use  in  |
|                         |                                  | in place of the default one.                     |
|-------------------------|----------------------------------|--------------------------------------------------|
| deferred                | Boolean                          | If true, the interceptor will not be applied     |
|                         |                                  | until a `DeadboltDeferred` annotation is         |
|                         |                                  | encountered.                                     |

##### Example 1. Deny access to any method of a controller unless there is a user present

    @SubjectPresent
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked unless there is a subject present
            ...
        }

        public static Result search() {
            // this method will not be invoked unless there is a subject present
            ...
        }
    }


##### Example 2. Deny access to a single method of a controller unless there is a user present

    public class MyController extends Controller {

        public static Result index() {
            // this method is accessible to anyone
            ...
        }

        @SubjectPresent
        public static Result search() {
            // this method will not be invoked unless there is a subject present
            ...
        }
    }

 {pagebreak} 

#### SubjectNotPresent
`SubjectNotPresent` is the opposite in functionality of `SubjectPresent`.  It checks if there is a subject present, by invoking `DeadboltHandler#getSubject` and allows access only if the result is null.

##### Scope
`@SubjectNotPresent` can be used at the class or method level.

##### Parameters

|Parameter                |Type                              |Notes                                             |
|-------------------------|----------------------------------|--------------------------------------------------|
| content                 | String                           | A hint to indicate the content  expected in      | 
|                         |                                  | in the response.  This value will be passed      |
|                         |                                  | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------------------------------------|
| handler                 | Class<? extends DeadboltHandler> | The **class** of a `DeadboltHandler` to use  in  |
|                         |                                  | in place of the default one.                     |
|-------------------------|----------------------------------|--------------------------------------------------|
| deferred                | Boolean                          | If true, the interceptor will not be applied     |
|                         |                                  | until a `DeadboltDeferred` annotation is         |
|                         |                                  | encountered.                                     |

##### Example 1. Deny access to any method of a controller if there is a user present

    @SubjectNotPresent
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked if there is a subject present
            ...
        }

        public static Result search() {
            // this method will not be invoked if there is a subject present
            ...
        }
    }


##### Example 2. Deny access to a single method of a controller if there is a user present

    public class MyController extends Controller {

        public static Result index() {
            // this method is accessible to anyone
            ...
        }

        @SubjectNotPresent
        public static Result search() {
            // this method will not be invoked unless there is not a subject present
            ...
        }
    }

 {pagebreak} 

#### Restrict
The Restrict constraint requires that a) there is a subject present, and b) the subject has ALL the roles specified in the constraint.  The key thing to remember about ´Restrict´ is that it ANDs together the role names.

##### Notation
The role names specified in the annotation can take two forms.

1. Exact form - the subject must have a role whose name matches the required role exactly.  For example, for a constraint `@Restrict("foo")` the Subject *must* have a `Role` whose name is "foo".
2. Negated form - if the required role starts starts with a !, the constraint is negated.  For example, for a constraint `@Restrict("!foo")` the Subject *must not* have a `Role` whose name is "foo".

##### Scope
`@Restrict` can be used at the class or method level.

##### Parameters

|Parameter                |Type                              |Notes                                             |
|-------------------------|----------------------------------|--------------------------------------------------|
| value                   | String[]                         | The roles that must (or in the case of negation, | 
|                         |                                  | must not) be held by the `Subject`.              |
|-------------------------|----------------------------------|--------------------------------------------------|
| content                 | String                           | A hint to indicate the content  expected in      | 
|                         |                                  | in the response.  This value will be passed      |
|                         |                                  | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------------------------------------|
| handler                 | Class<? extends DeadboltHandler> | The **class** of a `DeadboltHandler` to use  in  |
|                         |                                  | in place of the default one.                     |
|-------------------------|----------------------------------|--------------------------------------------------|
| deferred                | Boolean                          | If true, the interceptor will not be applied     |
|                         |                                  | until a `DeadboltDeferred` annotation is         |
|                         |                                  | encountered.                                     |

##### Example 1. For every action in a controller, require the `Subject` to have *editor* and *viewer* roles

    @Restrict({"editor", "viewer"})
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked unless the subject has editor and viewer roles
            ...
        }

        public static Result search() {
            // this method will not be invoked unless the subject has editor and viewer roles
            ...
        }
    }

##### Example 2. Have different requires for each action in the controller.  Note the, because only one role is specified, the annotation doesn't require array notation

    public class MyController extends Controller {

        @Restrict("editor")
        public static Result edit() {
            // this method will not be invoked unless the subject has editor role
            ...
        }

        @Restrict("view")
        public static Result view() {
            // this method will not be invoked unless the subject has viewer role
            ...
        }
    }

##### Example 3. Ensure the ´Subject` has the *editor* but not the *viewer* role

    @Restrict({"editor", "!viewer"})
    public class MyController extends Controller {

        public static Result edit() {
            // this method will not be invoked unless the subject has editor role AND does not have the viewer role
            ...
        }

        public static Result view() {
            // this method will not be invoked unless the subject has editor role AND does not have the viewer role
            ...
        }
    }

 {pagebreak} 

#### Restrictions
The Restrictions constraint requires that a) there is a subject present, and b) the subject has ALL the roles specified in one of the _role groups_ of the constraint.  `Restrictions` can be thought of as a way to OR together AND'd constraints, equivalent to `@Restrict(x) OR @Restrict(y) OR @Restrict(z)`.

##### Notation
`Restrictions` uses the `And` annotation to define String[] arrays, in the same way that `Restrict` has a String array as a parameter.  Within an `And`, the syntax is exactly like that of `Restrict#values`, with exact and negated forms.

##### Scope
`@Restrictions` can be used at the class or method level.

##### Parameters

|Parameter                |Type                              |Notes                                             |
|-------------------------|----------------------------------|--------------------------------------------------|
| value                   | And[]                            | For each `And`, the roles that must (or in the   |
|                         |                                  | case of negation, must not) be held by the       |
|                         |                                  | `Subject`.  When the restriction is applied, the | 
|                         |                                  | `And` instances are OR'd together.               |
|-------------------------|----------------------------------|--------------------------------------------------|
| content                 | String                           | A hint to indicate the content  expected in      | 
|                         |                                  | in the response.  This value will be passed      |
|                         |                                  | to `DeadboltHandler#onAccessFailure`.   The      |
|                         |                                  | value of this parameter is completely arbitrary. |
|-------------------------|----------------------------------|--------------------------------------------------|
| handler                 | Class<? extends DeadboltHandler> | The **class** of a `DeadboltHandler` to use  in  |
|                         |                                  | in place of the default one.                     |
|-------------------------|----------------------------------|--------------------------------------------------|
| deferred                | Boolean                          | If true, the interceptor will not be applied     |
|                         |                                  | until a `DeadboltDeferred` annotation is         |
|                         |                                  | encountered.                                     |

##### Example 1. Require the `Subject` to have *editor* or *viewer* roles

    @Restrictions({@And("editor"), @And("viewer")})
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked unless the subject has editor or viewer roles
            ...
        }

        public static Result search() {
            // this method will not be invoked unless the subject has editor or viewer roles
            ...
        }
    }

##### Example 2. Require the `Subject` to have *customer* AND "viewer", OR "support" AND *viewer* roles

    @Restrictions({@And("customer", "viewer"), @And("support", "viewer")})
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked unless the subject has customer and viewer,
            // or support and viewer roles
            ...
        }
    }

##### Example 2. Require the `Subject` to have *customer* AND NOT "viewer", OR "support" AND NOT *viewer* roles

    @Restrictions({@And("customer", "!viewer"), @And("support", "!viewer")})
    public class MyController extends Controller {

        public static Result index() {
            // this method will not be invoked unless the subject has customer but not viewer roles,
            // or support but not viewer roles
            ...
        }
    }

 {pagebreak} 

#### Unrestricted
todo

 {pagebreak} 

## Dynamic constraints
Dynamic constraints are, as far as D2 is concerned, completely arbitrary.  The logic behind the constraint comes entirely from a third party.

 {pagebreak} 

#### Dynamic
todo

 {pagebreak} 

#### Pattern
todo

 {pagebreak} 
 
## Deferring method-level annotation-driven interceptors
Play executes method-level annotations before controller-level annotations. This can cause issues when, for example, you want a particular action to be applied for method and before the method annotations. A good example is `Security.Authenticated(Secured.class)`, which sets a user’s user name for `request().username()`. Combining this with method-level annotations that require a user would fail, because the user would not be present at the time the method interceptor is invoked.

One way around this is to apply `Security.Authenticated` on every method, which violates DRY and causes bloat.

    public class DeferredController extends Controller {

        @Security.Authenticated(Secured.class)
        @Restrict(value="admin")
        public static Result someAdminFunction() {
            return ok(accessOk.render());
        }

        @Security.Authenticated(Secured.class)
        @Restrict(value="editor")
        public static Result someEditorFunction() {
            return ok(accessOk.render());
        }
    }

A better way is to set the `deferred` parameter of the D2 annotation to `true`, and then use `@DeferredDeadbolt` at the controller level to execute the method-level annotations at controller-annotation time. Since annotations are processed in order of declaration, you can specify `@DeferredDeadbolt` after `@Security.Authenticated` and so achieve the desired effect.

    @Security.Authenticated(Secured.class)
    @DeferredDeadbolt
    public class DeferredController extends Controller {

        @Restrict(value="admin", deferred=true)
        public static Result someAdminFunction() {
            return ok(accessOk.render());
        }

        @Restrict(value="admin", deferred=true)
        public static Result someAdminFunction() {
            return ok(accessOk.render());
        }
    }

Specifying a controller-level restriction as deferred will work, if the annotation is higher in the annotation list than `@DeferredDeadbolt` annotation, but this is essentially pointless.  If your constraint is already at the class level, there's no need to defer it.  Just ensure it appears below any other annotations you wish to have processed first.

## Invoking DeadboltHandler#beforeAuthCheck independently
`DeadboltHandler#beforeAuthCheck` is invoked by each interceptor prior to running the actual constraint tests.  If the method call returns null, the constraint is applied, otherwise the result is returned.  The same logic can be invoked independently, using the `@BeforeAccess` annotation, in which case the call to `beforeRoleCheck` itself becomes the constraint.  Or, to say it in code, 

    if (deadboltHandler.beforeAuthCheck(ctx) == null)
    {
        result = delegate.call(ctx);
    }

##### Scope
`@BeforeAccess` can be used at the class or method level.

##### Parameters

|Parameter                |Type                              |Notes                                             |
|-------------------------|----------------------------------|--------------------------------------------------|
| handler                 | Class<? extends DeadboltHandler> | The **class** of a DeadboltHandler to use  in    |
|                         |                                  | in place of the default one.                     |
|-------------------------|----------------------------------|--------------------------------------------------|
| alwaysExecute           | Boolean                          | By default, if another Deadbolt action has       |
|                         |                                  | already been executed in the same request and has|
|                         |                                  | allowed access, `beforeAuthCheck` will not be    |
|                         |                                  | executed again.  Set this to true if you want it |
|                         |                                  | to execute unconditionally.                      | 
|-------------------------|----------------------------------|--------------------------------------------------|
| deferred                | Boolean                          | If true, the interceptor will not be applied     |
|                         |                                  | until a DeadboltDeferred annotation is           |
|                         |                                  | encountered.                                     |

## Customising the inputs of annotation-driven actions
One of the problems with D2's annotations is they require strings to specify, for example, role names or pattern values.  It would be far safer to use enums, but this is not possible for a module - it would completely kill the generic applicability of the annotations.  If D2 shipped with an enum containing roles, how would you extend it?  You would be stuck with whatever was specified, or forced to fork the codebase and customise it.  Similarly, annotations can neither implement, extend or be extended.

To address this situation, D2 has four constraints whose inputs can be customised to some degree.  The trick lies, not with inheritence, but delegation and wrapping.  The constraints are

 * Restrict
 * Restrictions
 * Dynamic
 * Pattern

Here, I'll explain how to customise the Restrict constraint to use enums as annotation parameters, but the principle is the same for each constraint.

To start, create an enum that represents your roles.

    public enum MyRoles implements Role {
        foo,
        bar,
        hurdy;

        @Override
        public String getRoleName() {
            return name();
        }
    }

Next, create a new annotation to drive your custom version of Restrict.  Note that an array of `MyRoles` values can be placed in the annotation.  The standard `Restrict` annotation is also present to provide further configuration.  This means your customisations are minimised. 

    @With(CustomRestrictAction.class)
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD, ElementType.TYPE})
    @Documented
    @Inherited
    public @interface CustomRestrict {
        MyRoles[] value();

        Restrict config();
    }

The code above contains `@With(CustomRestrictAction.class)` in order to specify the action that should be triggered by the annotation.  This action can be implemented as follows. 

    @Override
    public Result call(Http.Context context) throws Throwable {

        final CustomRestrict outerConfig = configuration;
        RestrictAction restrictAction = new RestrictAction(configuration.config(),
                                                           this.delegate)
        {
            @Override
            public String[] getRoleNames()
            {
                List<String> roleNames = new ArrayList<String>();
                for (MyRoles role : outerConfig.value())
                {
                    roleNames.add(role.getRoleName());
                }
                return roleNames.toArray(new String[roleNames.size()]);
            }
        };
        return restrictAction.call(context);
    }

Pretty simple stuff - the code is basically converting an array of enum values to an array of strings.

To use your custom annotation, you apply it as you would any other D2 annotation.

    @CustomRestrict(value = {MyRoles.foo, MyRoles.bar}, config = @Restrict(""))
    public static Result customRestrictOne() {
        return ok(accessOk.render());
    }

Each customisable action has one or more extension points.  These are

|Action class       |Extension points                  |
|-------------------|----------------------------------|
|RestrictAction     | * String[] getRoleNames()        |
|-------------------|----------------------------------|
|RestrictionsAction | * List<String[]> getRoleGroups() |
|-------------------|----------------------------------|
|Dynamic            | * String getValue()              |
|                   | * String getMeta()               | 
|-------------------|----------------------------------|
|Pattern            | * String getValue()              |