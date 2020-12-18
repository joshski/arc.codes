---
title: Node
description: Node runtime helpers
---

Architect runtime helpers are optional but they do make working with CloudFormation provisioned resources nicer. CloudFormation resources are generated with names more friendly for machines than people. Other frameworks leave resource discovery up to end users which leads to ad hoc implementations becoming a frequent bug vector. Architect treats runtime discovery as a first class concern. 

> Amazon Resource Names (ARNs) are available at runtime to all Lambda functions defined in the same `app.arc`. Things such as DynamoDB tables, SNS topics, SQS queues, API Gateway endpoints, and S3 static bucket ARNs are baked into `@architect/functions` so your runtime program logic interacts with resources using people friendly and readable names defined in the `app.arc` file.

## Setup 

Install the Architect runtime helpers for Node:

```bash 
npm install @architect/functions
```

Ensure `arc` is available to your Lambda function code:

```javascript
let arc = require('@architect/functions')
```

## API

- [`arc.static`](#arc.static) Get a `@static` asset path 
- [`arc.http.async`](#arc.http.async) Middleware for `@http`
- [`arc.http.session`](#arc.http.session) Sessions for `@http` 
- [`arc.tables`](#arc.tables) Generates a DynamoDB client for `@tables`
- [`arc.events`](#arc.events) Publish/subscribe helpers for SNS `@events`
- [`arc.queues`](#arc.queues) Publish/subscribe helpers for SQS `@queues`

### `arc.static`

Get a static asset path:

```javascript
let css = arc.static('/index.css')
```

### `arc.http.async`

Middleware with `async` functions is defined on `arc.http.async` with middleware functions as parameters. The returned function adheres to the expected AWS Lambda function signature. A function can exit the middleware queue early by returning an HTTP response.

```javascript
let arc = require('@architect/functions')

exports.handler = arc.http.async(auth, handler)

async function auth(request) {
  if (!request.session.account) {
    return { status: 403 }
  }
}

async function handler(request) {
  return {
    json: { ok: true }
  }
}
```

#### Request

The incoming request object is [the standard API Gateway request](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html) with a few enhancements:

- `body` is automatically parsed
- `session` automatically parsed from the request cookie

#### Response

Architect honors the standard API Gateway response payload parameters:

- `statusCode`
- `headers`
- `body`
- `isBase64Encoded`

And adds the following clean convenience params:

- `cacheControl` sets the `Cache-Control` header
- `css` sets the `Content-Type` header to `text/css; charset=utf8`
- `code` alias for `statusCode`
- `cors` sets the `Access-Control-Allow-Origin` header to `*`
- `html` sets the `Content-Type` header to `text/html; charset=utf8`
- `js` sets the `Content-Type` header to `text/javascript; charset=utf8`
- `json` sets the `Content-Type` header to `text/html; charset=utf8`
- `session` write a value to the session
- `status` also an alias for `statusCode`
- `text` sets the `Content-Type` header to `text/plain; charset=utf8`
- `type` sets the `Content-Type` header
- `xml` sets the `Content-Type` header to `text/xml; charset=utf8`

### `arc.http.session`

Read the current session in an `@http` request and write it back to a cookie

```javascript
async function handler (req) {
  // read the session
  let session = arc.http.session.read(req)
  // save the session into a cookie string
  let cookie = await arc.http.session.write({ count: 1 })
  // write the cookie to the browser
  return { 
    statusCode: 200,
    headers: { 'set-cookie': cookie },
  }
}
```

### `arc.tables`

Create a DynamoDB client for `@tables`.

Given the following `app.arc` file:

```arc
@app
testapp

@tables
notes
  personID *String
  noteID **String
```

Generate a data access layer:

```javascript
let arc = require('@architect/functions')
let data = await arc.tables()
```

For the example above the generated API is:

- `data.notes.get`
- `data.notes.query`
- `data.notes.scan`
- `data.notes.put`
- `data.notes.delete`
- `data.notes.update`

> The generated client is facade for <code>AWS.DynamoDB.DocumentClient</code> with `TableName` parameter prepopulated. <a href="https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html" target=blank>See the official DynamoDB documentation for all available parameters.</a>

The generated data layer also allows direct access to DynamoDB through a few methods:

- `data._db` which returns an instance of [`AWS.DynamoDB`](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html)
- `data._doc` returns an instance of [`AWS.DynamoDB.DocumentClient`](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html)
- `data._name` helper function that returns a `@table` resource name when you need to go lower level

### `arc.events`

Subscribe to an SNS topic

```javascript
let arc = require('@architect/functions')

exports.handler = arc.events.subscribe(handler)

async function handler (event) {
  console.log(event)
}
```

Publish to an SNS topic

```javascript
let arc = require('@architect/functions')

await arc.events.publish({
  name: 'hit-counter',
  payload: {hits: 1},
})
```

### `arc.queues`

Subscribe to an SQS topic

```javascript
let arc = require('@architect/functions')

exports.handler = arc.queues.subscribe(handler)

async function handler (event) {
  console.log(event)
}
```

Publish to an SNS topic

```javascript
let arc = require('@architect/functions')

await arc.queues.publish({
  name: 'hit-counter',
  payload: {hits: 1},
})
```