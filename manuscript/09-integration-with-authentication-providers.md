# Integrating with authentication providers

Authorization is all well and good, but some constraints are pointless if the user is not known to the application.  You could write a `Dynamic` constraint that only allows access on Thursdays - this doesn't need to know anything about even the concept of a user; a `Restrict` constraint, on the other hand, uses `Roles` obtained from a `Subject`.  The question is, what is a subject and how do we know who it is?
  
Imagine an application where you can post short messages and read the messages of others, something along the lines of Twitter.  By default, you can read any message on the system unless the user has marked that message as restricted to only users who are logged into the application..  In order to write a message, you have to have an account and be logged in.  We can sketch out a controller for this very simple application thus:


{title="A controller with Deadbolt constraints", lang=java}
~~~~~~~
package be.objectify.messages;

import javax.inject.Inject;
import play.libs.F;
import play.libs.Json;
import play.mvc.Controller;
import play.mvc.Result;
import be.objectify.deadbolt.java.actions.SubjectPresent;

public class Messages extends Controller {

    private final MessageDao messageDao;

    @Inject
    public MessagesController(final MessageDao messageDao) {
        this.messageDao = messageDao;
    }

    public F.Promise<Result> getPublicMessages() {
        return F.Promise.promise(messageDao::getAllPublic)
                        .map(Json::toJson)
                        .map(Results::ok);
    }

    @SubjectPresent
    public F.Promise<Result> getAllMessages() {
        return F.Promise.promise(messageDao::getAll)
                        .map(Json::toJson)
                        .map(Results::ok);
    }

    @SubjectPresent
    public F.Promise<Result> createMessage() {
        return F.Promise.promise(() -> body.asJson())
                        .map(json -> Json.fromJson(json, Message.class))
                        .map(messageDao::save)
                        .map(Results::ok);
    }
}
~~~~~~~

This very simple app has a very simple `routes` file to go with it:

{title="The application routes"}
~~~~~~~
GET    /messages/public   be.objectify.messages.Messages.getPublicMessages()
GET    /messages          be.objectify.messages.Messages.getAllMessages()
POST   /create            be.objectify.messages.Messages.createMessage()
~~~~~~~

An authenticated user can access all three of these routes and obtain a successful result.  An unauthenticated user would hit a brick wall when accessing `/messages` or `/create` - specifically, they would run into whatever behaviour was specified in the `onAccessFailure` method of the current `DeadboltHandler`.

This is a good time to review the difference between the HTTP status codes `401 Unauthorized` and `403 Forbidden`.  A 401 means you don't have access *at the moment*, but you should try again after authenticating.   A 403 means the subject cannot access the resource with their current authorization rights, and re-authenticating will not solve the problem - in fact, the specification explicitly states that you shouldn't even attempt to re-authenticate.  A well-behaved application should respect the difference between the two.

We can consider the `onAccessFailure` method to be the Deadbolt equivalent of a 403.  For a `DeadboltHandler` used by a RESTful controller, the status code should be enough to indicate the problem.  If you have an application that uses server-side rendering, you may well want to return content in the body of the response.  The end result is the same though - You Can't Do That.  Note the return type, a `Promise` containing a `Result` and not an `Optional<Result>` - access has very definitely failed at this point, and it needs to be dealt with.


{title="Handling access failure", lang=java}
~~~~~~~
public F.Promise<Result> onAccessFailure(final Http.Context context,
                                         final String content) {
    return F.Promise.promise(Results::forbidden);
}
~~~~~~~


To manage the 401 use-case, we use the `beforeAuthCheck` method.  Unlike `onAccessFailure`, this method allows for the possibility that everything is fine.  It's perfectly reasonable to return an empty option from this method, indicating that no action should be taken and the request should continue unimpeded.

{title="Doing nothing before a constraint is applied", lang=java}
~~~~~~~
public F.Promise<Optional<Result>> beforeAuthCheck(final Http.Context context) {
    return F.Promise.promise(Optional::empty);
}
~~~~~~~

This has the net effect of not getting in the way of the constraint; with this implementation, a user accessing `/messages` in our little application would receive a 403.  But, our application aims to be well behaved and return a 401 when it's needed, so a little more work is required.  Because `beforeAuthCheck` is only called when a constraint has been applied, we can use this to trigger an authenication request if needed.  In this application, we're going to say that every constraint requires an authenticated subject to be present - do not confuse this with `@SubjectPresent` constraints in the example controller, the same would equally be true if we were using `@Restrict` or `@Pattern`.  For the more advanced app that uses the subject-less dynamic rule of Thursdays-only, either more logic is required or (preferably) a different `DeadboltHandler` implementation is used.

The logic here is simple - if a user is present, no action is required otherwise short-cut the request with a 401 response.

{title="Requiring a subject", lang=java}
~~~~~~~
public F.Promise<Optional<Result>> beforeAuthCheck(final Http.Context context) {
    return getSubject(context).map(maybeSubject -> 
        maybeSubject.map(subject -> Optional.<Result>empty())
                    .orElseGet(() -> Optional.of(Results.unauthorized())));
}
~~~~~~~

When we look at specific authenication providers, we will focus on the `beforeAuthCheck` as the integration.  It will become very clear, very quickly, the same approach is used for all authentication systems; this means that swapping out authentication without affecting authorization is both possible and trivial.


