# Zinc HTTP server settings

Zinc's HTTP server framework, `ZnServer` and its subclasses, 
works fine out of the box, with sensible defaults.
It is however useful to know about a number of specific HTTP server settings,
on top of [Common Zinc HTTP resource control settings](2025-09-10-common-zinc-settings.md).

A server needs to be configured to operate.
The HTTP protocol is handled by the Zinc framework:
it deals with sockets, networking, multiprocessing, logging, etc.
Once the server has accepted a request, it creates a `ZnRequest` object.
It then turns to its `delegate` whose task it is to return a `ZnResponse`
that will be send back to the client.

A delegate is any object that implements `#handleRequest:`.
The class `ZnValueDelegate` can be used to convert any block into a delegate.
The helper method `ZnSingleThreadedServer>>#onRequestRespond:` makes this even more convenient.

When you do not set a delegate yourself, an instance of `ZnDefaultServerDelegate` will be used.
This makes sure that there is something to see when you access the server.
It also contains several examples and useful testing and debugging methods.
It acts as a simple router: it maps URLs to handling code.
You can add to this router using `ZnDefaultServerDelegate>>#map:to:`.

You can start multiple servers, listening on different ports,
but to aid scripting, there is a default server that is managed and registered,
and that can be accessed via `ZnServer default`.

```smalltalk
ZnServer startDefaultOn: 1701.
```

You can access the server at [http://localhost:1701](http://localhost:1701).
Now, let's add a route.

```smalltalk
ZnServer default delegate 
  map: #adder to: [ :request | | x y sum |
    x := (request uri queryAt: #x) asNumber.
    y := (request uri queryAt: #y) asNumber.
    sum := x + y.
    ZnResponse ok: (ZnEntity textCRLF: sum asString) ].
```

This creates a new route `/adder` and handler that will take 2 query arguments, 
converts them to numbers and returns the result of adding them together.
We can test our new functionality using `ZnClient`.

```smalltalk
ZnClient new 
  url: ZnServer default localUrl; 
  addPathSegment: #adder;
  queryAt: #x put: 1;
  queryAt: #y put: 2;
  get.
```

This builds an appropriate request to `/adder` and executes it. 
By entering the proper URL directly, this becomes a one liner.

```smalltalk
'http://localhost:1701/adder?x=1&y=2' asUrl retrieveContents.
```

Or you can use CURL on the command line.

```console
$ curl "http://localhost:1701/adder?x=111&y=222"
333
```

Instead of adding a route to the map of the default server delegate,
you could also replace the whole delegate.

```smalltalk
ZnServer default 
  onRequestRespond: [ :request | | x y sum |
    x := (request uri queryAt: #x) asNumber.
    y := (request uri queryAt: #y) asNumber.
    sum := x + y.
    ZnResponse ok: (ZnEntity textCRLF: sum asString) ].
```

This is only good for testing, since the same response is given for any request path.

During the execution of `handleRequest:`, in its dynamic scope, 
two useful dynamic and process specific variables are available.
`ZnCurrentServer` holds a reference to the current `ZnServer` instance.
`ZnCurrentServerSession` holds a reference to the current `ZnServerSession` instance.
These sessions are created lazily.
You can use `server` and `session` accessors on the request and response objects as a convenience.

Creating and setting your own delegate is a more advanced option,
since you have to implement some form of routing and dispatching,
as well as error handling.

When a server starts, it optionally sends `start` to its delegate,
see `ZnSingleThreadedServer>>#startDelegate`.
When a server stops, it optionally sends `close` to its delegate, 
see `ZnSingleThreadedServer>>#closeDelegate`.

Sometimes, when you are several levels deep in handling a request 
you want to immediately return with a response.
For this, there is `ZnRespond`, a notification to signal the end of `handleRequest:` 
processing with a specific `ZnResponse`, earlier than normal stack unwinding.
Here is an example:

```smalltalk
ZnRespond signalWith: ZnResponse unauthorized
```

When using the delegate, the server protects itself from errors that might result from calling `handleRequest:`.
The normal behaviour is to catch this error, log it and return a 500 Internal Server Error.
With the `debugMode` option, you can force a debugger to pop up that allows you to inspect the full stack in detail.

Normally, the server socket binds to all interfaces of the machine it runs on.
That means that everybody on the same network can access the server.
You might want to restrict access, for example to allow only access from the local machine itself.
This can be done by setting the `bindingAddress`.

```smalltalk
(server := ZnServer on: 8080) 
  bindingAddress: NetNameResolver loopBackAddress;
  start.
```

One way to add access protection to a server is by setting the `authenticator`,
an object that implements `authenticateRequest:do:`.
When authentication succeeds, the block should be executed,
when authentication fails, an appropriate response should be returned.
An implementation is available in `ZnBasicAuthenticator`.

As a multihtreaded/multiprocessing server `ZnServer` has to manage its load,
as in how many clients connect at the same time.
HTTP/1.1 keeps connections open as do WebSockets and a server's resources are limited.
There are several settings that are applicable, the key one being `maximumNumberOfConcurrentConnections`.
The default behaviour is to accept new connections up to this number, 32 by default,
after which a 503 Service Unavailable response will be sent and the connection will be closed.

Related to this is `listenBacklogSize`, also 32 by default,
which defines the number of pending connections queued up, 
waiting to be accepted, at any one time.
This is at the socket/networking level of the operating system.

An advanced, experimental option is `acceptStrategy` which can have two values: unlimited and limited.

The `unlimited` accept strategy is the default and works as described above:
when the maximumNumberOfConcurrentConnections is reached, excess connections are rejected, 
but by briefly accepting them, sending a 503 Service Unavailable response and then closing the connection.

The `limited` accept strategy is different: once maximumNumberOfConcurrentConnections is reached,
no further connections are accepted, normally no 503 Service Unavailable response is ever sent.
What happens then, is that the backlog queue of the server socket at the OS level fills up
until listenBacklogSize are waiting to be accepted, after which connections at the OS level are rejected.
This option can be useful to create backpressure against upstream proxies or load balancers.
