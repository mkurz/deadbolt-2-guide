
Internally, calls to `getSubject` made via `be.objectify.deadbolt.java.cache.SubjectCache`; when `deadbolt.java.cache-user` is set to true, this cache is used to store the subject in and retrieve the subject from `Http.Context#args`.  The subject is not eagerly cached - the caching occurs on the first retrieval of the subject.


The standard implementation of this cache is `be.objectify.deadbolt.java.cache.DefaultSubjectCache`.  It caches the subject returned from `getSubject` directly, which may not be desirable if, for example, the subject is a database proxy entity. If you want to dissociate the subject from any baggage it happens to be carrying, you can override the caching method to return a copy of the subject.


You can either use your existing implementations of `Subject`, `Role` and `Permission` or create lightweight classes such as this.


    public class LightweightSubject implements Subject
    {
        private final List<? extends Role> roles;
        private final List<? extends Permission> permissions;
        private final String identifier;

        private LightweightSubject(final Builder builder)
        {
            roles = builder.roles;
            permissions = builder.permissions;
            identifier = builder.identifier;
        }

        @Override
        public List<? extends Role> getRoles()
        {
            return roles;
        }

        @Override
        public List<? extends Permission> getPermissions()
        {
            return permissions;
        }

        @Override
        public String getIdentifier()
        {
            return identifier;
        }

        public static final class Builder
        {
            private final List<Role> roles = new LinkedList<>();
            private final List<Permission> permissions = new LinkedList<>();
            private String identifier;

            public Builder role(Role role)
            {
                this.roles.add(role);
                return this;
            }

            public Builder permission(final Permission permission)
            {
                this.permissions.add(permission);
                return this;
            }

            public Builder identifier(final String identifier)
            {
                this.identifier = identifier;
                return this;
            }

            public LightweightSubject build()
            {
                return new LightweightSubject(this);
            }
        }
    }


Now you need to put it to use, taking the raw values such as `identifier` and `role.name` and using them to create a new representation of the subject.

    public class CopyOnCacheSubjectCache extends DefaultSubjectCache
    {
        @Inject
        public DefaultSubjectCache(final Configuration configuration)
        {
          super(configuration);
        }

        public Optional<Subject> cacheSubject(final Optional<Subject> subjectOption,
                                              final Http.Context context)
        {
            return subjectOption.map(subject -> {
                final LightweightSubject.Builder builder = new LightweightSubject.Builder()
                                                                                 .identifier(subject.getIdentifier());
                subject.getRoles()
                       .parallelStream()
                       .map(role -> builder.role(new LightweightRole(role.getName())));
                subject.getPermissions()
                       .parallelStream()
                       .map(permission -> builder.permission(new LightweightPermission(permission.getValue())));

                final Subject copy = builder.build();
                context.args.put(ConfigKeys.CACHE_DEADBOLT_USER,
                                 copy);
                return copy;
            });
        }
    }

As you'll see below, a small module is used to bind your hooks into Deadbolt.  You can use this module to override the default `SubjectCache`, substituting your own instead.

        public class CustomDeadboltHook extends Module {
            @Override
            public Seq<Binding<?>> bindings(final Environment environment,
                                            final Configuration configuration) {
                return seq(bind(HandlerCache.class).to(MyHandlerCache.class).in(Singleton.class));
            }
        }

