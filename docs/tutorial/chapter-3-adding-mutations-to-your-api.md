# Chapter 3 <br> Adding Mutations to Your API {docsify-ignore}

In this chapter you're going to add some write capability to your API. You'll learn about:

- Writing GraphQL mutations
- Exposing GraphQL objects for mutation operations
- Working with GraphQL Context
- Working with GraphQL arguments

To keep our learning gradual we'll stick to in-memory data for now but rest assured a proper databases is coming in an upcoming chapter.

## Wire Up The Context

The first thing we'll do is setup an in-memory database and expose it to our resolvers using the _GraphQL context_.

The GraphQL Context is a plain JavaScript object shared across all resolvers. Nexus creates a new one for each request and adds a few of its own properties. Largely though, what it will contains will be defined by your app. It is a good place to, for example, attach information about the logged-in user.

So go ahead and create the database.

```bash
touch api/db.ts
```

```ts
// api/db.ts

export const db = {
  posts: [{ id: 1, title: 'Nexus', body: '...', published: false }],
}
```

Now to expose it in our GraphQL context we'll use a new schema method called `addToContext`. We can do this anywhere in our app but a fine place is the `api/app.ts` module we already created in chapter 1.

```ts
// api/app.ts

import { schema } from 'nexus'
import { db } from './db'

schema.addToContext(() => {
  return {
    db,
  }
})
```

That's it. Behind the scenes Nexus will use the TypeScript compiler API to extract our return type here and propagate it to the parts of our app where the context is accessible. And if ever this process does not work for you for some reason you can use fallback to manually giving the types to Nexus like so:

```ts
module global {
  interface NexusContext {
    // type information here
  }
}
```

> **Note** For those familiar with GraphQL, you might be grimacing that we’re attaching static things to the context, instead of using export/import.
> This is a matter of convenience. Feel free to take a purer approach in your apps if you want.

## Use The Context

Now let's use this data to reimplement the `Query.drafts` resolver from the previous chapter.

```diff
schema.queryType({
  type: 'Query',
  definition(t) {
    t.list.field('drafts', {
      type: 'Post',
-     resolve() {
-       return [{ id: 1, title: 'Nexus', body: '...', published: false }]
+     resolve(_root, _args, ctx) {                             // 1
+       return ctx.db.posts.filter(p => p.published === false)  // 2
      },
    })
  },
})
```

1. Context is the _third_ parameter, usually identified as `ctx`
2. Return the filtered data by un-published posts, _aka_ drafts . Nexus makes sure the types line up.

**Did you notice?** Still no TypeScript type annotations required from you yet everything is still totally type safe. Prove it to yourself by hovering over the `ctx.db.posts` property and witness the correct type information it gives you. This is the type propagation we just mentioned in action. 🙌

## Your First Mutation

Alright, now that we know how to wire things into our context, let's implement our first mutation. We're going to make it possible for your API clients to create new drafts.

This mutation will need a name. Rather than simply call it `createPost` we'll use language from our domain. In this case `createDraft` seems reasonable. There are similarities with our previous work with `Query.drafts`:

- `Mutation` is a root type, its fields are entrypoints.
- We can colocate mutation fields with the objects they relate to or centralize all mutation fields.

As before we will take the colocation approach.

<div class="TightRow">

<!-- prettier-ignore -->
```ts
// api/graphql/Post.ts
// ...

schema.extendType({
  type: 'Mutation',
  definition(t) {
    t.field('createDraft', {
      type: 'Post',
      nullable: false,              // 1
      resolve(_root, args, ctx) {
        ctx.db.posts.push(/*...*/)
        return // ...
      },
    })
  },
})
```

```graphql
Mutation {
  createDraft: Post!
}
```

</div>

