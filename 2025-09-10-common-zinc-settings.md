# Common Zinc HTTP resource control settings

Sept 10, 2025

Parties involved in a network protocol should protect themselves from potential abuse by imposing some limits on the requests/responses they accept. Let’s have a look at some common resource control settings in Zinc HTTP Components. Common, because they apply to both clients and servers.

The abuse we are talking about can be described as a form of denial of service: sending too much data or trying to send malformed data to trigger failures. Keeping a connection open for too long or being deliberately slow is also bad behaviour. To deal with these issues a number of maxima are imposed. These help prevent runaway resource usage.

The common resource control settings are:
- maximumEntitySize
- maximumLineLength
- maximumNumberOfDictionaryEntries
- socketStreamTimeout

These are all implemented using the ZnOptions framework, which allows for settings to be changed and controlled at various levels, both statically and dynamically.

An option is defined at the class side of the class `ZnOptions`, returning its global static default value. Options are marked by a special `znOption` pragma.

```smalltalk
ZnOptions class>>#socketStreamTimeout
	"The timeout in seconds for network socket stream operations like connecting, reading and writing."

	<znOption>

	^ 30
```

The access the current applicable value of an option, you use the special `ZnCurrentOptions` dynamic variable.

```smalltalk
ZnCurrentOptions at: #socketStreamTimeout
```

Be aware though that this might return a different value depending on where you evaluate this expression. The values of options could have been overwritten dynamically by code up the call chain. 

Let’s first have a look at how you can change the global default. This is done by accessing the global default. This is a writeable clone of the readonly set of predefined options.

```smalltalk
ZnOptions globalDefault at: #socketStreamTimeout put: 10
```

The above expression sets the timeout for _all_ (new) socket streams to 10 seconds, down from the default 30 seconds. Note that you can only set options that exist, i.e. that are defined on the class side op ZnOptions.

To dynamically change an option you clone the global options, make changes and run a code block in which the new options will be applicable.

```smalltalk
ZnOptions globalDefault clone
	at: #socketStreamTimeout put: 5;
	during: [ ZnNetworkingUtils socketStreamTimeout ]
```

Since we have the following implementation:

```smalltalk
ZnNetworkingUtils class>>socketStreamTimeout
	"Return the timeout in seconds for network socket stream operations like connecting, reading and writing."

	^ ZnCurrentOptions at: #socketStreamTimeout 
```

The expression will return 5. The global default remains the same. The dynamic change overrides the global one, or any earlier ones, but only inside the block.

This allows you to execute code with changed options. To use a low timeout on a single HTTP GET, we can do:

```smalltalk
ZnOptions globalDefault clone
	at: #socketStreamTimeout put: 2;
	during: [ ZnClient new get: 'https://zn.stfx.eu/small' ]
```

To confirm this works, we can introduce an artificial delay.

```smalltalk
ZnOptions globalDefault clone
	at: #socketStreamTimeout put: 2;
	during: [ ZnClient new get: 'https://zn.stfx.eu/echo?delay=3' ]
```

This will result in a `ConnectionTimedOut` exception.

It is also possible to set the timeout on a `ZnClient` or `ZnServer` instance. This way, the timeout holds for all usage of that instance. The implementation is more complicated but follows the same principles.

```smalltalk
ZnClient new
  timeout: 2; 
  get: 'https://zn.stfx.eu/echo?delay=1'
```

Vs. 

```smalltalk
ZnClient new
  timeout: 2;
  get: 'https://zn.stfx.eu/echo?delay=3'
```

The documentation strings of the other settings mentioned earlier include the following descriptions.

**maximumEntitySize** (default 16MB)

The maximum entity size in bytes that can be read from a stream before ZnEntityTooLarge is signalled. Entities are the resources transferred by the HTTP protocol.

**maximumLineLength** (default 4096)

The maximum line length in bytes that can be read from a stream before ZnLineTooLong is signalled. This is applicable to each message's first line and each header line.
		
**maximumNumberOfDictionaryEntries** (default 256)

The maximum number dictionary entries to accept. Used by ZnMultiValueDictionary and thus for reading headers, url query and application form url encoded entity fields.

By now you should know what to do when the following fails with a `ZnEntityTooLarge` exception, taking into account that that file is 21 MB.

```smalltalk
ZnClient new get: 'https://files.pharo.org/image/PharoV20.sources'
```
	