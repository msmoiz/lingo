# Lingo

Lingo is a lightweight protocol for interprocess communication. It uses HTTP as
a transport layer and introduces a thin semantic layer on top to structure
application logic. This document specifies the protocol.

## Motivation

HTTP was originally designed to support reading and writing HTML documents in a
browser and its initial features as well as those that have been added since its
creation reflect that primary use case. It has been repurposed to execute
arbitrary requests across process boundaries but often in a manner that results
in a semantic mismatch. This mismatch is described in greater detail [in this
discussion](https://github.com/msmoiz/philosophy?tab=readme-ov-file#application-programming-interfaces).

The REST approach tries to remedy these issues by ascribing application
semantics to various HTTP constructs (e.g., a path represents a specific
application entity), but there is no formal specification which means that no
two people interpret REST the same way.

At the same time, HTTP has accrued a number of useful features related to the
communication itself (authentication flows, encryption, rate limiting,
redirection, caching) that make it hard to beat as a high level transport
protocol. We can save ourselves a great deal of work by leaning on HTTP for
transportation instead of trying to recreate these features on top of TCP
directly.

Lingo strikes a balance between using HTTP for nothing and using it for
everything. It uses HTTP for transportation but not for application logic. The
HTTP transport layer allows us to leverage existing tooling and will be familiar
to many developers. The application layer creates a dedicated space for
application constructs so that developers are not tempted to mix them with HTTP
constructs. It also provides just enough structure to make it easy to put things
in the right place while affording developers the expressive flexibility that
they already have and expect when creating intraprocess operations. For
instance, it draws a clear distinction between the following scenarios which
often get marked with the same error code:

- _The endpoint does not exist_: This is a transportation error and returns a
  HTTP Not Found status code.

- _The entity does not exist_: This is an application error and returns an
  application-specific error code.

It also aims to address some of the shortcomings of existing RPC protocols. For
example, it uses a more efficient representation for operation routing than
JSON-RPC. It also uses a human-readable serialization format (in contrast to
something like gRPC) to make it easy to interact with using standard tools like
curl and Postman or to roll your own clients and servers for unsupported
environments.

## Specification

### Operations

An _operation_ is a function call that crosses a process boundary.

### Medium

An operation is invoked using an HTTP request and the result is returned using
an HTTP response.

### Roles

Lingo is based on the classic client-server model. The client is the process
that invokes an operation and the server is the process that implements it.
There is no restriction in terms of which roles a given process can assume. For
instance, a process might be a client for one operation and a server for
another. In theory, it is even possible for the same process to be both the
client and server for an operation.

### Routing

#### Method

Operations are expressed as POST requests.

> The read/write nature of the operation is conveyed using the operation name
> instead of the HTTP method. The operation input is sent using the request body
> and so we need to use an HTTP method that permits request bodies. This has
> implications for caching since POST requests are typically not meant to be
> cached according to the HTTP spec. Instead, caching needs to be handled in the
> application layer. This is similar to the approach that AWS uses for its APIs.

#### Path

The operation name is expressed using the request path. It should be defined at
the root level. For instance, an operation to update a todo might use the
following path: `/update-todo`.

> We use the request path for a number of reasons. It is more straightforward to
> use than the query string and harder to get wrong (e.g., it is possible to
> specify the operation name multiple times in a query string). It is easier to
> use with existing tooling like server frameworks and load balancers, which
> typically support path-based routing but may not support header-based routing.
> It is also more efficient than including the path in the request body because
> the server and intermediate infrastructure can route without parsing the
> request body (this is a common complaint about JSON-RPC).

### Authentication

The request must include an `Authorization` header with a Bearer token. If the
header is missing or the token is invalid, the server will return an HTTP
Unauthorized status code.

### Authorization

If the caller is authenticated but is not permitted to invoke the operation, the
server will return an HTTP Forbidden status code.

### Input

The operation input is expressed using the request body. The input is structured
as a JSON object and should always be present even when there are no input
arguments. The request should contain a content type header set to
`application/json`.

> JSON is human-readable which makes it easy to work with in a variety of
> settings. It is less performant than a binary representation like Protobuf but
> we are biasing toward developer efficiency. Sending an empty object instead of
> an empty request when there are no input parameters is also less performant
> but simplifies implementation enough to prefer it on the first pass.

### Result

The operation result is expressed using the response body. The result is
structured as a JSON object and should always be present. The result represents
either success or failure and its type is determined by the value of the
`result` field. If it is `ok`, the result represents the success value (the
operation output), and if it is `err`, the result represents the failure value
(the operation error). The output or error data is flattened directly into the
result object alongside the `result` field.

> It is less performant to put the result tag in the response body instead of in
> a header or similar vehicle, but this lines up more closely with the
> semantics of calling an in-memory function in that the discriminant is part of
> the return value, not external to it.

There are no restrictions on the content of the operation output except that it
must be structured as a JSON object and must not include a field named `result`.
The operation error is structured as a JSON object and must include a `code`
field that contains a machine-readable error code. It must similarly not include
a field named `result`. It may include other structured data fields alongside
`code`. The response should contain a content type header set to
`application/json`.

### Status code

The request should return an HTTP OK status code so long as there is no issue in
sending the request or in receiving the response. This is the case even if the
operation result is an error. Any other status code indicates an error at the
transport layer.

### Input errors

An input error can be surfaced as either an HTTP Bad Request status code or as
an application error. The line is drawn using the in-memory test. If the shape
of an input does not match the shape expected by the receiving function, because
it is of the wrong type or is missing fields, a compiler for a statically typed
language would catch the issue; the issue thus only arises in a transport
context and so is treated as a transport error. If the shape is proper, but
there is a semantic issue, a compiler would not catch it; it is treated as an
application error.

> The line is admittedly blurry here and depends in large part upon how you
> choose to handle validation on the server. For instance, if a number field
> must be between 10 and 20, you could validate it in the operation handler
> (application error), but you could also theoretically validate it during
> deserialization from JSON if you define the field as a newtype with validation
> built into its construction (transport error). Neither error is automatically
> retryable though, so in practice both types of errors are likely to be treated
> similarly.

### Internal errors

An internal error is an unexpected failure that occurs during the execution of
an operation. It is not part of the operation's error contract and its details
should not be returned to the caller. The server will return an HTTP Internal
Server Error status code with no response body.

> This is an edge case where the in-memory test fails. There is no good analogue
> for an internal error in the context of an in-memory function call: for the
> most part, you can inspect an error that occurs in the same process, no matter
> how deep the call stack. The concept of an opaque error that you cannot
> inspect is an artifact of a distributed system. Therefore, we treat an
> internal error as a transport error, even in cases where the operation handler
> has been invoked and the transport itself is not at issue.

### Other errors

The following conditions are also treated as transport errors:

- _Unsupported operation:_ If an operation is not supported, it is surfaced as
  an HTTP Not Found status code.
- _Rate limit exceeded:_ If the rate limit has been exceeded, it is surfaced as
  an HTTP Too Many Requests status code.
