<a name="slonik"></a>
# Slonik

[![Travis build status](http://img.shields.io/travis/gajus/slonik/master.svg?style=flat-square)](https://travis-ci.org/gajus/slonik)
[![Coveralls](https://img.shields.io/coveralls/gajus/slonik.svg?style=flat-square)](https://coveralls.io/github/gajus/slonik)
[![NPM version](http://img.shields.io/npm/v/slonik.svg?style=flat-square)](https://www.npmjs.org/package/slonik)
[![Canonical Code Style](https://img.shields.io/badge/code%20style-canonical-blue.svg?style=flat-square)](https://github.com/gajus/canonical)
[![Twitter Follow](https://img.shields.io/twitter/follow/kuizinas.svg?style=social&label=Follow)](https://twitter.com/kuizinas)

A PostgreSQL client with strict types, detail logging and assertions.

<a name="slonik-features"></a>
## Features

* Predominantly compatible with [node-postgres](https://github.com/brianc/node-postgres) (see [Incompatibilities with `node-postgres`](#incompatibilities-with-node-postgres)).
* [Convenience methods](#slonik-query-methods) with built-in assertions.
* [Middleware](#slonik-interceptors) support.
* [Syntax highlighting](#slonik-syntax-highlighting) (Atom plugin compatible with Slonik).
* [SQL injection guarding](#slonik-value-placeholders-tagged-template-literals).
* Detail [logging](#slonik-debugging).
* [Parsing and logging of the auto_explain logs.](#logging-auto_explain).
* Built-in [asynchronous stack trace resolution](#log-stack-trace).
* [Safe connection pooling](#checking-out-a-client-from-the-connection-pool).
* [Flow types](#types).
* [Mapped errors](#error-handling).
* [Transactions](#transactions).
* [ESLint plugin](https://github.com/gajus/eslint-plugin-sql).

---

<a name="slonik-documentation"></a>
## Documentation

* [Slonik](#slonik)
    * [Features](#slonik-features)
    * [Documentation](#slonik-documentation)
    * [Usage](#slonik-usage)
        * [Configuration](#slonik-usage-configuration)
        * [Checking out a client from the connection pool](#slonik-usage-checking-out-a-client-from-the-connection-pool)
    * [Interceptors](#slonik-interceptors)
        * [Interceptor methods](#slonik-interceptors-interceptor-methods)
    * [Built-in interceptors](#slonik-built-in-interceptors)
        * [Field name transformation interceptor](#slonik-built-in-interceptors-field-name-transformation-interceptor)
        * [Query normalization interceptor](#slonik-built-in-interceptors-query-normalization-interceptor)
    * [Recipes](#slonik-recipes)
    * [Incompatibilities with `node-postgres`](#slonik-incompatibilities-with-node-postgres)
    * [Conventions](#slonik-conventions)
        * [No multiline values](#slonik-conventions-no-multiline-values)
    * [Value placeholders](#slonik-value-placeholders)
        * [Tagged template literals](#slonik-value-placeholders-tagged-template-literals)
        * [Sets](#slonik-value-placeholders-sets)
        * [Multiple value sets](#slonik-value-placeholders-multiple-value-sets)
    * [Query methods](#slonik-query-methods)
        * [`any`](#slonik-query-methods-any)
        * [`anyFirst`](#slonik-query-methods-anyfirst)
        * [`insert`](#slonik-query-methods-insert)
        * [`many`](#slonik-query-methods-many)
        * [`manyFirst`](#slonik-query-methods-manyfirst)
        * [`maybeOne`](#slonik-query-methods-maybeone)
        * [`maybeOneFirst`](#slonik-query-methods-maybeonefirst)
        * [`one`](#slonik-query-methods-one)
        * [`oneFirst`](#slonik-query-methods-onefirst)
        * [`query`](#slonik-query-methods-query)
        * [`transaction`](#slonik-query-methods-transaction)
    * [Error handling](#slonik-error-handling)
        * [Handling `NotFoundError`](#slonik-error-handling-handling-notfounderror)
        * [Handling `DataIntegrityError`](#slonik-error-handling-handling-dataintegrityerror)
        * [Handling `NotNullIntegrityConstraintViolationError`](#slonik-error-handling-handling-notnullintegrityconstraintviolationerror)
        * [Handling `ForeignKeyIntegrityConstraintViolationError`](#slonik-error-handling-handling-foreignkeyintegrityconstraintviolationerror)
        * [Handling `UniqueIntegrityConstraintViolationError`](#slonik-error-handling-handling-uniqueintegrityconstraintviolationerror)
        * [Handling `CheckIntegrityConstraintViolationError`](#slonik-error-handling-handling-checkintegrityconstraintviolationerror)
    * [Types](#slonik-types)
    * [Debugging](#slonik-debugging)
        * [Logging](#slonik-debugging-logging)
        * [Log stack trace](#slonik-debugging-log-stack-trace)
    * [Syntax highlighting](#slonik-syntax-highlighting)
        * [Atom](#slonik-syntax-highlighting-atom)


<a name="slonik-usage"></a>
## Usage

Slonik exports two factory functions:

* `createPool`
* `createConnection`

The API of the query method is equivalent to that of [`pg`](https://travis-ci.org/brianc/node-postgres).

Refer to [query methods](#slonik-query-methods) for documentation of Slonik-specific query methods.

<a name="slonik-usage-configuration"></a>
### Configuration

Both functions accept the same parameters:

* `connectionConfiguration`
* `clientConfiguration`

```js
type DatabaseConnectionUriType = string;

type DatabaseConfigurationType =
  DatabaseConnectionUriType |
  {|
    +database?: string,
    +host?: string,
    +idleTimeoutMillis?: number,
    +max?: number,
    +password?: string,
    +port?: number,
    +user?: string
  |};

/**
 * @property interceptors An array of [Slonik interceptors](https://github.com/gajus/slonik#slonik-interceptors).
 */
type ClientConfigurationType = {|
  +interceptors?: $ReadOnlyArray<InterceptorType>
|};

```

Example:

```js
import {
  createPool
} from 'slonik';

const pool = createPool('postgres://localhost');

await pool.query(sql`SELECT 1`);

```

<a name="slonik-usage-checking-out-a-client-from-the-connection-pool"></a>
### Checking out a client from the connection pool

Slonik only allows to check out a connection for a duration of promise routine supplied to the `connect()` method.

```js
import {
  createPool
} from 'slonik';

const pool = createPool('postgres://localhost');

const result = await pool.connect(async (connection) => {
  await connection.query(sql`SELECT 1`);
  await connection.query(sql`SELECT 2`);

  return 'foo';
});

result;
// 'foo'

```

Connection is released back to the pool after the promise produced by the function supplied to `connect()` method is either resolved or rejected.

The primary reason for implementing _only_ this connection pooling method is because the alternative is inherently unsafe, e.g.

```js
// Note: This example is using unsupported API.

const main = async () => {
  const connection = await pool.connect();

  await connection.query(sql`SELECT produce_error()`);

  await connection.release();
};

```

In this example, the error causes early rejection of the promise and a hanging connection. A fix to the above is to ensure that `connection#release()` is always called, i.e.

```js
// Note: This example is using unsupported API.

const main = async () => {
  const connection = await pool.connect();

  let lastExecutionResult;

  try {
    lastExecutionResult = await connection.query(sql`SELECT produce_error()`);
  } finally {
    await connection.release();
  }

  return lastExecutionResult;
};

```

Slonik abstracts the latter pattern into `pool#connect()` method.


<a name="slonik-interceptors"></a>
## Interceptors

Functionality can be added to Slonik client by adding interceptors.

Each interceptor can implement several functions which can be used to change the behaviour of the database client.

```js
type InterceptorType = {|
  +afterConnection?: (connection: DatabaseSingleConnectionType) => MaybePromiseType<void>,
  +afterPoolConnection?: (connection: DatabasePoolConnectionType) => MaybePromiseType<void>,
  +afterQueryExecution?: (
    queryExecutionContext: QueryExecutionContextType,
    query: QueryType,
    result: QueryResultType<QueryResultRowType>
  ) => MaybePromiseType<QueryResultType<QueryResultRowType>>,
  +beforeConnectionEnd?: (connection: DatabaseSingleConnectionType) => MaybePromiseType<void>,
  +beforePoolConnectionRelease?: (connection: DatabasePoolConnectionType) => MaybePromiseType<void>,
  +beforeQueryExecution?: (
    queryExecutionContext: QueryExecutionContextType,
    query: QueryType
  ) => MaybePromiseType<QueryResultType<QueryResultRowType>> | MaybePromiseType<void>,
  +transformQuery?: (
    queryExecutionContext: QueryExecutionContextType,
    query: QueryType
  ) => MaybePromiseType<QueryType>
|};

```

Interceptors are configured using [client configuration](#slonik-usage-configuration), e.g.

```js
import {
  createPool
} from 'slonik';

const interceptors = [];

const connection = createPool('postgres://', {
  interceptors
});

```

Interceptors are executed in the order they are added.

<a name="slonik-interceptors-interceptor-methods"></a>
### Interceptor methods

<a name="slonik-interceptors-interceptor-methods-afterconnection"></a>
#### <code>afterConnection</code>

Executed after a connection is established, e.g.

```js
// Interceptor is executed here. ↓
await createConnection('postgres://');

```

<a name="slonik-interceptors-interceptor-methods-afterpoolconnection"></a>
#### <code>afterPoolConnection</code>

Executed after a connection is , e.g.

```js
const pool = createPool('postgres://');

// Interceptor is executed here. ↓
pool.connect();

```

<a name="slonik-interceptors-interceptor-methods-afterqueryexecution"></a>
#### <code>afterQueryExecution</code>

`afterQueryExecution` must return the result of the query, which will be passed down to the client.

Use `afterQuery` to modify the query result.

<a name="slonik-interceptors-interceptor-methods-beforequeryexecution"></a>
#### <code>beforeQueryExecution</code>

This function can optionally return a direct result of the query which will cause the actual query never to be executed.

<a name="slonik-interceptors-interceptor-methods-beforeconnectionend"></a>
#### <code>beforeConnectionEnd</code>

`beforeConnectionEnd` is executed before a connection is explicitly ended, e.g.

```js
const connection = await createConnection('postgres://');

// Interceptor is executed here. ↓
connection.end();

```

<a name="slonik-interceptors-interceptor-methods-beforepoolconnectionrelease"></a>
#### <code>beforePoolConnectionRelease</code>

Executed before connection is released back to the connection pool, e.g.

```js
const pool = await createPool('postgres://');

pool.connect(async () => {
  await 1;

  // Interceptor is executed here. ↓
});

```

<a name="slonik-interceptors-interceptor-methods-transformquery"></a>
#### <code>transformQuery</code>

Executed before `beforeQueryExecution`.

Transforms query.

<a name="slonik-built-in-interceptors"></a>
## Built-in interceptors

<a name="slonik-built-in-interceptors-field-name-transformation-interceptor"></a>
### Field name transformation interceptor

`createFormatFieldNameInterceptor` creates an interceptor that formats query result field names.

This interceptor removes the necessity to alias field names, e.g.

```js
connection.any(sql`
  SELECT
    id,
    full_name "fullName"
  FROM person
`);

```

Field name formatter uses `afterQuery` interceptor to format field names.

<a name="slonik-built-in-interceptors-field-name-transformation-interceptor-api"></a>
#### API

```js
/**
 * @property format The only supported format is CAMEL_CASE.
 * @property test Tests whether the field should be formatted. The default behaviour is to include all fields that match ^[a-z0-9_]+$ regex.
 */
type ConfigurationType = {|
  +format: 'CAMEL_CASE',
  +test: (field: FieldType) => boolean
|};

(configuration: ConfigurationType) => InterceptorType;

```

<a name="slonik-built-in-interceptors-field-name-transformation-interceptor-example-usage"></a>
#### Example usage

```js
import {
  createFormatFieldNameInterceptor,
  createPool
} from 'slonik';

const interceptors = [
  createFormatFieldNameInterceptor({
    format: 'CAMEL_CASE'
  })
];

const connection = createPool('postgres://', {
  interceptors
});

connection.any(sql`
  SELECT
    id,
    full_name
  FROM person
`);

// [
//   {
//     id: 1,
//     fullName: 1
//   }
// ]

```

<a name="slonik-built-in-interceptors-query-normalization-interceptor"></a>
### Query normalization interceptor

Normalizes the query.

<a name="slonik-built-in-interceptors-query-normalization-interceptor-api-1"></a>
#### API

```js
/**
 * @property stripComments Strips comments from the query (default: true).
 */
type ConfigurationType = {|
  +stripComments?: boolean
|};

(configuration?: ConfigurationType) => InterceptorType;

```


<a name="slonik-recipes"></a>
## Recipes

N/A


<a name="slonik-incompatibilities-with-node-postgres"></a>
## Incompatibilities with <code>node-postgres</code>

* `timestamp` and `timestamp with time zone` returns UNIX timestamp in milliseconds.
* Connection pool `connect()` method requires that connection is restricted to a single promise routine (see [Checking out a client from the connection pool](#checking-out-a-client-from-the-connection-pool)).

<a name="slonik-conventions"></a>
## Conventions

<a name="slonik-conventions-no-multiline-values"></a>
### No multiline values

Slonik will strip all comments and line-breaks from a query before processing it.

This makes logging of the queries easier.

The implication is that your query cannot contain values that include a newline character, e.g.

```sql
// Do not do this
connection.query(sql`INSERT INTO foo (bar) VALUES ('\n')`);

```

If you want to communicate a value that includes a multiline character, use value placeholder interpolation, e.g.

```sql
connection.query(sql`INSERT INTO foo (bar) VALUES (${'\n'})`);

```

<a name="slonik-value-placeholders"></a>
## Value placeholders

<a name="slonik-value-placeholders-tagged-template-literals"></a>
### Tagged template literals

Slonik query methods can only be executed using `sql` [tagged template literal](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Template_literals#Tagged_template_literals), e.g.

```js
import {
  sql
} from 'slonik'

connection.query(sql`
  INSERT INTO reservation_ticket (reservation_id, ticket_id)
  VALUES ${values}
`);

```

<a name="slonik-value-placeholders-sets"></a>
### Sets

Array expressions produce sets, e.g.

```js
await connection.query(sql`
  SELECT ${[1, 2, 3]}
`);

```

Produces:

```sql
SELECT ($1, $2, $3)

```

<a name="slonik-value-placeholders-multiple-value-sets"></a>
### Multiple value sets

An array containing array expressions produce a collection of sets, e.g.

```js
await connection.query(sql`
  SELECT ${[
    [1, 2, 3],
    [1, 2, 3]
  ]}
`);

```

Produces:

```sql
SELECT ($1, $2, $3), ($4, $5, $6)

```

<a name="slonik-value-placeholders-multiple-value-sets-creating-dynamic-delimited-identifiers"></a>
#### Creating dynamic delimited identifiers

[Delimited identifiers](https://www.postgresql.org/docs/current/static/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS) are created by enclosing an arbitrary sequence of characters in double-quotes ("). To create create a delimited identifier, create an `sql` tag function placeholder value using `sql.identifier`, e.g.

```js
sql`
  SELECT ${'foo'}
  FROM ${sql.identifier(['bar', 'baz'])}
`;

// {
//   sql: 'SELECT ? FROM "bar"."baz"',
//   values: [
//     'foo'
//   ]
// }

```

<a name="slonik-value-placeholders-multiple-value-sets-inlining-dynamic-raw-sql"></a>
#### Inlining dynamic/ raw SQL

Raw SQL can be inlined using `sql.raw`, e.g.

```js
sql`
  SELECT ${'foo'}
  FROM ${sql.raw('"bar"')}
`;

// {
//   sql: 'SELECT ? FROM "bar"',
//   values: [
//     'foo'
//   ]
// }

```


<a name="slonik-query-methods"></a>
## Query methods

<a name="slonik-query-methods-any"></a>
### <code>any</code>

Returns result rows.

Example:

```js
const rows = await connection.any(sql`SELECT foo`);

```

`#any` is similar to `#query` except that it returns rows without fields information.

<a name="slonik-query-methods-anyfirst"></a>
### <code>anyFirst</code>

Returns value of the first column of every row in the result set.

* Throws `DataIntegrityError` if query returns multiple rows.

Example:

```js
const fooValues = await connection.anyFirst(sql`SELECT foo`);

```

<a name="slonik-query-methods-insert"></a>
### <code>insert</code>

Used when inserting 1 row.

Example:

```js
const {
  insertId
} = await connection.insert(sql`INSERT INTO foo SET bar='baz'`);

```

The reason for using this method over `#query` is to leverage the strict types. `#insert` method result type is `InsertResultType`.

<a name="slonik-query-methods-many"></a>
### <code>many</code>

Returns result rows.

* Throws `NotFoundError` if query returns no rows.

Example:

```js
const rows = await connection.many(sql`SELECT foo`);

```

<a name="slonik-query-methods-manyfirst"></a>
### <code>manyFirst</code>

Returns value of the first column of every row in the result set.

* Throws `NotFoundError` if query returns no rows.
* Throws `DataIntegrityError` if query returns multiple rows.

Example:

```js
const fooValues = await connection.many(sql`SELECT foo`);

```

<a name="slonik-query-methods-maybeone"></a>
### <code>maybeOne</code>

Selects the first row from the result.

* Returns `null` if row is not found.
* Throws `DataIntegrityError` if query returns multiple rows.

Example:

```js
const row = await connection.maybeOne(sql`SELECT foo`);

// row.foo is the result of the `foo` column value of the first row.

```

<a name="slonik-query-methods-maybeonefirst"></a>
### <code>maybeOneFirst</code>

Returns value of the first column from the first row.

* Returns `null` if row is not found.
* Throws `DataIntegrityError` if query returns multiple rows.
* Throws `DataIntegrityError` if query returns multiple columns.

Example:

```js
const foo = await connection.maybeOneFirst(sql`SELECT foo`);

// foo is the result of the `foo` column value of the first row.

```

<a name="slonik-query-methods-one"></a>
### <code>one</code>

Selects the first row from the result.

* Throws `NotFoundError` if query returns no rows.
* Throws `DataIntegrityError` if query returns multiple rows.

Example:

```js
const row = await connection.one(sql`SELECT foo`);

// row.foo is the result of the `foo` column value of the first row.

```

> Note:
>
> I've been asked "What makes this different from [knex.js](http://knexjs.org/) `knex('foo').limit(1)`?".
> `knex('foo').limit(1)` simply generates "SELECT * FROM foo LIMIT 1" query.
> `knex` is a query builder; it does not assert the value of the result.
> Slonik `#one` adds assertions about the result of the query.

<a name="slonik-query-methods-onefirst"></a>
### <code>oneFirst</code>

Returns value of the first column from the first row.

* Throws `NotFoundError` if query returns no rows.
* Throws `DataIntegrityError` if query returns multiple rows.
* Throws `DataIntegrityError` if query returns multiple columns.

Example:

```js
const foo = await connection.oneFirst(sql`SELECT foo`);

// foo is the result of the `foo` column value of the first row.

```

<a name="slonik-query-methods-query"></a>
### <code>query</code>

API and the result shape are equivalent to [`pg#query`](https://github.com/brianc/node-postgres).

<a name="slonik-query-methods-transaction"></a>
### <code>transaction</code>

`transaction` method is used wrap execution of queries in `START TRANSACTION` and `COMMIT` or `ROLLBACK`. `COMMIT` is called if the transaction handler returns a promise that resolves; `ROLLBACK` is called otherwise.

`transaction` method can be used together with `createPool` method. When used to create a transaction from an instance of a pool, a new connection is allocated for the duration of the transaction.

```js
const result = await connection.transaction(async (transactionConnection) => {
  await transactionConnection.query(sql`INSERT INTO foo (bar) VALUES ('baz')`);
  await transactionConnection.query(sql`INSERT INTO qux (quux) VALUES ('quuz')`);

  return 'FOO';
});

result === 'FOO';

```


<a name="slonik-error-handling"></a>
## Error handling

All Slonik errors extend from `SlonikError`, i.e. You can catch Slonik specific errors using the following logic.

```js
import {
  SlonikError
} from 'slonik';

try {
  await query();
} catch (error) {
  if (error instanceof SlonikError) {
    // This error is thrown by Slonik.
  }
}

```

<a name="slonik-error-handling-handling-notfounderror"></a>
### Handling <code>NotFoundError</code>

To handle the case where query returns less than one row, catch `NotFoundError` error.

```js
import {
  NotFoundError
} from 'slonik';

let row;

try {
  row = await connection.one(sql`SELECT foo`);
} catch (error) {
  if (!(error instanceof NotFoundError)) {
    throw error;
  }
}

if (row) {
  // row.foo is the result of the `foo` column value of the first row.
}

```

<a name="slonik-error-handling-handling-dataintegrityerror"></a>
### Handling <code>DataIntegrityError</code>

To handle the case where the data result does not match the expectations, catch `DataIntegrityError` error.

```js
import {
  NotFoundError
} from 'slonik';

let row;

try {
  row = await connection.one(sql`SELECT foo`);
} catch (error) {
  if (error instanceof DataIntegrityError) {
    console.error('There is more than one row matching the select criteria.');
  } else {
    throw error;
  }
}

```

<a name="slonik-error-handling-handling-notnullintegrityconstraintviolationerror"></a>
### Handling <code>NotNullIntegrityConstraintViolationError</code>

`NotNullIntegrityConstraintViolationError` is thrown when Postgres responds with [`unique_violation`](https://www.postgresql.org/docs/9.4/static/errcodes-appendix.html) (`23502`) error.

<a name="slonik-error-handling-handling-foreignkeyintegrityconstraintviolationerror"></a>
### Handling <code>ForeignKeyIntegrityConstraintViolationError</code>

`ForeignKeyIntegrityConstraintViolationError` is thrown when Postgres responds with [`unique_violation`](https://www.postgresql.org/docs/9.4/static/errcodes-appendix.html) (`23503`) error.

<a name="slonik-error-handling-handling-uniqueintegrityconstraintviolationerror"></a>
### Handling <code>UniqueIntegrityConstraintViolationError</code>

`UniqueIntegrityConstraintViolationError` is thrown when Postgres responds with [`unique_violation`](https://www.postgresql.org/docs/9.4/static/errcodes-appendix.html) (`23505`) error.

<a name="slonik-error-handling-handling-checkintegrityconstraintviolationerror"></a>
### Handling <code>CheckIntegrityConstraintViolationError</code>

`CheckIntegrityConstraintViolationError` is thrown when Postgres responds with [`unique_violation`](https://www.postgresql.org/docs/9.4/static/errcodes-appendix.html) (`23514`) error.

<a name="slonik-types"></a>
## Types

This package is using [Flow](https://flow.org/) types.

Refer to [`./src/types.js`](./src/types.js).

The public interface exports the following types:

* `DatabaseConnectionType`
* `DatabasePoolConnectionType`
* `DatabaseSingleConnectionType`

Use these types to annotate `connection` instance in your code base, e.g.

```js
// @flow

import type {
  DatabaseConnectionType
} from 'slonik';

export default async (
  connection: DatabaseConnectionType,
  code: string
): Promise<number> => {
  const countryId = await connection.oneFirst(sql`
    SELECT id
    FROM country
    WHERE code = ${code}
  `);

  return countryId;
};

```


<a name="slonik-debugging"></a>
## Debugging

<a name="slonik-debugging-logging"></a>
### Logging

Slonik uses [roarr](https://github.com/gajus/roarr) to log queries.

To enable logging, define `ROARR_LOG=true` environment variable.

By default, Slonik logs the input query, query execution time and affected row count.

You can enable additional logging details by configuring the following environment variables.

```bash
# Logs query parameter values
export SLONIK_LOG_VALUES=true

# Logs normalised query and input values
export SLONIK_LOG_NORMALISED=true

```

<a name="slonik-debugging-log-stack-trace"></a>
### Log stack trace

`SLONIK_LOG_STACK_TRACE=1` will create a stack trace before invoking the query and include the stack trace in the logs, e.g.

```json
{"context":{"package":"slonik","namespace":"slonik","logLevel":20,"executionTime":"357 ms","queryId":"01CV2V5S4H57KCYFFBS0BJ8K7E","rowCount":1,"sql":"SELECT schedule_cinema_data_task();","stackTrace":["/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist:162:28","/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist:314:12","/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist:361:20","/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist/utilities:17:13","/Users/gajus/Documents/dev/applaudience/data-management-program/src/bin/commands/do-cinema-data-tasks.js:59:21","/Users/gajus/Documents/dev/applaudience/data-management-program/src/bin/commands/do-cinema-data-tasks.js:590:45","internal/process/next_tick.js:68:7"],"values":[]},"message":"query","sequence":4,"time":1540915127833,"version":"1.0.0"}
{"context":{"package":"slonik","namespace":"slonik","logLevel":20,"executionTime":"66 ms","queryId":"01CV2V5SGS0WHJX4GJN09Z3MTB","rowCount":1,"sql":"SELECT cinema_id \"cinemaId\", target_data \"targetData\" FROM cinema_data_task WHERE id = ?","stackTrace":["/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist:162:28","/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist:285:12","/Users/gajus/Documents/dev/applaudience/data-management-program/node_modules/slonik/dist/utilities:17:13","/Users/gajus/Documents/dev/applaudience/data-management-program/src/bin/commands/do-cinema-data-tasks.js:603:26","internal/process/next_tick.js:68:7"],"values":[17953947]},"message":"query","sequence":5,"time":1540915127902,"version":"1.0.0"}

```

Use [`@roarr/cli`](https://github.com/gajus/roarr-cli) to pretty-print the output.

![Log Roarr pretty-print output.](./.README/log-roarr-pretty-print-output.png)


<a name="slonik-syntax-highlighting"></a>
## Syntax highlighting

<a name="slonik-syntax-highlighting-atom"></a>
### Atom

Using [Atom](https://atom.io/) IDE you can leverage the [`language-babel`](https://github.com/gandm/language-babel) package in combination with the [`language-sql`](https://github.com/atom/language-sql) to enable highlighting of the SQL strings in the codebase.

![Syntax highlighting in Atom](./.README/atom-syntax-highlighting.png)

To enable highlighting, you need to:

1. Install `language-babel` and `language-sql` packages.
1. Configure `language-babel` "JavaScript Tagged Template Literal Grammar Extensions" setting to use `language-sql` to highlight template literals with `sql` tag (configuration value: `sql:source.sql`).
1. Use [`sql` helper to construct the queries](https://github.com/gajus/slonik#tagged-template-literals).

For more information, refer to the [JavaScript Tagged Template Literal Grammar Extensions](https://github.com/gandm/language-babel#javascript-tagged-template-literal-grammar-extensions) documentation of `language-babel` package.
