# HTTP without networking or multiprocessing

We all know that [HTTP](https://en.wikipedia.org/wiki/HTTP) is a networking protocol
with request-response interactions between a client and a server.
Since there can be multiple clients for each server,
multiprocessing is involved to handle concurrent operations.

When testing a web service like a REST API, you set up your server
with its configuration and all its dependencies like a database 
and then use a client to issue a request and validate the response.

This is a functional test: you are testing the full stack.
Which is OK, but sometimes you do not need to test the HTTP stack,
in the assumption that it already works.
It might be faster to bypass the HTTP stack itself,
to skip the networking part.
For debugging, the multiprocessing involved might be annoying,
since most application errors occur outside the thread running the test.

So could we somehow do HTTP tests without a client or a server,
without networking and without multiprocessing ?
Of course we can !
As an object oriented framework modelling all the main HTTP concepts,
[Zinc HTTP Components](https://zn.stfx.eu) is ready to do so.

A request is an HTTP message object that is put on the wire by the client 
and retrieved from the wire by the server.
Similarly, the response flows back from the server to the client.
Internally, a server has a delegate which gets sent the request and answer the response.
Out of the box, an object called `ZnDefaultServerDelegate` is used as delegate,
but in advanced projects you will most probably end up with your own custom delegate.

To bypass the client, server, network and multiprocessing we manually create a request object,
give it directly to our delegate to handle and validate the response.

Let's test [/random/128](https://zn.stfx.eu/random/128), part of the default setup.

```smalltalk
testRandom
  | request response random |
  request := ZnRequest get: '/random/128'.
  response := ZnDefaultServerDelegate new handleRequest: request.
  self assert: response isSuccess.
  random := response contents.
  self assert: random size equals: 128.
  "make sure we get a different response"
  response := ZnDefaultServerDelegate new handleRequest: request.
  self deny: response contents equals: random
```

This works for more complex requests, like those with a payload,
like when submitting the form at [/form-test-2](https://zn.stfx.eu/form-test-2).

```smalltalk
testFormTest2
  | input entity request response contents |
  input := 'contents-' , 999 atRandom asString.
  request := ZnRequest post: '/form-test-2'.
  entity := ZnApplicationFormUrlEncodedEntity new.
  entity at: 'input' put: input.
  request entity: entity.
  response := ZnDefaultServerDelegate new handleRequest: request.
  self assert: response isSuccess.
  contents := response contents.
  self assert: (contents includesSubstring: input).
```

By using the delegate without an actual server,
you cannot reference the server (as there is none).
You might need a full stack test, 
but often there are ways to improve the situation by doing more setup.
For example, here is a way to inject a hostname into the request.

```smalltalk
testGet
  | request response contents |
  "specify a full URL to set the host header"
  request := ZnRequest get: 'https://ci.example.com:8080/echo?test=true'.
  response := ZnDefaultServerDelegate new handleRequest: request.
  self assert: response isSuccess.
  contents := response contents.
  self assert: (contents includesSubstring: 'ci.example.com:8080').
  self assert: (contents includesSubstring: 'test=true').
```

In the end, it is up to you to decide if this is useful in the context of your projects.
Both `ZnClient` and `ZnServer` offer lots of functionality that you might have to manually replicate,
at some point this becomes annoying.
