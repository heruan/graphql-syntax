# graphql-syntax
A catalog of different packages and syntaxes to generate a GraphQL-JS schema

**Table of Contents**

- [gestalt](#gestalt)
- [graphql-helpers](#graphql-helpers)
- [graphql-tools](#graphql-tools)
- [json-to-graphql](#json-to-graphql)
- [modelizr](#modelizr)
- [mongoose-schema-to-graphql](#mongoose-schema-to-graphql)
- [ts2gql](#ts2gql)


## [gestalt](https://github.com/charlieschwabacher/gestalt)

### Outline

Gestalt will generate a schema with database backed resolution based on type definitions written
in the GraphQL schema definition language.  The schema language is a
[proposed](https://github.com/facebook/graphql/pull/90) addition to the GraphQL spec and
is already used in the [documentation](http://graphql.org/docs/typesystem/).

Gestalt has an interface for pluggable database adapters, but at the moment the only adapter is
for PostgresQL.

### Example

The following snippet of GraphQL would be all that it takes to define a working API and database
schema for a twitter like app.

```graphql
type User implements Node {
  email: String! @hidden @unique
  passwordHash String! @hidden
  firstName: String
  lastName: String
  posts: Post @relationship(path: "=AUTHORED=>")
  followedUsers: User @relationship(path: "=FOLLOWED=>")
  followers: User @relationship(path: "<=FOLLOWED=")
  feed: Post @relationship(path: "=FOLLOWED=>User=AUTHORED=>")
}
type Post implements Node {
  text: String
  author: User @relationship(path: "<-AUTHORED-")
}
```

The `@hidden` directive on `email` and `passwordHash` marks fields that should result in database
columns, but not be exposed in the GraphQL schema.  The `@unique` column on `email` results in a
uniqueness constraint in the database.

The `@relationship` directives  provide the information needed to create a database schema
and efficient queries for resolution.  The syntax of the path strings is inspired by
[Neo4j](//github.com/neo4j/neo4j)'s Cypher query language.

This arrow syntax has three parts - the label `AUTHORED`, the direction of
the arrow `in` or `out`, and a cardinality ('singular' or 'plural') based on the
`-` or `=` characters.

Arrows with identical labels and types at their head and tail are matched, and
the combination of their cardinalities determines how the relationship between
their types will be stored in the database.

For the schema above, Gestalt is able to tell that it should add a `authored_by_user_id`
column to the `posts` table for the `User AUTHORED Post` relationship, and a `user_followed_user`
join table for the `User FOLLOWED User` relationship.

For the plural relationships, gestalt will also create relay connection types `UsersConnection`
and `PostsConnection`, and update the `posts`, `followedUsers`, `followers`, and `feed`
fields.

We can serve our API like this:

```
import gestaltServer from 'gestalt-server';
import gestaltPostgres from 'gestalt-postgres';

const app = express();

app.use('/graphql', gestaltServer({
  schemaPath: `${__dirname}/schema.graphql`,
  database: gestaltPostgres({
    databaseURL: 'postgres://localhost'
  }),
  objects: [], // this array would include any custom resolution definitions
  mutations: [], // this array would include mutation definitions
  secret: '!',
}));

app.listen(3000);
```

Or get a GraphQLSchema object like this:

```
import fs from 'fs';
import gestaltGraphQL from 'gestalt-graphql';
import gestaltPostgres from 'gestalt-postgres';

const schemaText = fs.readFileSync(`${__dirname}/schema.graphql`);

const {schema} = gestaltGraphQL(
  schemaText,
  [], // this array would include any custom resolution definitions for objects
  [], // this array would include mutation definitions
  gestaltPostgres({
    databaseURL: 'postgres://localhost',
  }),
);
```

If we wanted a `fullName` field on `User` that concatenated a user's first and last names, we
could add it to the schema like so.

```graphql
type User implements Node {
  firstName: String
  lastName: String
  fullName: String @virtual
}
```

The `@virtual` directive will prevent it from resulting in a database column, and we could
create an object defining resolution like this:

```javascript
export default {
  name: 'User',
  fields: {
    // calculate user's first name from first and last names
    fullName: obj => `${obj.firstName} ${obj.lastName}`,
  },
}
```


## [json-to-graphql](https://github.com/Aweary/json-to-graphql)

### Outline

Generate a GraphQL schema file from any JSON data. json-to-graphql parses the JSON you pass to it, generating a schema file that matches both the structure and types of your JSON fields.

It can parse whether a field is nullable, create custom GraphQL types, and parse arrays into `GraphQLList` instance. It also supports deeply nested custom types. If you pass it an array of JSON objects it will iterate through all of them and check for type consistency.

The API is simple. Just import the `generateSchema` file and pass your JSON data to it. It returns a string, so you can write the file anywhere you like. You can see a full example [here](https://github.com/Aweary/json-to-graphql#example).

```js
import generateSchema from 'json-to-graphql'
import data from './data.json'

const schema = generateSchema(data)
fs.writeFile('schema.js', schema, callback)
```

### Example

The snippet below uses a simplified version of the response returned from [Github's repo API](https://api.github.com/repos/aweary/json-to-graphql). All we have to do is take our JSON data
and pass it to `generateSchema` which is the default export of the json-to-graphql package.

```js
import generateSchema from 'json-to-graphql'

var data = {
  "id": 61638820,
  "name": "json-to-graphql",
  "full_name": "Aweary/json-to-graphql",
  "owner": {
    "login": "Aweary",
    "id": 6886061,
    "avatar_url": "https://avatars.githubusercontent.com/u/6886061?v=3",
  },
  "private": false,
  "html_url": "https://github.com/Aweary/json-to-graphql",
  "description": "Create GraphQL schema from JSON files and APIs",
  "fork": false,
  }

var schema = generateSchema(data)
console.log(schema)

```
After running that snippet we should see the schema logged out in the console. It correctly parses field types and generates custom types. All that's left to the user is implementing the `resolve` functions for each field.

```js
const {
    GraphQLBoolean,
    GraphQLString,
    GraphQLInt,
    GraphQLFloat,
    GraphQLObjectType,
    GraphQLSchema,
    GraphQLID,
    GraphQLNonNull
} = require('graphql')


const OwnerType = new GraphQLObjectType({
    name: 'owner',
    fields: {
        login: {
            description: 'enter your description',
            type: new GraphQLNonNull(GraphQLString),
            // TODO: Implement resolver for login
            resolve: () => null,
        },
        id: {
            description: 'enter your description',
            type: new GraphQLNonNull(GraphQLID),
            // TODO: Implement resolver for id
            resolve: () => null,
        },
        avatar_url: {
            description: 'enter your description',
            type: new GraphQLNonNull(GraphQLString),
            // TODO: Implement resolver for avatar_url
            resolve: () => null,
        }
    },
});


module.exports = new GraphQLSchema({
    query: new GraphQLObjectType({
        name: 'RootQueryType',
        fields: () => ({
            id: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLID),
                // TODO: Implement resolver for id
                resolve: () => null,
            },
            name: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for name
                resolve: () => null,
            },
            full_name: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for full_name
                resolve: () => null,
            },
            login: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for login
                resolve: () => null,
            },
            id: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLID),
                // TODO: Implement resolver for id
                resolve: () => null,
            },
            avatar_url: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for avatar_url
                resolve: () => null,
            },
            owner: {
                description: 'enter your description',
                type: new GraphQLNonNull(OwnerType),
                // TODO: Implement resolver for owner
                resolve: () => null,
            },
            private: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLBoolean),
                // TODO: Implement resolver for private
                resolve: () => null,
            },
            html_url: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for html_url
                resolve: () => null,
            },
            description: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLString),
                // TODO: Implement resolver for description
                resolve: () => null,
            },
            fork: {
                description: 'enter your description',
                type: new GraphQLNonNull(GraphQLBoolean),
                // TODO: Implement resolver for fork
                resolve: () => null,
            }
        })
    })
})
```


## [graphql-helpers](https://github.com/depop/graphql-helpers)

### Outline

Build individual GraphQL types from the familiar GraphQL Definition Language, whilst co-locating your resolve functions with the types. Designed so that it can be adopted incrementally. Provides a small set of generic resolver functions to cover common use cases.

#### Example

Visit the project repository for more thorough examples.

```javascript
import { GraphQLSchema } from 'graphql';
import { Registry } from 'graphql-helpers';

const registry = new Registry();

registry.create(`
  type BlogEntry {
    id: ID!
    title: String
    slug: String
    content: String
  }
`;

registry.create(`
  type Query {
    blogEntry(slug: String!): BlogEntry
    blogEntries: [BlogEntry]
  }
`, {
  blogEntry: /* resolver */,
  blogEntries: /* resolver */,
};

const schema = new GraphQLSchema({
  query: registry.getType('Query'),
});
```

## [modelizr](https://github.com/julienvincent/modelizr)

A Combination of normalizr, json-schema-faker and GraphQL that allows you to define multipurpose models that can generate GraphQL queries, mock deeply nested data and normalize


### What can I use this for?

+ Easily generating GraphQL queries from models.
+ Flat-mapping responses using [normalizr](https://github.com/gaearon/normalizr).
+ Mocking deeply nested data that match their query.

___

Read my [medium post](https://medium.com/@julienvincent/modelizr-99e59c1c4431#.applec5ut) on why I wrote modelizr.

### What does it look like?

```javascript
import { model, query } from 'modelizr'

const user = model('users', {
    id: {type: 'primary|integer', alias: 'ID'},
    firstName: {type: 'string', faker: 'name.firstName'}
})

const book = model('books', {
    id: {type: 'primary|integer'},
    title: {type: 'string'}
})
user.define({books: [book]})
book.define({author: user})

query(
    user(
        book()
    ),

    book(
        user().as('author')
    ).params(ids: [1, 2, 3])
)
    .path("http:// ... ")
    .then((res, normalize) => {
        normalize(res.body) // -> normalized response.
    })
```
This will generate the following query and make a request using it.
```
{
  users {
     id: ID,
     firstName,
     books {
        id,
        title
     }
  },
  books(ids: [1, 2, 3]) {
     id,
     title,
     author {
        id: ID,
        firstName
     }
  }
}
```

### Documentation

All documentation is located at [julienvincent.github.io/modelizr](http://julienvincent.github.io/modelizr)

## [ts2gql](https://github.com/convoyinc/ts2gql)

Converts a collection of TypeScript interfaces into GraphQL's IDL.

### What can I use this for?

+ Easily generating your GraphQL server's schema from an existing TypeScript type hierarchy.
+ Keeping your GraphQL schema in sync with your TypeScript interfaces.

### What does it look like?

`input.ts`
```ts
/** @graphql ID */
export type Id = string;

export type Url = string;

export interface User {
  id: Id;
  name: string;
  photo: Url;
}

export interface PostContent {
  title: string;
  body: string;
}

export interface Post extends PostContent {
  id: Id;
  postedAt: Date;
  author: User;
}

export interface Category {
  id: Id;
  name: string;
  posts: Post[];
}

export interface QueryRoot {
  users(args: {id: Id}): User[]
  posts(args: {id: Id, authorId: Id, categoryId: Id}): Post[]
  categories(args: {id: Id}): Category[]
}

/** @graphql schema */
export interface Schema {
  query: QueryRoot;
}
```

```
> ts2gql input.ts

scalar Date

scalar Url

type User {
  id: ID
  name: String
  photo: Url
}

interface PostContent {
  body: String
  title: String
}

type Post {
  author: User
  body: String
  id: ID
  postedAt: Date
  title: String
}

type Category {
  id: ID
  name: String
  posts(authorId: ID): [Post]
}

type QueryRoot {
  categories(id: ID): [Category]
  posts(id: ID, authorId: ID, categoryId: ID): [Post]
  users(id: ID): [User]
}

schema {
  query: QueryRoot
}

```

## [graphql-tools](https://github.com/apollostack/graphql-tools)

Documentation: [docs.apollostack.com](http://docs.apollostack.com/graphql-tools/).

Tutorial for mocking: [medium.com/apollo-stack](https://medium.com/apollo-stack/mocking-your-server-with-just-one-line-of-code-692feda6e9cd)

GraphQL Tools provides a set of useful tools for manipulating GraphQL.js schemas and mocking GraphQL servers. The main component is a set of functions that let you quickly create an executable GraphQL schema from GraphQL schema language. It also contains functionality that makes standing up a mock GraphQL server as easy as writing a GraphQL schema.

### Example usage

This code will create a server which returns a mock person when queried.

All of the functionality demonstrated here can also be used separately (check out the examples in the documentation).
```js
import express from 'express';
import { graphqlHTTP } from 'express-graphql';
import { makeExecutableSchema, addMockFunctionsToSchema } from 'graphql-tools';

const Schema = `
 type Person {
   id: ID!
   name: String
   age: Int
 }

 type Query {
   findPerson(id: ID!): Person
 }

 schema {
   query: Query
 }
`;

const Resolvers = {
  Query: {
    findPerson(root, { id }){
      // here you would write the code that returns an actual person object
      // but since we're using mocks, you don't have to.
    },
  },
};

// if given an empty object, the default mock functions will be used.
// for how to override the default mocks, see the documentation or tutorial.
const Mocks = {};

const GRAPHQL_PORT = 3000;
const graphQLServer = express();

const executableSchema = makeExecutableSchema({
  typeDefs: Schema,
  resolvers: Resolvers,
});

addMockFunctionsToSchema({
  schema: executableSchema,
  mocks: Mocks,
  preserveResolvers: false,
});

graphQLServer.use('/graphql', graphqlHTTP({
  schema: executableSchema,
}));

graphQLServer.listen(GRAPHQL_PORT);
```


## [mongoose-schema-to-graphql](https://github.com/sarkistlt/mongoose-schema-to-graphql)

### Outline
Full support of all graphQL "Definitions" and "Scalars" besides "GraphQLFloat", because in Mongoose schema you can use only int. numbers. But you can use ```props``` property to pass it, details below. 

### What can I use this for?
This package will help you  avoid typing schemas for same essence.
If you already have Mongoose schema that's enough to generate graphQL schema.

#### Example
First:
~~~shell
npm i --save mongoose-schema-to-graphql
~~~
Make sure that your ```graphql``` package is the same version as used in ```mongoose-schema-to-graphql``` or vice versa.

Then:
~~~js
import MTGQL from 'mongoose-schema-to-graphql';
~~~

MTGQL function accept obj as argument with following structure:
~~~js
let configs = {
              name: 'couponType', //graphQL type's name
              description: 'Coupon base schema', //graphQL type's description
              class: 'GraphQLObjectType', //"definitions" class name
              schema: couponSchema, //your Mongoose schema "let couponSchema = mongoose.Schema({...})"
              exclude: ['_id'], //fields which you want to exclude from mongoose schema
              props: {
                price: {type: GraphQLFloat}
              } //add custom properties or overwrite existed
          }
~~~

After you declared config. object you're ready to go. Examples below:
~~~js
dbSchemas.js

export let couponSchema = mongoose.Schema({
    couponCode: Array,
    description: String,
    discountType: String,
    discountAmount: String,
    minimumAmount: String,
    singleUseOnly: Boolean,
    createdAt: mongoose.Schema.Types.Date,
    updatedAt: mongoose.Schema.Types.Date,
    expirationDate: mongoose.Schema.Types.Date,
    isMassPromo: Boolean,
    couponBatchId: String,
    maximumAmount: String,
    isPublished: Boolean
});
~~~

~~~js
import MTGQL from 'mongoose-schema-to-graphql';
import {couponSchema} from './dbSchemas';

let configs = {
              name: 'couponType',
              description: 'Coupon schema',
              class: 'GraphQLObjectType',
              schema: couponSchema,
              exclude: ['_id']
          }
          
export let couponType = MTGQL(configs);
~~~
 
It will be equal to:
~~~js
import {...} from 'graphql';
import MTGQL from 'mongoose-schema-to-graphql';
import {couponSchema} from './dbSchemas';

export let couponType = new GraphQLObjectType({
    name: 'couponType',
    description: 'Coupon schema',
    fields: {
        couponCode: {type: new GraphQLList(GraphQLString)},
        description: {type: GraphQLString},
        discountType: {type: GraphQLString},
        discountAmount: {type: GraphQLString},
        minimumAmount: {type: GraphQLString},
        singleUseOnly: {type: GraphQLBoolean},
        createdAt: {type: GraphQLString},
        updatedAt: {type: GraphQLString},
        expirationDate: {type: GraphQLString},
        isMassPromo: {type: GraphQLBoolean},
        couponBatchId: {type: GraphQLString},
        maximumAmount: {type: GraphQLString},
        isPublished: {type: GraphQLBoolean}
    }
});
~~~

Note: If you pass mongoose type ```Array``` it will be converted to ```{type: new GraphQLList(GraphQLString)}```
If you want to create a list of another type, you would need to declare it in Mongoose schema too:
~~~js
let quizSchema = mongoose.Schema({
    message: String,
    createdAt: mongoose.Schema.Types.Date,
    updatedAt: mongoose.Schema.Types.Date
});

let customerSchema = mongoose.Schema({
    createdAt: mongoose.Schema.Types.Date,
    updatedAt: mongoose.Schema.Types.Date,
    firstName: String,
    lastName: String,
    email: String,
    quiz: [quizSchema],
    subscription: {
        status: String,
        plan: String,
        products: Array
    }
});
~~~

Then: 
~~~js
import MTGQL from 'mongoose-schema-to-graphql';
import {customerSchema} from './dbSchemas';

let configs = {
              name: 'customerType',
              description: 'Customer schema',
              class: 'GraphQLObjectType',
              schema: customerSchema,
              exclude: ['_id']
          }
          
export let couponType = MTGQL(configs);
~~~

It's equal to:
~~~js
import {...} from 'graphql';
import MTGQL from 'mongoose-schema-to-graphql';
import {customerSchema} from './dbSchemas';

let quizType = new GraphQLObjectType({
    name: 'quizType',
    description: 'quiz type for customer',
    fields: {
        message: {type: GraphQLString},
        updatedAt: {type: GraphQLString},
        createdAt: {type: GraphQLString}
    }
});

export let customerType = new GraphQLObjectType({
    name: 'customerType',
    description: 'Customer schema',
    fields: {
        createdAt: {type: GraphQLString},
        updatedAt: {type: GraphQLString},
        firstName: {type: GraphQLString},
        lastName: {type: GraphQLString},
        email: {type: GraphQLString},
        quiz: {type: new GraphQLList(quizType)},
        subscription: {
            type: new GraphQLObjectType({
                name: 'subscription',
                fields: () => ({
                    status: {type: GraphQLString},
                    plan: {type: GraphQLString},
                    products: {type: new GraphQLList(GraphQLString)}
                })
            })
        }
    }
});
~~~

###```props``` property in config object.
You can use this field to pass some additional props. to graphQL schema, for example:
~~~js
import {GraphQLFloat} from 'graphql';
import MTGQL from 'mongoose-schema-to-graphql';
import {customerSchema} from './dbSchemas';

let configs = {
              name: 'customerType',
              description: 'Customer schema',
              class: 'GraphQLObjectType',
              schema: customerSchema,
              exclude: ['_id'],
              props: {
                price: {type: GraphQLFloat}
              }
          }
          
export let couponType = MTGQL(configs);
~~~

It's equal to:
~~~js
import {...} from 'graphql';
import MTGQL from 'mongoose-schema-to-graphql';
import {customerSchema} from './dbSchemas';

let quizType = new GraphQLObjectType({
    name: 'quizType',
    description: 'quiz type for customer',
    fields: {
        message: {type: GraphQLString},
        updatedAt: {type: GraphQLString},
        createdAt: {type: GraphQLString}
    }
});

export let customerType = new GraphQLObjectType({
    name: 'customerType',
    description: 'Customer schema',
    fields: {
        price: {type: GraphQLFloat},
        createdAt: {type: GraphQLString},
        updatedAt: {type: GraphQLString},
        firstName: {type: GraphQLString},
        lastName: {type: GraphQLString},
        email: {type: GraphQLString},
        quiz: {type: new GraphQLList(quizType)},
        subscription: {
            type: new GraphQLObjectType({
                name: 'subscription',
                fields: () => ({
                    status: {type: GraphQLString},
                    plan: {type: GraphQLString},
                    products: {type: new GraphQLList(GraphQLString)}
                })
            })
        }
    }
});
~~~

If passed props. already exist in Mongoose schema, for example ```price: Number``` it will be overwrite with prop. we passed in config. object. 
