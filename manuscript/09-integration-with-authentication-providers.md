-# Integrating with authentication providers

Authorization is all well and good, but some constraints are pointless if the user is not known to the application.  You could write a `Dynamic` constraint that only allows access on Thursdays - this doesn't need to know anything about even the concept of a user; a `Restrict` constraint, on the other hand, uses `Roles` obtained from a `Subject`.  The question is, what is a subject and how do we who it is?
  
Imagine an application where you can post short messages and read the messages of others, something along the lines of Twitter.  By default, you can read any message on the system unless the user has marked that message as private - these private messages require you to be logged in to view them.  In order to write a message, you have to have an account and be logged in.  We can sketch out a controller for this very simple application thus:


{title="An application with Deadbolt constraints", lang=java}
~~~~~~~
package be.objectify.messages;

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
    	                .map(json -> Json.bind(Message.class, json))
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

This is a good time to review the difference between the HTTP status codes `401 Unauthorized` and `403 Forbidden`.  A 401 means you don't have access *at the moment*, but try again after authenticating.   A 403 means the subject cannot access the resource with their current authorization rights, and re-authenticating will not solve the problem.  A well-behaved application should respect the difference between the two.

We can consider the `onAccessFailure` method to be the Deadbolt equivalent of a 403.  For a `DeadboltHandler` used by a RESTful controller, the status code should be enough to indicate the problem.  If you have an application that uses server-side rendering, you may well want to return content in the body of the response.  The end result is the same though - You Can't Do That.  Note the return type - the `Promise` contains a `Result` - access has very definitely failed at this point, and it needs to be dealt with.


{title="Handling access failure", lang=java}
~~~~~~~
public F.Promise<Result> onAccessFailure(final Http.Context context,
                                         final String content) {
    return F.Promise.promise(Results::forbidden);
}
~~~~~~~


To manage the 401 use-case, we use the `beforeAuthCheck` method.  Unlike `onAccessFailure`, this method allows for the possibility that everything is fine.  It's perfectly reasonable to return an empty option from this method.

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
    return getSubject().map(subject -> subject.map(Optional::empty).orElseGet(() -> Optional.of(Results::unauthorized)));
}
~~~~~~~

When we look at specific authenication providers, we will focus on the `beforeAuthCheck` as the integration 