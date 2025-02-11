# dgraph-js [![npm version](https://img.shields.io/npm/v/dgraph-js.svg?style=flat)](https://www.npmjs.com/package/dgraph-js) [![Build Status](https://img.shields.io/travis/dgraph-io/dgraph-js/master.svg?style=flat)](https://travis-ci.org/dgraph-io/dgraph-js) [![Coverage Status](https://img.shields.io/coveralls/github/dgraph-io/dgraph-js/master.svg?style=flat)](https://coveralls.io/github/dgraph-io/dgraph-js?branch=master)

Official Dgraph client implementation for JavaScript (Node.js v6 and above),
using [gRPC].

**Looking for browser support? Check out [dgraph-js-http].**

[grpc]: https://grpc.io/
[dgraph-js-http]: https://github.com/dgraph-io/dgraph-js-http

This client follows the [Dgraph Go client][goclient] closely.

[goclient]: https://github.com/dgraph-io/dgo

Before using this client, we highly recommend that you go through [docs.dgraph.io],
and understand how to run and work with Dgraph.

[docs.dgraph.io]:https://docs.dgraph.io

## Table of contents

- [Install](#install)
- [Quickstart](#quickstart)
- [Using a Client](#using-a-client)
  - [Creating a Client](#creating-a-client)
  - [Altering the Database](#altering-the-database)
  - [Creating a Transaction](#creating-a-transaction)
  - [Running a Mutation](#running-a-mutation)
  - [Running a Query](#running-a-query)
  - [Running an Upsert: Query + Mutation](#running-an-upsert-query--mutation)
  - [Running a Conditional Upsert](#running-a-conditional-upsert)
  - [Committing a Transaction](#committing-a-transaction)
  - [Cleanup Resources](#cleanup-resources)
  - [Debug mode](#debug-mode)
- [Examples](#examples)
- [Development](#development)
  - [Building the source](#building-the-source)
  - [Running tests](#running-tests)

## Install

Install using npm:

```sh
npm install dgraph-js grpc --save
# If you are using Typescript, you might also need:
# npm install @types/google-protobuf @types/protobufjs --save-dev
```

or yarn:

```sh
yarn add dgraph-js grpc
# If you are using Typescript, you might also need:
# yarn add @types/google-protobuf @types/protobufjs --dev
```

## Quickstart

Build and run the [simple][] project in the `examples` folder, which
contains an end-to-end example of using the Dgraph JavaScript client. Follow the
instructions in the README of that project.

Depending on the version of Dgraph that you are connecting to, you will have to
use a different version of this client.

| Dgraph version | dgraph-js version |
|:--------------:|:-----------------:|
|     1.0.X      |      *1.X.Y*      |
|     1.1.X      |      *2.X.Y*      |

Note: Only API breakage from *v1.X.Y* to *v2.X.Y* is in
the function `DgraphClient.newTxn().mutate()`. This function returns a `messages.Assigned`
type in *v1.X* but a `messages.Response` type in *v2.X*.

## Using a Client

### Creating a Client

A `DgraphClient` object can be initialised by passing it a list of
`DgraphClientStub` clients as variadic arguments. Connecting to multiple Dgraph
servers in the same cluster allows for better distribution of workload.

The following code snippet shows just one connection.

```js
const dgraph = require("dgraph-js");
const grpc = require("grpc");

const clientStub = new dgraph.DgraphClientStub(
  // addr: optional, default: "localhost:9080"
  "localhost:9080",
  // credentials: optional, default: grpc.credentials.createInsecure()
  grpc.credentials.createInsecure(),
);
const dgraphClient = new dgraph.DgraphClient(clientStub);
```

To facilitate debugging, [debug mode](#debug-mode) can be enabled for a client.

### Altering the Database

To set the schema, create an `Operation` object, set the schema and pass it to
`DgraphClient#alter(Operation)` method.

```js
const schema = "name: string @index(exact) .";
const op = new dgraph.Operation();
op.setSchema(schema);
await dgraphClient.alter(op);
```

> NOTE: Many of the examples here use the `await` keyword which requires
> `async/await` support which is available on Node.js >= v7.6.0. For prior versions,
> the expressions following `await` can be used just like normal `Promise`:
> 
> ```js
> dgraphClient.alter(op)
>     .then(function(result) { ... }, function(err) { ... })
> ```

`Operation` contains other fields as well, including drop predicate and drop all.
Drop all is useful if you wish to discard all the data, and start from a clean
slate, without bringing the instance down.

```js
// Drop all data including schema from the Dgraph instance. This is useful
// for small examples such as this, since it puts Dgraph into a clean
// state.
const op = new dgraph.Operation();
op.setDropAll(true);
await dgraphClient.alter(op);
```

### Creating a Transaction

To create a transaction, call `DgraphClient#newTxn()` method, which returns a
new `Txn` object. This operation incurs no network overhead.

It is good practise to call `Txn#discard()` in a `finally` block after running
the transaction. Calling `Txn#discard()` after `Txn#commit()` is a no-op
and you can call `Txn#discard()` multiple times with no additional side-effects.

```js
const txn = dgraphClient.newTxn();
try {
  // Do something here
  // ...
} finally {
  await txn.discard();
  // ...
}
```

### Running a Mutation

`Txn#mutate(Mutation)` runs a mutation. It takes in a `Mutation` object, which
provides two main ways to set data: JSON and RDF N-Quad. You can choose whichever
way is convenient.

We define a person object to represent a person and use it in a `Mutation` object.

```js
// Create data.
const p = {
    name: "Alice",
};

// Run mutation.
const mu = new dgraph.Mutation();
mu.setSetJson(p);
await txn.mutate(mu);
```

For a more complete example with multiple fields and relationships, look at the
[simple] project in the `examples` folder.

Sometimes, you only want to commit a mutation, without querying anything further.
In such cases, you can use `Mutation#setCommitNow(true)` to indicate that the
mutation must be immediately committed.

`Mutation#setIgnoreIndexConflict(true)` can be applied on a `Mutation` object to
not run conflict detection over the index, which would decrease the number of
transaction conflicts and aborts. However, this would come at the cost of potentially
inconsistent upsert operations.

Mutation can be run using `txn.doRequest` as well.

```js
const mu = new dgraph.Mutation();
mu.setSetJson(p);

const req = new dgraph.Request();
req.setCommitNow(true);
req.setMutationsList([mu]);

await txn.doRequest(req);
```

### Running a Query

You can run a query by calling `Txn#query(string)`. You will need to pass in a
GraphQL+- query string. If you want to pass an additional map of any variables that
you might want to set in the query, call `Txn#queryWithVars(string, object)` with
the variables object as the second argument.

The response would contain the method `Response#getJSON()`, which returns the response
JSON.

Let’s run the following query with a variable $a:

```console
query all($a: string) {
  all(func: eq(name, $a))
  {
    name
  }
}
```

Run the query, deserialize the result from Uint8Array (or base64) encoded JSON and
print it out:

```js
// Run query.
const query = `query all($a: string) {
  all(func: eq(name, $a))
  {
    name
  }
}`;
const vars = { $a: "Alice" };
const res = await dgraphClient.newTxn().queryWithVars(query, vars);
const ppl = res.getJson();

// Print results.
console.log(`Number of people named "Alice": ${ppl.all.length}`);
ppl.all.forEach((person) => console.log(person.name));
```

This should print:

```console
Number of people named "Alice": 1
Alice
```

You can also use `txn.doRequest` function to run the query.
```js
const req = new dgraph.Request();
const vars = req.getVarsMap();
vars.set("$a", "Alice");
req.setQuery(query);

const res = await txn.doRequest(req);
console.log(JSON.stringify(res.getJson()));
```

### Running an Upsert: Query + Mutation

The `txn.doRequest` function allows you to run upserts consisting of one query and one mutation. 
Query variables could be defined and can then be used in the mutation. You can also use the 
`txn.doRequest` function to perform just a query or a mutation.

To know more about upsert, we highly recommend going through the docs at https://docs.dgraph.io/mutations/#upsert-block.

```js
const query = `
  query {
      user as var(func: eq(email, "wrong_email@dgraph.io"))
  }`

const mu = new dgraph.Mutation();
mu.setSetNquads(`uid(user) <email> "correct_email@dgraph.io" .`);

const req = new dgraph.Request();
req.setQuery(query);
req.setMutationsList([mu]);
req.setCommitNow(true);

// Upsert: If wrong_email found, update the existing data
// or else perform a new mutation.
await dgraphClient.newTxn().doRequest(req);
```

### Running a Conditional Upsert

The upsert block allows specifying a conditional mutation block using an `@if` directive. The mutation is executed
only when the specified condition is true. If the condition is false, the mutation is silently ignored.

See more about Conditional Upsert [Here](https://docs.dgraph.io/mutations/#conditional-upsert).

```js
const query = `
  query {
      user as var(func: eq(email, "wrong_email@dgraph.io"))
  }`

const mu = new dgraph.Mutation();
mu.setSetNquads(`uid(user) <email> "correct_email@dgraph.io" .`);
mu.setCond(`@if(eq(len(user), 1))`);

const req = new dgraph.Request();
req.setQuery(query);
req.addMutations(mu);
req.setCommitNow(true);

await dgraphClient.newTxn().doRequest(req);
```

### Committing a Transaction

A transaction can be committed using the `Txn#commit()` method. If your transaction
consisted solely of calls to `Txn#query` or `Txn#queryWithVars`, and no calls to
`Txn#mutate`, then calling `Txn#commit()` is not necessary.

An error will be returned if other transactions running concurrently modify the same
data that was modified in this transaction. It is up to the user to retry
transactions when they fail.

```js
const txn = dgraphClient.newTxn();
try {
  // ...
  // Perform any number of queries and mutations
  // ...
  // and finally...
  await txn.commit();
} catch (e) {
  if (e === dgraph.ERR_ABORTED) {
    // Retry or handle exception.
  } else {
    throw e;
  }
} finally {
  // Clean up. Calling this after txn.commit() is a no-op
  // and hence safe.
  await txn.discard();
}
```

### Cleanup Resources

To cleanup resources, you have to call `DgraphClientStub#close()` individually for
all the instances of `DgraphClientStub`.

```js
const SERVER_ADDR = "localhost:9080";
const SERVER_CREDENTIALS = grpc.credentials.createInsecure();

// Create instances of DgraphClientStub.
const stub1 = new dgraph.DgraphClientStub(SERVER_ADDR, SERVER_CREDENTIALS);
const stub2 = new dgraph.DgraphClientStub(SERVER_ADDR, SERVER_CREDENTIALS);

// Create an instance of DgraphClient.
const dgraphClient = new dgraph.DgraphClient(stub1, stub2);

// ...
// Use dgraphClient
// ...

// Cleanup resources by closing all client stubs.
stub1.close();
stub2.close();
```

### Debug mode

Debug mode can be used to print helpful debug messages while performing alters,
queries and mutations. It can be set using the`DgraphClient#setDebugMode(boolean?)`
method.

```js
// Create a client.
const dgraphClient = new dgraph.DgraphClient(...);

// Enable debug mode.
dgraphClient.setDebugMode(true);
// OR simply dgraphClient.setDebugMode();

// Disable debug mode.
dgraphClient.setDebugMode(false);
```


## Examples

- [simple][]: Quickstart example of using dgraph-js.
- [tls][]: Example of using dgraph-js with a Dgraph cluster secured with TLS.

[simple]: ./examples/simple
[tls]: ./examples/tls

## Development

### Building the source

```sh
npm run build
```

If you have made changes to the `proto/api.proto` file, you need need to
regenerate the source files generated by Protocol Buffer tools. To do that,
install the [Protocol Buffer Compiler][protoc] and then run the following
command:

[protoc]: https://github.com/google/protobuf#readme

```sh
npm run build:protos
```

### Running tests

Make sure you have a Dgraph server running on localhost before you run this task.

```sh
npm test
```
