# Request

```text
POST /update-todo HTTP/1.1
Host: example.com
User-Agent: reqwest
Accept: */*
Content-Type: application/json
Content-Length: <variable>
Authorization: Bearer <key>
{
    "todo_id": 12345,
    "text": "Come up with a good wire protocol"
}
```

# Response (success)

```text
HTTP/1.1 200 OK
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Content-Type: application/json
Content-Length: <variable>
Request-Id: 333
{
    "result": "ok"
}
```

# Response (app failure, general)

```text
HTTP/1.1 200 OK
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Content-Type: application/json
Content-Length: <variable>
Request-Id: 333
{
    "result": "err",
    "code": "todo_not_found",
    "todo_id": 12345
}
```

# Response (transport failure, invalid input)

```text
HTTP/1.1 400 Bad Request
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Content-Type: application/json
Content-Length: <variable>
Request-Id: 333
{
    "message": "missing field 'name'"
}
```

# Response (transport failure, missing token)

```text
HTTP/1.1 401 Unauthorized
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Request-Id: 333
```

# Response (transport failure, not permitted to use endpoint)

```text
HTTP/1.1 403 Forbidden
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Request-Id: 333
```

# Response (transport failure, operation not found)

```text
HTTP/1.1 404 Not Found
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Request-Id: 333
```

# Response (transport failure, throttle)

```text
HTTP/1.1 429 Too Many Requests
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Request-Id: 333
```

# Response (transport failure, internal)

```text
HTTP/1.1 500 Internal Server Error
Server: axum
Date: Fri, 6 Jun 2026 12:12:00 GMT
Request-Id: 333
```
