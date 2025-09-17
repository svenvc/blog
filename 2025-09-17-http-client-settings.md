# Zinc HTTP client settings

Sept 17, 2025

Making an HTTP client request with `ZnClient` works out of the box, with sensible defaults. It is however useful to know about a number of HTTP client settings.

Modern HTTP defaults to keep alive its connections. Consider the following example.

```smalltalk
| client result |
ZnLogEvent logToTranscript.
client := ZnClient new.
result := (1 to: 4) collect: [ :id |
  client get: 'https://zn.stfx.eu/echo?id=' , id asString ].
client close.
ZnLogEvent stopLoggingToTranscript.
result.
```

Which produces this output on the Transcript.

```
2025-09-17 14:05:02 035 677464 Connection Established zn.stfx.eu:443 146.185.177.20:443 97ms
2025-09-17 14:05:02 036 677464 Request Written a ZnRequest(GET /echo?id=1) 2ms
2025-09-17 14:05:02 037 677464 Response Read a ZnResponse(200 OK text/plain;charset=utf-8 297B) 29ms
2025-09-17 14:05:02 038 677464 GET /echo?id=1 200 297B 31ms
2025-09-17 14:05:02 039 677464 Request Written a ZnRequest(GET /echo?id=2) 0ms
2025-09-17 14:05:02 040 677464 Response Read a ZnResponse(200 OK text/plain;charset=utf-8 297B) 30ms
2025-09-17 14:05:02 041 677464 GET /echo?id=2 200 297B 30ms
2025-09-17 14:05:02 042 677464 Request Written a ZnRequest(GET /echo?id=3) 0ms
2025-09-17 14:05:02 043 677464 Response Read a ZnResponse(200 OK text/plain;charset=utf-8 297B) 23ms
2025-09-17 14:05:02 044 677464 GET /echo?id=3 200 297B 23ms
2025-09-17 14:05:02 045 677464 Request Written a ZnRequest(GET /echo?id=4) 0ms
2025-09-17 14:05:02 046 677464 Response Read a ZnResponse(200 OK text/plain;charset=utf-8 297B) 23ms
2025-09-17 14:05:02 047 677464 GET /echo?id=4 200 297B 23ms
2025-09-17 14:05:02 048 677464 Connection Closed 146.185.177.20:443
```

You can clearly see that all 4 request/response cycles are bracketed by just one connection established/closed event. This is done because opening a connection has a cost and reusing an existing connection is more efficient. As a side note, this only works when the request all go to the same host and port using the same scheme.

But what if you know up front that you need to make only one request/response ? If you are lazy, you let garbage collection and finalisation clean up for you. The better solution is to use the `beOneShot` option.

```smalltalk
ZnLogEvent logToTranscript.
ZnClient new
  beOneShot;
  get: 'https://zn.stfx.eu/random'.
ZnLogEvent stopLoggingToTranscript.
```

Which produces this log output.

```
2025-09-17 14:16:25 049 677464 Connection Established zn.stfx.eu:443 146.185.177.20:443 93ms
2025-09-17 14:16:25 050 677464 Request Written a ZnRequest(GET /random) 2ms
2025-09-17 14:16:26 051 677464 Response Read a ZnResponse(200 OK text/plain;charset=utf-8 64B) 29ms
2025-09-17 14:16:26 052 677464 GET /random 200 64B 31ms
2025-09-17 14:16:26 053 677464 Connection Closed 146.185.177.20:443
```

So even though we did not manually close the client, this happened automatically and immediately. Furthermore, the server was informed of this as well by way of a `Connect: close` header in the request, so the other side also closed the connection right after sending the response. Nice!

As a protocol, HTTP might give you an answer, but not the answer that you expect. Say you want to fetch a text file with some numbers from a server. If the file was renamed or removed from the server, you will get a Not found response instead of the string with numbers. It would be as if you mistyped the name. Another possible issue is that you request say an image, but get an HTML document back.

The next script shows how to make requests with a higher reliability.

```smalltalk
ZnClient new
  systemPolicy;
  accept: ZnMimeType textPlain;
  get: 'https://zn.stfx.eu/zn/numbers.txt'.
```

The `systemPolicy` option does 3 things:
- it sets the `enforceHttpSuccess` option to true, meaning non successful responses will trigger an exception
- it set the `enforceAcceptContentType` option to true, meaning that any response with a content type incompatible with what we expect will trigger an exception
- it sets the `numberOfRetries` to 2, so it tries again after the first two failures related to networking, possibly handling transient network errors

With the `accept` option, we tell the server what we expect, but this option is also used to validate the response when `enforceAcceptContentType` is set to true.
 
Regarding transient network errors, it is instructive to look at the configuration of the client used in the unit tests.
 
```smalltalk
ZnClientTest>>#httpClient
  ^ ZnClient new
      timeout:  self class timeout;
      numberOfRetries: self class numberOfRetries;
      retryDelay: self class retryDelay;
      yourself
 ```

This code adds specific controls of the `timeout`, the `numberOfRetries` and the `retryDelay`, with the goal of overcoming slowdowns coming from either the client side, the server side or the network - in the hope that these were transient problems.

Often you use an HTTP client to interact with a JSON REST api. The option `forJsonREST` helps us to set up things with one message send. Here is how you use it against a service that echoes your GET request as a JSON structure.

```smalltalk
ZnClient new
  forJsonREST;
  beOneShot;
  systemPolicy;
  url: 'https://postman-echo.com/get';
  queryAt: #id put: #pharo; 
  get.
```

This same service can also handle POST requests at a different endpoint. Notice how the conversion from Smalltalk to JSON is done automatically as well, as it is done when JSON is received.

```smalltalk
ZnClient new
  forJsonREST;
  beOneShot;
  systemPolicy;
  url: 'https://postman-echo.com/post';
  contents: { #id -> 'Pharo Smalltalk' } asDictionary;
  post.
```

To conclude, I want to point out two authentication options that might be useful.
- `ZnClient>>#setBasicAuthenticationUsername:password:`
- `ZnClient>>#setBearerAuthentication:`

These allow you to tackle the most commonly used authentication systems used by web services.
