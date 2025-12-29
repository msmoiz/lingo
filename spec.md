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
communication itself (authentication flows, rate limiting, redirection,
caching) that make it hard to beat as a high level transport protocol. We can
save ourselves a great deal of work by leaning on HTTP for transportation
instead of trying to recreate these features on top of TCP directly.

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
  HTTP Not Found error code.

- _The entity does not exist_: This is an application error and returns an
  application-specific error code.

It also aims to address some of the shortcomings of existing RPC protocols. For
example, it uses a more efficient representation for operation names and error
flags than JSON-RPC. It also uses a human readable serialization format (in
contrast to something like gRPC) to make it easy to interact with using standard
tools like curl and Postman or to roll your own clients and servers for
unsupported environments.
