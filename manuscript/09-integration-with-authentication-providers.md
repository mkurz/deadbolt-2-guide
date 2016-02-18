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

An authenticated user can access all three of these routes and obtain a successful result.  An unauthenticated user would hit a brick wall when accessing `/messages` or `/create` - specifically, they would run into whatever behaviour was specified in the `onAuthFailure` method of the current `DeadboltHandler`.

This is a good time to review the difference between the HTTP status codes `401 Unauthorized` and `403 Forbidden`.  A 401 means you don't have access *at the moment*, but you should try again after authenticating.   A 403 means the subject cannot access the resource with their current authorization rights, and re-authenticating will not solve the problem - in fact, the specification explicitly states that you shouldn't even attempt to re-authenticate.  A well-behaved application should respect the difference between the two.

We can consider the `onAuthFailure` method to be the Deadbolt equivalent of a brick wall.  For a `DeadboltHandler` used by a RESTful controller, the status code should be enough to indicate the problem.  If you have an application that uses server-side rendering, you may well want to return content in the body of the response.  The end result is the same though - You Can't Do That.  Note the return type, a `Promise` containing a `Result` and not an `Optional<Result>` - access has very definitely failed at this point, and it needs to be dealt with.


{title="Handling access failure", lang=java}
~~~~~~~
public F.Promise<Result> onAuthFailure(final Http.Context context,
                                       final String content) {
    return F.Promise.promise(Results::forbidden);
}
~~~~~~~


Unlike `onAuthFailure`, the `beforeAuthCheck` method allows for the possibility that everything is fine.  It's perfectly reasonable to return an empty option from this method, indicating that no action should be taken and the request should continue unimpeded.

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

On the other hand, you may choose to never implement any logic in `beforeAuthCheck` and instead have the behaviour driven by authorization failure.  The choice is entirely in the hands of the implementor; personally, I tend to use `onAuthFailure` to handle the 401/403 behaviour, because it removes the assumptions required by implementing checks in `beforeAuthCheck`.

