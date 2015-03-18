pg-promise
===========

Complete access layer to [PG] via [Promises/A+].

[![Build Status](https://travis-ci.org/vitaly-t/pg-promise.svg?branch=master)](https://travis-ci.org/vitaly-t/pg-promise)

---
<img align="right" width="190" height="190" src="https://upload.wikimedia.org/wikipedia/commons/a/a8/Pg-promise.jpg">

* Supporting [Promise], [Bluebird], [When], [Q], etc.
* Transactions, functions, flexible query formatting;
* Automatic database connections;
* Strict query result filters.

# Installing
```
$ npm install pg-promise
```

# Testing
* Install project dependencies
```
$ npm install
```
* Run tests
```
$ make test
```
On Windows you can also run tests with `test.bat`

# Getting started

### 1. Load the library
```javascript
// Loading the library:
var pgpLib = require('pg-promise');
```
### 2. Initialize the library
```javascript
// Initializing the library, with optional global settings:
var pgp = pgpLib(/*options*/);
```
You can pass additional `options` parameter when initializing the library (see chapter [Initialization Options](#advanced) for details).

### 3. Configure database connection
Use one of the two ways to specify connection details:
* Configuration Object:
```javascript
var cn = {
    host: 'localhost', // server name or IP address;
    port: 5432,
    database: 'my_db_name',
    user: 'user_name',
    password: 'user_password'
};
```
* Connection String:
```javascript
var cn = "postgres://username:password@host:port/database";
```
This library doesn't use any of the connection's details, it simply passes them on to [PG] when opening a new connection.
For more details see [ConnectionParameters] class in [PG], such as additional connection properties supported.

### 4. Instantiate your database
```javascript
var db = pgp(cn); // create a new database instance from the connection details
```
There can be multiple database objects instantiated in the application from different connection details.

You are now ready to make queries against the database.

# Usage

The library supports promise-chaining queries on shared and detached connections.
Choosing which one you want depends on the situation and personal preferences.

### Detached Connections

Queries in a detached promise chain maintain connection independently, they each acquire a connection from the pool,
execute the query and then release the connection.
```javascript
db.one("select * from users where id=$1", 123) // find the user from id;
    .then(function(data){
        // find 'login' records for the user found:
        return db.query("select * from audit where event=$1 and userId=$2", ["login", data.id]);
    })
    .then(function(data){
        // display found audit records;
        console.log(data);
    }, function(reason){
        console.log(reason); // display reason why the call failed;
    })
```
In a situation where only one request is to be made against the database, a detached chain is the only one that makes sense.
And even if you intend to execute multiple queries in a chain, keep in mind that even though each will use its own connection,
such will be used from a connection pool, so effectively you end up with the same connection, without any performance penalty.

### Shared Connections

A promise chain with a shared connection always starts with ```connect()```, which allocates a connection that's shared with all the
query requests down the chain. The connection must be released when no longer needed.

```javascript
var sco; // shared connection object;
db.connect()
    .then(function(obj){
        sco = obj; // save the connection object;
        // find active users created before today:
        return sco.query("select * from users where active=$1 and created < $2::date", [true, new Date()]);
    })
    .then(function(data){
        console.log(data); // display all the user details;
    }, function(reason){
        console.log(reason); // display reason why the call failed;
    })
    .done(function(){
        if(sco){
            sco.done(); // release the connection, if it was acquired successfully;
        }
    });
```
Shared-connection chaining is for those who want absolute control over connection, either because they want to execute lots of queries in one go,
or because they like squeezing every bit of performance out of their code. Other than, the author hasn't seen any real performance difference
from the detached-connection chaining.

### Transactions

Transactions can be executed within both shared and detached promise chains in the same way, performing the following actions:

1. Acquires a new connection (detached chains only);
2. Executes `BEGIN` command;
3. Invokes your callback function with the connection object;
4. Executes `COMMIT`, if the callback resolves, or `ROLLBACK`, if the callback rejects;
5. Releases the connection (detached chains only);
6. Resolves with the callback result, if success; rejects with the reason, if failed.

###### Example of a detached transaction:

```javascript
var promise = require('promise'); // or any other supported promise library;
db.tx(function(ctx){

    // creating a sequence of transaction queries:
    var q1 = ctx.none("update users set active=$1 where id=$2", [true, 123]);
    var q2 = ctx.one("insert into audit(entity, id) values($1, $2) returning id", ['users', 123]);

    // returning a promise that determines a successful transaction:
    return promise.all([q1, q2]); // all of the queries are to be resolved

}).then(function(data){
    console.log(data); // printing successful transaction output
}, function(reason){
    console.log(reason); // printing the reason why the transaction was rejected
});
```
A detached transaction acquires a connection and exposes object `ctx` to let all containing queries execute on the same connection.

And when executing a transaction within a shared connection chain, the only thing that changes is that parameter `ctx` becomes the
same as parameter `sco` from opening a shared connection, so either one can be used inside such a transaction interchangeably.

###### Shared-connection transaction:
```javascript
var promise = require('promise'); // or any other supported promise library;
var sco; // shared connection object;
db.connect()
    .then(function(obj){
        sco = obj;
        return sco.oneOrNone("select * from users where active=$1 and id=$1", [true, 123]);
    })
    .then(function(data){
        return sco.tx(function(ctx){

            // Since it is a transaction within a shared chain, it doesn't matter whether
            // the two calls below use object `ctx` or `sco`, as they are exactly the same:
            var q1 = ctx.none("update users set active=$1 where id=$2", [false, data.id]);
            var q2 = sco.one("insert into audit(entity, id) values($1, $2) returning id", ['users', 123]);

            // returning a promise that determines a successful transaction:
            return promise.all([q1, q2]); // all of the queries are to be resolved;
        });
    }, function(reason){
        console.log(reason); // printing the reason why the transaction was rejected;
    })
    .done(function(){
        if(sco){
            sco.done(); // release the connection, if it was acquired successfully;
        }
    });
```
If you need to execute just one transaction, the detached transaction pattern is all you need.
But even if you need to combine it with other queries in then a detached chain, it will work just as fine.
As stated earlier, choosing a shared chain over a detached one is mostly a matter of special requirements and/or personal preference.

### Queries and Parameters

Every connection context of the library shares the same query protocol, starting with generic method `query`, that's defined as shown below:
```javascript
function query(query, values, qrm);
```
* `query` (required) - query string that supports standard variables formatting, using $1, $2, ...etc;
* `values` (optional) - simple value or an array of simple values to replace the query variables;
* `qrm` - (optional) Query Result Mask, as explained below...

In order to eliminate the chances of unexpected query results and make code more robust, each request supports parameter `qrm`
(Query Result Mask), via type `queryResult`:
```javascript
///////////////////////////////////////////////////////
// Query Result Mask flags;
//
// Any combination is supported, except for one + many.
queryResult = {
    one: 1,     // single-row result is expected;
    many: 2,    // multi-row result is expected;
    none: 4,    // no rows expected;
    any: 6      // (default) = many|none = any result.
};
```

In the following generic-query example we indicate that the call can return anything:
```javascript
db.query("select * from users");
```
which is equivalent to calling either one of the following:
```javascript
db.query("select * from users", null, queryResult.many | queryResult.none);
db.query("select * from users", null, queryResult.any);
db.manyOrNone("select * from users");
db.any("select * from users");
```
This usage pattern is facilitated through result-specific methods that can be used instead of the generic query:
```javascript
db.many(query, values); // expects one or more rows
db.one(query, values); // expects single row
db.none(query, values); // expects no rows
db.any(query, values); // expects anything, same as `manyOrNone`
db.oneOrNone(query, values); // expects 1 or 0 rows
db.manyOrNone(query, values); // expects anything, same as `any`
```

Each query function resolves its **data** object according to the `qrm` that was used:
* `none` - **data** is `null`. If the query returns any kind of data, it is rejected.
* `one` - **data** is a single object. If the query returns no data or more than one row of data, it is rejected.
* `many` - **data** is an array of objects. If the query returns no rows, it is rejected.
* `one` | `none` - **data** is `null`, if no data was returned; or a single object, if there was one row of data returned.
If the query returns more than one row of data, the query is rejected.
* `many` | `none` - **data** is an array of objects. When no rows are returned, **data** is an empty array.

If you try to specify `one` | `many` in the same query, such query will be rejected without executing it, telling you that such mask is not valid.

If `qrm` is not specified when calling generic `query` method, it is assumed to be `many` | `none`, i.e. any kind of data expected.

> This is all about writing robust code, when the client specifies what kind of data it is ready to handle on the declarative level,
leaving the burden of all extra checks to the library.

### Functions and Procedures
In PostgreSQL stored procedures are just functions that usually do not return anything.

Suppose we want to call function **findAudit** to find audit records by **user id** and maximum timestamp.
We can make such call as shown below:
```javascript
db.func('findAudit', [123, new Date()])
    .then(function(data){
        console.log(data); // printing the data returned
    }, function(reason){
        console.log(reason); // printing the reason why the call was rejected
    });
```
We passed it **user id** = 123, plus current Date/Time as the timestamp. We assume that the function signature matches the parameters that we passed.
All values passed are serialized automatically to comply with PostgreSQL type formats.

Method `func` accepts optional third parameter - `qrm` (Query Result Mask), the same as method `query`.

And when you are not expecting any return results, call `db.proc` instead. Both methods return a [Promise] object,
but `db.proc` doesn't take a `qrm` parameter, always assuming it is `one`|`none`.

Summary for supporting procedures and functions:
```javascript
db.func(query, values, qrm); // expects the result according to `qrm`
db.proc(query, values); // calls db.func(query, values, queryResult.one | queryResult.none)
```

### Type Helpers
The library provides several helper functions to convert basic javascript types into their proper PostgreSQL presentation that can be passed
directly into queries or functions as parameters. All of such helper functions are located within namespace ```pgp.as```:
```javascript
pgp.as.bool(value); // returns proper PostgreSQL boolean presentation

pgp.as.text(value); // returns proper PostgreSQL text presentation,
                    // fixing single-quote symbols, wrapped in quotes

pgp.as.date(value); // returns proper PostgreSQL date/time presentation,
                    // wrapped in quotes.

pgp.as.csv(array);  // returns a CSV string with values formatted according
                    // to their type, using the above methods.

pgp.as.format(query, values);
            // Replaces variables in a query with their `values` as specified.
            // `values` is either a simple value or an array of simple values.
            // This method is used implicitly by every query method in the library,
            // and the main reason it was added here is to make it testable, because
            // it represents a very important aspect of the library's functionality.
```
As these helpers are not associated with any database, they can be used from anywhere.

# Advanced

### Initialization Options

When initializing the library, you can pass object `options` with a set of properties
for global override of the library's behaviour:
```javascript
var options = {
    // pgFormatting - redirects query formatting to PG;
    // promiseLib - overrides default promise library;
    // connect - database 'connect' notification;
    // disconnect - database 'disconnect' notification;
    // query - query execution notification.
};
var pgp = pgpLib(options);
```

Below is the list of all the properties that are currently supported.

---
* `pgFormatting`

By default, **pg-promise** provides its own implementation of the query value formatting,
using the standard syntax of $1, $2, etc.

Any query request accepts values for the variable within query string as the second parameter.
It accepts a single simple value for queries that use only one variable, as well as an array of simple values,
for queries with multiple variables in them.

**pg-promise** implementation of query formatting supports only the basic javascript types: text, boolean, date, numeric and null.

If, instead, you want to use query formatting that's implemented by the [PG] library, set parameter `pgFormatting`
to be `true` when initializing the library, and every query formatting will redirect to the [PG]'s implementation.

Although this has huge implication to the library's functionality, it is not within the scope of this project to detail.
For any further reference you should use documentation of the [PG] library.

**NOTE:** As of the current implementation, formatting parameters for calling functions (methods `func` and `proc`) is not affected by this
override. If needed, use the generic `query` instead to invoke functions with redirected query formatting.

---
* `promiseLib`

Set this property to an alternative promise library compliant with the [Promises/A+] standard.

By default, **pg-promise** uses version of [Promises/A+] provided by [Promise]. If you want to override
this and force the library to use a different implementation of the standard, just set this parameter
to the library's instance.

Example of switching over to [Bluebird]:
```javascript
var promise = require('bluebird');
var options = {
    promiseLib: promise
};
var pgp = pgpLib(options);
```

[Promises/A+] libraries that passed our compatibility test and are currently supported:

* [Promise] - used by default
* [Bluebird]
* [When]
* [Q]
* [RSVP] - doesn't have `done()`, use `finally/catch` instead
* [Lie] - doesn't have `done()`


Compatibility with other [Promises/A+] libraries though possible, is an unknown.

---
* `connect`

Global notification function of acquiring a new database connection.
```javascript
var options = {
    connect: function(client){
        var cp = client.connectionParameters;
        console.log("Connected to database '" + cp.database + "'");
    }
}
```
It can be used for diagnostics / connection monitoring within your application.

The function takes only one parameter - `client` object from the [PG] library that represents connection
with the database.

---
* `disconnect`

Global notification function of releasing a database connection.
```javascript
var options = {
    disconnect: function(client){
        var cp = client.connectionParameters;
        console.log("Disconnecting from database '" + cp.database + "'");
    }
}
```
It can be used for diagnostics / connection monitoring within your application.

The function takes only one parameter - `client` object from the [PG] library that represents the connection
that's being released.

---
* `query`

Global notification of a query that's being executed.
```javascript
var options = {
    query: function(client, query, params){
        console.log("Executing query: " + query);
    }
}
```
It can be useful for diagnostics / logging within your application.

The function receives the following parameters:
* `client` - object from the [PG] library that represents the connection;
* `query` - query that's being executed;
* `params` - `null` by default, it may contains query parameters, if `pgFormatting` was set to be `true`.

Please note, that should you set property `pgFormatting` to be `true`, the library no longer formats
the queries, and the `query` arrives pre-formatted. This is why extra parameter `params` was added.

### Library de-initialization
When exiting your application, make the following call:
```javascript
pgp.end();
```
This will release pg connection pool globally and make sure that the process terminates without any delay.
If you do not call it, your process may be waiting for 30 seconds (default) or so, waiting for the pg connection pool to expire.

# History
* Version 0.5.3 - minor changes; March 14, 2015.
* Version 0.5.1 included wider support for alternative promise libraries. Released: March 12, 2015.
* Version 0.5.0 introduces many new features and fixes, such as properties **pgFormatting** and **promiseLib**. Released on March 11, 2015.
* Version 0.4.9 represents a solid code base, backed up by comprehensive testing. Released on March 10, 2015.
* Version 0.4.0 is a complete rewrite of most of the library, made first available on March 8, 2015.
* Version 0.2.0 introduced on March 6th, 2015, supporting multiple databases.
* A refined version 0.1.4 released on March 5th, 2015.
* First solid Beta, 0.1.2 on March 4th, 2015.
* It reached first Beta version 0.1.0 on March 4th, 2015.
* The first draft v0.0.1 was published on March 3rd, 2015, and then rapidly incremented due to many initial changes that had to come in, mostly documentation.

[PG]:https://github.com/brianc/node-postgres
[ConnectionParameters]:https://github.com/brianc/node-postgres/blob/master/lib/connection-parameters.js
[Promises/A+]:https://promisesaplus.com/
[Promise]:https://github.com/then/promise
[Bluebird]:https://github.com/petkaantonov/bluebird
[When]:https://github.com/cujojs/when
[Q]:https://github.com/kriskowal/q
[RSVP]:https://github.com/tildeio/rsvp.js
[Lie]:https://github.com/calvinmetcalf/lie

# License

Copyright (c) 2015 Vitaly Tomilov (vitaly.tomilov@gmail.com)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"),
to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense,
and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.
