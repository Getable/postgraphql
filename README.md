# Getable Fork of PostGraphQL v1.7.0

We decided to table implementation of GraphQL because we didn't have an immediate need for it. If you want to continue setting it up for marketable, here are some problems you'll have to solve:

## TODO
  - [ ] API authentication/authorization. A short-term solution would be to whitelist the tables that we don't mind exposing publicly (i.e. any that doesn't refer to a person) where PostgraphQL creates the schema. The longer-term solution would be to take advantage of PostgraphQL's JSON Web Token authorization system, which would require us to create database users, roles, and permissions in Postgres. See [PostGraphQL's JWT spec](https://github.com/calebmer/postgraphql/blob/master/docs/pg-jwt-spec.md). Another possibility is to continue updating this postgraphql fork and do authorization checks with graphQL (potentially helpful article on node-level permissions [here](https://medium.com/apollo-stack/auth-in-graphql-part-2-c6441bcc4302#.3iz2n34f8)).
  - [ ] Set up ApolloClient in marketable. A partial implementation can be found on the branch called `apollo`. It's only set up for  development mode on marketable and we don't yet have a working page example (though you can see `page-type` for a skeleton).
  - [ ] Spin up a Heroku app to host GraphQL. Rather than using a Procfile we can update `scripts/starts.sh` to start the server, pointing at our database using creds in environment variables. Don't forget to add `?ssl=true` at the end of the db connection string
  - [ ] Enabling [procedures](https://github.com/calebmer/postgraphql/blob/master/docs/procedures.md) is a nice-to-have if we want to use the full power of graphQL. We added an option to disable procedures in this fork, because we found that postgis procedures were breaking a GraphQL constraint that all types need to have a unique name. We talked about replacing all our location queries with Algolia's geo search so we could get rid of the postgis extension but decided that was way too big of a project.

## PostGraphQL README

[![Package on npm](https://img.shields.io/npm/v/postgraphql.svg?style=flat)](https://www.npmjs.com/package/postgraphql)
[![Gitter chat room](https://badges.gitter.im/calebmer/postgraphql.svg)](https://gitter.im/calebmer/postgraphql?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

*A GraphQL schema created by reflection over a PostgreSQL schema.*

The strongly typed GraphQL data querying language is a revolutionary new way to interact with your server. Similar to how JSON very quickly overtook XML, GraphQL will likely take over REST. Why? Because GraphQL allows us to express our data in the exact same way we think about it.

The PostgreSQL database is the self acclaimed “world’s most advanced open source database” and even after 20 years that still rings true. PostgreSQL is the most feature rich SQL database available and provides an excellent public reflection API giving its users a lot of power over their data. And despite being 20 years old, the database still has frequent releases.

With PostGraphQL, you can access the power of PostgreSQL through a well designed GraphQL server. PostGraphQL uses PostgreSQL reflection APIs to automatically detect primary keys, relationships, types, comments, and more providing a GraphQL server that is highly intelligent about your data.

PostGraphQL holds a fundamental belief that a *well designed database schema should be all you need to serve well thought out APIs*. PostgreSQL already has amazing user management and relationship infrastructure, *why duplicate that logic* in a custom API? PostGraphQL is likely to provide a more performant and standards compliant GraphQL API then any created in house. Focus on your product and let PostGraphQL manage how the data gets to the product.

For a critical evaluation of PostGraphQL to determine if it fits in your tech stack, read the [evaluating PostGraphQL for your project](#evaluating-postgraphql-for-your-project) section.

## Usage
First install using npm:

```bash
npm install -g postgraphql
```

and then just run it!

```bash
postgraphql postgres://localhost:5432/mydb --schema forum --development
```

For more information run:

```bash
postgraphql --help
```

Check out the **[forum example][]** for a demo of PostGraphQL in action.

[forum example]: https://github.com/calebmer/postgraphql/tree/master/examples/forum

## Benefits
PostGraphQL uses the joint benefits of PostgreSQL and GraphQL to provide a number of key benefits.

### Automatic Relation Detection
Does your table’s `authorId` column reference another table? PostGraphQL knows and will give you a field for easily querying that reference.

A schema like:

```sql
create table post (
  id serial primary key,
  author_id int non null references user(id),
  headline text,
  body text,
  …
);
```

Can query relations like so:

```graphql
{
  postNodes {
    nodes {
      headline
      body
      author: userByAuthorId {
        name
      }
    }
  }
}
```

### Custom Mutations and Computed Columns
Procedures in PostgreSQL are powerful for writing business logic in your database schema, and PostGraphQL allows you to access those procedures through a GraphQL interface. Create a custom mutation, write an advanced SQL query, or even extend your tables with computed columns! Procedures allow you to write logic for your app in SQL instead of in the client all while being accessible through the GraphQL interface.

So a search query could be written like this:

```sql
create function search_posts(search text) returns setof post as $$
  select *
  from post
  where
    headline ilike ('%' || search || '%') or
    body ilike ('%' || search || '%')
$$ language sql stable;
```

And queried through GraphQL like this:

```graphql
{
  searchPosts(search: "Hello world", first: 5) {
    pageInfo {
      hasNextPage
    }
    totalCount
    nodes {
      headline
      body
    }
  }
}
```

For more information, check out our [procedure documentation][] and our [advanced queries documentation][].

[procedure documentation]: https://github.com/calebmer/postgraphql/blob/master/docs/procedures.md
[advanced queries documentation]: https://github.com/calebmer/postgraphql/blob/master/docs/advanced-queries.md

### Fully Documented APIs
Introspection of a GraphQL schema is powerful for developer tooling and one element of introspection is that every type in GraphQL has an associated `description` field. As PostgreSQL allows you to document your database objects, naturally PostGraphQL exposes these documentation comments through GraphQL.

Documenting PostgreSQL objects with the [`COMMENT`][sql-comment] command like so:

```sql
create table user (
  username text non null unique,
  …
);

comment on table user is 'A human user of the forum.';
comment on column user.username is 'A unique name selected by the user to represent them on our site.';
```

Will let you reflect on the schema and get the JSON below:

```graphql
{
  __type(name: "User") { … }
}
```

```json
{
  "__type": {
    "name": "User",
    "description": "A human user of the forum.",
    "fields": {
      "username": {
        "name": "username",
        "description": "A unique name selected by the user to represent them on our site."
      }
    }
  }
}
```

[sql-comment]: http://www.postgresql.org/docs/current/static/sql-comment.html

### UI For Development Comes Standard
[GraphiQL][graphiql] is a great tool by Facebook to let you interactively explore your data. When development mode is enabled in PostGraphQL, the GraphiQL interface will be *automatically* displayed at your GraphQL endpoint.

Just navigate with your browser to the URL printed to your console after starting PostGraphQL and use GraphiQL with your data! Even if you don’t want to use GraphQL in your app, this is a great interface for working with any PostgreSQL database.

Just remember to use the `--development` flag when starting PostGraphQL!

[graphiql]: https://github.com/graphql/graphiql

### Token Based Authorization
PostGraphQL let’s you use token based authentication with [JSON Web Tokens][jwt] (JWT) to secure your API. It doesn’t make sense to redefine your authentication in the API layer, instead just put your authorization logic in the database schema! With an advanced [grants][grants] system and [row level security][row-level-security], authorization in PostgreSQL is more than enough for your needs.

PostGraphQL follows the [PostgreSQL JSON Web Token Serialization Specification][pg-jwt-spec] for serializing JWTs to the database for your use in authorization. The `role` claim of your JWT will become your PostgreSQL role and all other claims can be found under the `jwt.claims` namespace (see [retrieving claims in PostgreSQL][retrieving-claims]).

To enable token based authorization use the `--secret <string>` command line argument with a secure string PostGraphQL will use to sign and verify tokens. And if you don’t want authorization, just don’t set the `--secret` argument and PostGraphQL will ignore all authorization information!

[jwt]: https://jwt.io
[grants]: http://www.postgresql.org/docs/current/static/sql-grant.html
[row-level-security]: http://www.postgresql.org/docs/current/static/ddl-rowsecurity.html
[pg-jwt-spec]: https://github.com/calebmer/postgraphql/blob/master/docs/pg-jwt-spec.md
[retrieving-claims]: https://github.com/calebmer/postgraphql/blob/master/docs/pg-jwt-spec.md#retrieving-claims-in-postgresql

### Cursor Based Pagination For Free
There are some problems with traditional limit/offset pagination and realtime data. For more information on such problems, read [this article][pagination-for-graphql].

PostGraphQL not only provides limit/offset pagination, but it also provides cursor based pagination ordering on the column of your choice. Never again skip an item with free cursor based pagination!

[pagination-for-graphql]: https://medium.com/apollo-stack/understanding-pagination-rest-graphql-and-relay-b10f835549e7#.8ehp4qwsq

### Relay Specification Compliant
You don’t have to use GraphQL with React and Relay, but if you are, PostGraphQL implements the brilliant Relay specifications for GraphQL. Even if you are not using Relay your project will benefit from having these strong, well thought out specifications implemented by PostGraphQL.

The specific specs PostGraphQL implements are:

- [Global Object Identification Specification.](http://facebook.github.io/relay/graphql/objectidentification.htm)
- [Cursor Connections Specification.](http://facebook.github.io/relay/graphql/connections.htm)
- [Input Object Mutations Specification.](http://facebook.github.io/relay/graphql/mutations.htm)

* * *

## Roadmap
In the future, things PostGraphQL will be able to do will include:

- Authentication using PostgreSQL roles.
- Whitelisted queries in production.
- Subscriptions using the PostgreSQL [`NOTIFY`][pg-notify] feature.

## Evaluating PostGraphQL For Your Project
Hopefully you’ve been convinced that PostGraphQL serves an awesome GraphQL API, but now let’s take a more critical look at whether or not you should adopt PostGraphQL for your project.

PostGraphQL’s audience is for people who’s core business is not the API and want to prioritize their product. PostGraphQL allows you to define your content model in the database as you normally would, however instead of building the bindings to the database (your API) PostGraphQL takes care of it.

This takes a huge maintenance burden of your shoulders. Now you don’t have to worry about optimizing the API and the database, instead you can focus on just optimizing your database.

### No Lock In
PostGraphQL does not lock you into using PostGraphQL forever. Its purpose is to help your business in a transitory period. When you feel comfortable with the cost of building your API PostGraphQL is simple to switch with a custom solution.

How is it simple? Well first of all, your PostgreSQL schema is still your PostgreSQL schema. PostGraphQL does not ask you to do anything too divergent with your schema allowing you to take your schema (and all your data) to whatever solution you build next. GraphQL itself provides a simple and clear deprecation path if you want to start using different fields.

Ideally PostGraphQL scales with your company and we would appreciate your contributions to help it scale, however there is a simple exit path even years into the business.

### Schema Driven APIs
If you fundamentally disagree with a one-to-one mapping of a SQL schema to an API (GraphQL or otherwise) this section is for you. First of all, PostGraphQL is not necessarily meant to be a permanent solution. PostGraphQL was created to allow you to focus on your product and not the API. If you want a custom API there is a simple transition path (read [no lock in](#no-lock-in)). If you still can’t get over the one-to-one nature of PostGraphQL consider the following arguments why putting your business logic in PostgreSQL is a good idea:

1. PostgreSQL already has a powerful [user management system][user-management] with fine grained [row level security][row-level-security]. A custom API would mean you have to build your own user management and security.
2. PostgreSQL allows you to hide implementation details with [views][pg-views] of your data. Simple views can even be [auto-updatable][pg-udpatable-views]. This provides you with the same freedom and flexibility as you might want from a custom API except more performant.
3. PostgreSQL gives you automatic relations with the `REFERENCES` constraint and PostGraphQL automatically detects them. With a custom API, you’d need to hardcode these relationships.
4. For what it’s worth, you can write in your favorite scripting language in PostgreSQL including [JavaScript][js-in-pg] and [Ruby][ruby-in-pg].
5. And if you don’t want to write your Ruby in PostgreSQL, you could also use PostgreSQL’s [`NOTIFY`][pg-notify] feature to fire events to a listening Ruby or [JavaScript][node-pg-notify] microservice can listen and respond to PostgreSQL events (this could include email transactions and event reporting).

Still worried about a certain aspect of a schema driven API? Open an issue, confident we can convince you otherwise 😉

[user-management]: http://www.postgresql.org/docs/current/static/user-manag.html
[row-level-security]: http://www.postgresql.org/docs/current/static/ddl-rowsecurity.html
[pg-views]: http://www.postgresql.org/docs/current/static/sql-createview.html
[pg-udpatable-views]: http://www.postgresql.org/docs/current/static/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS
[js-in-pg]: https://blog.heroku.com/archives/2013/6/5/javascript_in_your_postgres
[ruby-in-pg]: https://github.com/knu/postgresql-plruby
[pg-notify]: http://www.postgresql.org/docs/current/static/sql-notify.html
[node-pg-notify]: https://www.npmjs.com/package/pg-pubsub

* * *

## Thanks
Thanks so much to the people working on [PostgREST](https://github.com/begriffs/postgrest) which was definetly a huge inspiration for this project! The core contributors are awesome and taught me so much 😊. Ironically, they also convinced me that GraphQL was superior to REST…

Enjoy this server? Want updates and seeing what I’m up to next? Follow me on Twitter [`@calebmer`](https://twitter.com/calebmer).

Thanks and enjoy 👍