It will become very clear, very quickly, the same approach is used for all authentication systems; this means that swapping out authentication without affecting authorization is both possible and trivial.  We'll start with the built-in authentication mechanism of Play and adapt from there; with surprisingly few changes, you'll see how we can move from basic authentication through to using anything from Play-specific OAuth libraries such as [Play Authenticate](https://joscha.github.io/play-authenticate/) and even HTTP calls to dedicated identity platforms such as [Auth0](https://auth0.com/). 

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
        return F.Promise.promise(() -> forbidden("You can't do that"));
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

## Third-party user management

I like the concept of third-party user (or identity) management.  It minimizes sensitive data held locally, provides features like multifactor authentication and provides a unified API for multiple authentication sources.  This last feature makes it very easy to create a simple integration point with Deadbolt, driving interaction with the user management on a per-event basis (with a little caching thrown in).  The sequence for this looks a little complicated, but it can be broken down into 4 distinct steps:

* the initial authentication
* subsequent use of cached user details
* re-retrieving user details when the cache doen't contain them
* re-authenticating when the user's authentication has expired on the user management platform

If you're using Deadbolt, it's reasonable to assume you have one of two security models - either all actions require authorization or some actions require authorization, and that authorization may simply be "a user must be present".  As a result, the point the initial authentication occurs depends on your application.  The good news, as far as this requirement goes, is the implementation is both quite simple and common to all cases.  It comes down to the implementation of the `onAuthFailure` method of your `DeadboltHandler`, and might look something like this:

{title="Triggering authentication when authorization fails - naive version", lang=java}
~~~~~~~
public F.Promise<Result> onAuthFailure(final Http.Context context,
                                       final String contentType) {
    return F.Promise.promise(login::render)
                    .map(Results::unauthorized);
}
~~~~~~~

But wait, this is wrong!  As discussed above, this assumes that all authorization failure occurs because there is no user present, and this ignores the difference between `401 Unauthorized` and `403 Forbidden`.  A better implementation wiil take this into account, by checking if there is a subject present.  If there is a subject present, it's a 403; if there isn't, it's a 401 and we can redirect to somewhere the user can log in.


{title="Triggering authentication when authorization fails", lang=java}
~~~~~~~
public F.Promise<Result> onAuthFailure(final Http.Context context,
                                       final String contentType) {
    return getSubject(context).map(maybeSubject -> 
        maybeSubject.map(subject -> 
            Optional.of((User)subject))
                    .map(denied::render)
                    .orElseGet(() -> login.render()))
                    .map(Results::unauthorized);
}
~~~~~~~

There's still a problem here - while the rendered output observes the difference between unauthorized and forbidden, but the HTTP status code is hard-wired to a 401.  One more tweak should fix this.

{title="Synchronizing the content and HTTP status code", lang=java}
~~~~~~~
@Override
public F.Promise<Result> onAuthFailure(final Http.Context context,
                                       final String s) {
    return getSubject(context)
            .map(maybeSubject ->
                         maybeSubject.map(subject -> 
                                              Optional.of((User)subject))
                                     .map(user -> 
                                              new F.Tuple<>(true,
                                                            denied.render(user)))
                                     .orElseGet(() -> 
                                              new F.Tuple<>(false,
                                                            login.render(clientId,
                                                                         domain,
                                                                         redirectUri))))
            .map(subjectPresentAndContent -> 
                     subjectPresentAndContent._1
                         ? Results.forbidden(subjectPresentAndContent._2)
                         : Results.unauthorized(subjectPresentAndContent._2));
}
~~~~~~~


### Integrating with Auth0
[Auth0](https://auth0.com/) is a great identity management platform, and I'm not writing that just because they gave me a t-shirt.  One of the nice features they offer is a whole bunch of code you can pretty much drop into your application, including code for a Play 2 Scala controller.   Since we're in the Java portion of the book, and the Java example provided by Auth0 uses the JEE Servlet API, I've rewritten the Scala version for Java.  This was the only customization required, which I have to say was pretty impressive - the total time to integrate and have a working solution was less than 15 minutes.

There are three core elements to the solution.  These are, in no particular order,

* a log-in page
* a controller to receive callbacks from Auth0
* a DeadboltHandler implementation

I've also added a small utility class called `AuthSupport` to help with the cache usage, which also makes testing easier, but this contains code that could happily live in the controller.

A working example for this section can be found at [auth0-integration](https://github.com/schaloner/deadbolt-2-guide-examples/tree/master/auth0-integration).  To run it, you will need to create an application on Auth0 and fill in the client ID, client secret, etc, into `conf/application.conf`.  For the redirect URI, you can use `http://localhost:9000/callback` - don't forget to adjust the port if necessary.

##### AuthSupport
This class has two simple function - it standardises the key used for caching the subject, and it wraps the cache result in an `Optional`. 

{title="Integrating the authentication flow", lang=java}
~~~~~~~
package be.objectify.whale.security;

import java.util.Optional;
import javax.inject.Inject;
import javax.inject.Singleton;
import be.objectify.whale.models.User;
import play.cache.CacheApi;
import play.mvc.Http;

@Singleton
public class AuthSupport {

    private final CacheApi cache;

    @Inject
    public AuthSupport(final CacheApi cache) {
        this.cache = cache;
    }

    public Optional<User> currentUser(final Http.Context context) {
        return Optional.ofNullable(cache.get(cacheKey(context.session().get("idToken"))));
    }

    public String cacheKey(final String key) {
        return "user.cache." + key;
    }
}

~~~~~~~


##### The log-in page
In order to log in, Auth0 provide a JavaScript solution that customises the form based on your configuration options; for example, an app registered in Auth0 for username/password support plus a couple of OAuth providers will receive a form that reflects those choices.  The simplest possible implementation of a log-in page, without concern for appearance, is as follows.

{title="An Auth0 log-in form", lang=html}
~~~~~~~
@(clientId: String, domain: String, redirectUri: String)

<!DOCTYPE html>
<html lang="en">
    <body>
        <div id="root">
            Log-in area
        </div>
        <script src="https://cdn.auth0.com/js/lock-7.12.min.js"></script>
        <script>
            var lock = new Auth0Lock('@clientId', '@domain');
            lock.show({
                container: 'root',
                callbackURL: '@redirectUri',
                responseType: 'code',
                authParams: { scope: 'openid profile' }
            });
        </script>
    </body>
</html>
~~~~~~~

In the browser, you now have a completely functional log-in form that will trigger the authentication flow.  Once the form is submitted, Auth0 takes over until the authentication requirements are satisfied and then we receive a callback.

![An Auth0 log-in form]()

##### The controller
The bulk of the logic is contained here, and this code is reasonably generic - barring the `User` and `UserDao` classes, this code can be used in any Play 2 application.  In broad terms, three things happen during a successful authentication flow - all of these are rooted in the `callback` method of the controller.

1. The controller receives a callback, in the form of a HTTP request from Auth0 containing an authorization code.
2. The controller makes a HTTP request to Auth0 to get the token.
3. The controller makes a HTTP request to Auth0 to get the user details.

{title="Processing the callback from Auth0", lang=java}
~~~~~~~
public F.Promise<Result> callback(final F.Option<String> maybeCode,
                                  final F.Option<String> maybeState) {
    return maybeCode.map(code -> getToken(code) // get the authentication token
             .flatMap(token -> getUser(token))  // get the user details
             .map(userAndToken -> {
                 // userAndToken._1 is the user
                 // userAndToken._2 is the token
                 cache.set(authSupport.cacheKey(userAndToken._2._1),
                           userAndToken._1,
                           60 * 15); // cache the subject for 15 minutes
                 session("idToken",
                         userAndToken._2._1);
                 session("accessToken",
                         userAndToken._2._2);
                 return redirect(routes.Application.index());
             }))
            .getOrElse(F.Promise.pure(badRequest("No parameters supplied")));
}
~~~~~~~

This callback provides the starting point for further interaction with Auth0 by giving us an authorization code.  With this code, we can request token information; `access_token` allows us to work with the subject's attributes, and ´id_token´ is a signed Json Web Token used to authenticate API calls.

{title="Get the token information", lang=java}
~~~~~~~
private F.Promise<F.Tuple<String, String>> getToken(final String code) {
    final ObjectNode root = Json.newObject();
    root.put("client_id",
             this.clientId);
    root.put("client_secret",
             this.clientSecret);
    root.put("redirect_uri",
             this.redirectUri);
    root.put("code",
             code);
    root.put("grant_type",
             "authorization_code");
    return WS.url(String.format("https://%s/oauth/token",
                                this.domain))
             .setHeader(Http.HeaderNames.ACCEPT,
                        Http.MimeTypes.JSON)
             .post(root)
             .map(WSResponse::asJson)
             .map(json -> new F.Tuple<>(json.get("id_token").asText(),
                                        json.get("access_token").asText()));
}
~~~~~~~

With these token data, we can retrieve the subject attributes.  At this point, it's possible to cache the subject to reduce network calls.  In this example, we have no concept of a database and so we rely entirely on Auth0 to provide subject information.  If you keep some user information local, this might be a good place to either create or retrieve that information.

{title="Get the subject attributes", lang=java}
~~~~~~~
private F.Promise<F.Tuple<User, F.Tuple<String, String>>>
                        getUser(final F.Tuple<String, String> token) {
    return WS.url(String.format("https://%s/userinfo",
                                this.domain))
             .setQueryParameter("access_token",
                                token._2)
             .get()
             .map(WSResponse::asJson)
             .map(json -> new User(json.get("user_id").asText(),
                                   json.get("name").asText(),
                                   json.get("picture").asText()))
             .map(localUser -> new F.Tuple<>(localUser,
                                             token));
}
~~~~~~~

Logging out simply requires the token information to be removed from the session, and the removal of the subject from the cache.

{title="Log the subject out", lang=java}
~~~~~~~
public F.Promise<Result> logOut() {
    return F.Promise.promise(() -> {
        final Http.Session session = session();
        final String idToken = session.remove("idToken");
        session.remove("accessToken");
        cache.remove(authSupport.cacheKey(idToken));
        return "ignoreThisValue";
    }).map(id -> redirect(routes.AuthController.logIn()));
}
~~~~~~~

This is a lot of code, but authentication is now handled.  We now have a way to log in, and a way to retrieve the user details from Auth0.  This controller needs to be exposed in the routes file, and this also provides a nice overview to see what we've achieved.

{title="Authentication controller routes"}
~~~~~~~
GET  /logIn     be.objectify.whale.controllers.AuthController.logIn()
GET  /callback  be.objectify.whale.controllers.AuthController.callback(code: play.libs.F.Option[String], state: play.libs.F.Option[String])
GET  /logOut    be.objectify.whale.controllers.AuthController.logOut()
GET  /denied    be.objectify.whale.controllers.AuthController.denied()
~~~~~~~

Now we have a `/logIn` route, that means you can have an explicit link to log in from your application.  The one thing remaining to do is to have the log-in view displayed automatically when authorization fails.

##### The DeadboltHandler
There are only two methods that are required for this example to work.  `getSubject` will retrieve the subject from the cache, and `onAuthFailure` will handle things as discussed above.

{title="Authentication controller routes", lang=java}
~~~~~~~
@Override
public F.Promise<Optional<Subject>> getSubject(final Http.Context context) {
    return F.Promise.promise(() -> Optional.ofNullable(cache.get(authSupport.cacheKey(context.session().get("idToken")))));
}

@Override
public F.Promise<Result> onAuthFailure(final Http.Context context,
                                       final String s) {
    return getSubject(context)
            .map(maybeSubject ->
                         maybeSubject.map(subject -> Optional.of((User)subject))
                                     .map(user -> 
                                          new F.Tuple<>(true,
                                          denied.render(user)))
                                     .orElseGet(() -> 
                                          new F.Tuple<>(false,
                                                        login.render(clientId,
                                                                     domain,
                                                                     redirectUri))))
            .map(subjectPresentAndContent -> 
                     subjectPresentAndContent._1
                         ? Results.forbidden(subjectPresentAndContent._2)
                         : Results.unauthorized(subjectPresentAndContent._2));
}
~~~~~~~

#### Improvements
This is a very simple example, but it demonstrates how easily it is to use event-driven behaviour and third-party identity management.  There is one major problem, however, and you have until the end of this sentence to figure out what it is.

When the subject attributes are retrieved from Auth0, the resulting `User` object is cached for an arbitrary time - 15 minutes, in this case.  With the implementation of `DeadboltHandler` given above, once that 15 minutes have passed the user will need to re-authenticate.  However, it's possible their authenticate period on Auth0 is still valid and so we're placing an unnecessary burden on the end user.  A simple improvement would be to attempt retrieval of the user attributes from `DeadboltHandler#getSubject` when the cache doesn't contain the user.

It's also possible to store meta data in Auth0, and so you can represent your roles and permissions there and bind them into local models when retrieving the subject's attributes.