1. By default in Nexus all output types are nullable. This is for [best practice reasons](https://graphql.org/learn/best-practices/#nullability). In this case, we want to guarantee [to the client] that a Post object will always be returned upon a successful signup mutation.

   If you're ever dissatisfied with Nexus' defaults, not to worry, [you can change them globally](https://www.nexusjs.org/#/api/modules/main/exports/settings?id=schemanullableinputs).

We need to get the client's input data to complete our resolver. This brings us to a new concept, GraphQL arguments. Every field in GraphQL may accept them. Effectively you can think of each field in GraphQL like a function, accepting some input, doing something, and returning an output. Most of the time "doing something" is a matter of some read-like operation but with `Mutation` fields the "doing something" usually entails a process with side-effects (e.g. writing to the databse).

Let's revise our implementation with GraphQL arguments.

<div class="IntrinsicRow">

```diff
schema.extendType({
  type: 'Mutation',
  definition(t) {
    t.field('createDraft', {
      type: 'Post',
+     args: {                                        // 1
+       title: schema.stringArg({ required: true }), // 2
+       body: schema.stringArg({ required: true }),  // 2
+     },
      resolve(_root, args, ctx) {
+       const draft = {
+         id: ctx.db.posts.length + 1,
+         title: args.title,                         // 3
+         body: args.body,                           // 3
+         published: false,
+       }
+       ctx.db.posts.push(draft)
+       return draft
-       ctx.db.posts.push(/*...*/)
-       return // ...
      },
    })
  },
})
```

```diff
Mutation {
-  createDraft: Post
+  createDraft(title: String!, body: String!): Post
}
```

</div>

1. Add an `args` property to the field definition to define its args. Keys are arg names and values are type specifications.
2. Use the Nexus helpers for defining an arg type. There is one such helper for every GraphQL scalar such as `schema.intArg` and `schema.booleanArg`. If you want to reference a type like some InputObject then use `schema.arg({ type: "..." })`.
3. In our resolver, access the args we specified above and pass them through to our custom logic. If you hover over the `args` parameter you'll see that Nexus has properly typed them including the fact that they cannot be undefined.

## Model The Domain pt 2

Before we wrap this chapter let's flush out our schema a bit more. We'll add a `publish` mutation to transform a draft into an actual published post, then we'll let API clients read the the published posts.

<div class="IntrinsicRow">

```ts
// api/graphql/Post.ts

import { schema } from 'nexus'

schema.extendType({
  type: 'Mutation',
  definition(t) {
    // ...
    t.field('publish', {
      type: 'Post',
      args: {
        draftId: schema.intArg({ required: true }),
      },
      resolve(_root, args, ctx) {
        let draftToPublish = ctx.inDb.posts.find((p) => p.id === args.draftId)

        if (!draftToPublish) {
          throw new Error('Could not find draft with id ' + args.draftId)
        }

        draftToPublish.published = true

        return draftToPublish
      },
    })
  },
})
```

```diff
type Mutation {
  createDraft(body: String!, title: String!): Post
+ publish(draftId: Int!): Post
}
```

</div>

<div class="IntrinsicRow">

```ts
// api/graphql/Post.ts

import { schema } from 'nexus'

schema.extendType({
  type: 'Query',
  definition(t) {
    // ...
    t.list.field('posts', {
      type: 'Post',
      resolve(_root, _args, ctx) {
        return ctx.inDb.posts.filter((p) => p.published === true)
      },
    })
  },
})
```

```diff
type Query {
  drafts: [Post!]
+ posts: [Post!]
}
```

</div>

## Try It Out

Great, now head on over to the GraphQL Playground and run this query (left). If everything went well, you should see a response like this (right):

<div class="TightRow">

```graphql
mutation {
  publish(draftId: 1) {
    id
    title
    body
    published
  }
}
```

```json
{
  "data": {
    "publish": {
      "id": 1,
      "title": "Nexus",
      "body": "...",
      "published": true
    }
  }
}
```

</div>

Now, that published draft should be visible via the `posts` Query. Run this query (left) and expect the following response (right):

<div class="TightRow">

```graphql
query {
  posts {
    id
    title
    body
    published
  }
}
```

```json
{
  "data": {
    "posts": [
      {
        "id": 1,
        "title": "Nexus",
        "body": "...",
        "published": true
      }
    ]
  }
}
```

</div>

## Wrapping Up

Congratulations! You can now read and write to your API.

But, so far you've been validating your work by manually interacting with the Playground. That may be reasonable at first (depending on your relationship to TDD) but it will not scale. At some point you are going to want automated testing. Nexus takes testing seriously and in the next chapter we'll show you how. See you there!

<div class="NextIs NextChapter"></div>

[➳](/tutorial/chapter-4-testing-your-api)