## Play's built-in authentication support

Play ships with a very simple interceptor that requires an authenticated user to be present; there's no concept of authorization.  It uses an annotation-driven approach similar to that of Deadbolt, allowing you to annotate either entire controllers at the class level, or individual methods within a controller.

{title="Using Play's authentication support", lang=java}
~~~~~~~
@Security.Authenticated
public F.Promise<Result> getAllMessages() {
    return F.Promise.promise(messageDao::getAll)
                    .map(Json::toJson)
                    .map(Results::ok);
}
~~~~~~~

By default, the `Security.Authenticated` annotation will trigger an interceptor that uses `Security.Authenticator` to look in the session for a value mapped to `"username"` - if the value is non-null, the user is considered to be authenticated.  If you want to customize how the user identification string is obtained, you can extend `Security.Authenticator` implement your own solution.

{title="Customizing Play's authentication support", lang=java}
~~~~~~~
package be.objectify.messages.security;

import java.util.Optional;
import javax.inject.Inject;
import be.objectify.messages.dao.UserDao;
import be.objectify.messages.models.User;
import play.mvc.Http;
import play.mvc.Security;

public class AuthenticationSupport extends Security.Authenticator {

    private final UserDao userDao;

    @Inject
    public AuthenticationSupport(final UserDao userDao) {
        this.userDao = userDao;
    }
    
    @Override
    public String getUsername(final Http.Context context) {
        return getTokenFromHeader(context).flatMap(userDao::findByToken)
                                          .map(User::getIdentifier)
                                          .orElse(null);
    }

    private Optional<String> getTokenFromHeader(final Http.Context context) {
        return Optional.ofNullable(context.request().headers().get("X-AUTH-TOKEN"))
                       .filter(arr -> arr.length == 1)
                       .filter(arr -> arr[0] != null)
                       .map(arr -> arr[0]);
    }
}
~~~~~~~

The class of this customized implementation can then be passed to the annotation with `@Security.Authenticated(AuthenticationSupport.class)`.  However, all mention of Deadbolt's constraints have vanished and so we've replaced a fine-grained authorization system with a coarse-grained authentication-only system.  To fix this, we need to revert back to using Deadbolt in the controller and move `AuthenticationSupport` (or even the basic `Security.Authenticator`) integration into the `DeadboltHandler`.

{title="Integrating Play's authentication with Deadbolt", lang=java}
~~~~~~~
package be.objectify.messages.security;

import java.util.Optional;
import javax.inject.Inject;
import be.objectify.deadbolt.core.models.Subject;
import be.objectify.deadbolt.java.AbstractDeadboltHandler;
import be.objectify.messages.dao.UserDao;
import play.libs.F;
import play.mvc.Http;
import play.mvc.Result;
import play.mvc.Results;

public class MyDeadboltHandler extends AbstractDeadboltHandler {

    private final AuthenticationSupport authenticator;
    private final UserDao userDao;

    @Inject
    public MyDeadboltHandler(final AuthenticationSupport authenticator,
                             final UserDao userDao) {
        this.authenticator = authenticator;
        this.userDao = userDao;
    }

    @Override
    public F.Promise<Optional<Result>> beforeAuthCheck(final Http.Context context) {
        return getSubject(context).map(maybeSubject -> 
            maybeSubject.map(subject -> Optional.<Result>empty())
                        .orElseGet(() -> Optional.of(Results.unauthorized())));
    }

    @Override
    public F.Promise<Optional<Subject>> getSubject(final Http.Context context) {
        return F.Promise.promise(() -> 
            Optional.ofNullable(authenticator.getUsername(context))
                    .flatMap(userDao::findByUserName)
                    .map(user -> user));
    }

    @Override
    public F.Promise<Result> onAuthFailure(final Http.Context context,
                                           final String content) {
        // you could also use the behaviour of the authenticator, e.g.
        // return F.Promise.promise(() -> authenticator.onUnauthorized(context));
        return F.Promise.promise(() -> unauthorized("You can't do that"));
    }
}
~~~~~~~

There are three things going on here, only one of which is explicitly tied into the authentication system, and that is `getSubject`.  Of the other two methods, `onAuthFailure` gives an arbitrary (but hopefully meaningful) response and `beforeAuthCheck` is essentially generic code.  There is scope here for performance improvements and will depend on your specific implementations; for example, the user retrieved from the database by the authenticator can be stored in `context.args` for re-use by the Deadbolt handler.

{title="Per-request caching of the user", lang=java}
~~~~~~~
public class AuthenticationSupport extends Security.Authenticator {

    // other methods

    @Override
    public String getUsername(final Http.Context context) {
        final Optional<User> maybeUser = getTokenFromHeader(context).flatMap(userDao::findByToken);
        return maybeUser.map(user -> {
            context.args.put("user", maybeUser);
            return user.getIdentifier();
        }).orElse(null);
    }
}

public class MyDeadboltHandler extends AbstractDeadboltHandler {

    // other methods

    @Override
    public F.Promise<Optional<Subject>> getSubject(final Http.Context context) {
        return F.Promise.promise(() -> (Optional<Subject>)context.args.computeIfAbsent(
                "user",
                key -> {
                    final String userName = authenticator.getUsername(context);
                    return userDao.findByUserName(userName);
                }));
    }
}
~~~~~~~
