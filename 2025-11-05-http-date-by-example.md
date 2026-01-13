# Formatting and parsing HTTP dates by example

HTTP dates as used in header field values have a bit of a particular syntax. Zinc HTTP Components can format and parse HTTP dates. There is however also a way to handle this format ‘by example’.

In the [HTTP](https://en.wikipedia.org/wiki/HTTP) protocol, [header fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields) play an important role in describing requests or responses. Some common headers have as value a timestamp (date time). These are using the [HTTP date format](https://datatracker.ietf.org/doc/html/rfc9110#section-5.6.7).

A typical HTTP response includes a `Date` header that timestamps the message.

```console
$ curl -D - https://zn.stfx.eu/zn/small.html
HTTP/2 200 
server: nginx/1.24.0 (Ubuntu)
date: Wed, 05 Nov 2025 10:47:24 GMT
content-type: text/html;charset=utf-8
content-length: 113
modification-date: Tue, 11 Dec 2018 11:56:31 GMT
x-server: Zinc HTTP Components 2.0 (Pharo/13.1)

<html>
<head><title>Small</title></head>
<body><h1>Small</h1><p>This is a small HTML document</p></body>
</html>
```

In [Zinc HTTP Components](https://zn.stfx.eu) there are two helper methods to deal with this format.

```smalltalk
ZnUtils parseHttpDate: 'Wed, 05 Nov 2025 10:47:24 GMT'.

  "2025-11-05T10:47:24+00:00"

ZnUtils httpDate: DateAndTime now. 

  "'Wed, 05 Nov 2025 10:53:05 GMT'"
```

So far, so good. The implementations in `ZnUtils` are ad hoc and written specifically as not to depend on anything else.

More than 10 years ago I created a project called [ZTimestamp](https://github.com/svenvc/ztimestamp). It was written a bit ‘in anger’, as an alternative to the more complex and less efficient `DateAndTime`. ZTimestamps are always in the UTC/GMT/Zulu timezone and are more ISO oriented, but still compatible with `DateAndTime`. An explicit `ZTimezone` helps in conversions when dealing with user’s timezones.

There is also `ZTimestampFormat`, which is a formatter and parser whose format you configure by providing an example of how your want a particular reference to look like. Given that we already have the existing implementation in ZnUtils, we can use that to come up with the necessary example.

```smalltalk
ZTimestampFormat reference24.

  "2001-02-03T16:05:06Z"

spec := ZnUtils httpDate: ZTimestampFormat reference24. 

  "'Sat, 03 Feb 2001 16:05:06 GMT'"
```

The magic is that the reference numbers its components, 1 for the year, 2 for the month, and so on. Check the class comment for the details.

The `spec` above is the reference as formatted by `‌ZnUtils class>>#httpDate:`. The numbered components, or their particular equivalent values like ‘Wed’ or ‘Nov’ make up the example or specification of the format.

We now have a working formatter and parser!

```smalltalk
(ZTimestampFormat fromString: spec) format: ZTimestamp now. 

  "'Wed, 05 Nov 2025 12:14:09 GMT'"

(ZTimestampFormat fromString: spec) parse: ZnUtils httpDate. 

  "2025-11-05T12:14:13Z"
```

Since this blog post, `ZTimestampFormat class>>#rfc9110` was added as a predefined format.

Finally, here is some proof of concept code for the 2 obsolete formats, `rfc850` and `ansiC`, which were already predefined earlier. Especially rfc850 is problematic because it uses only 2 digits for years. Hence the rule ‘Recipients of a timestamp value in rfc850-date format, which uses a two-digit year, MUST interpret a timestamp that appears to be more than 50 years in the future as representing the most recent year in the past that had the same last two digits’.

The code at the end handles this situation, exposing some of the internals as just subtracting 100 years is **not** the same due to leap years.

```smalltalk
t := ZTimestampFormat rfc850 parse: 'Sunday, 06-Nov-94 08:49:37 GMT'.
ZTimestampFormat ansiC parse: 'Sun Nov  6 08:49:37 1994'.

((t - ZTimestamp now) > 50 years)	
  ifTrue: [
    t dayMonthYearDo: [ :d :m :y |
      ZTimestamp new 
        jdn: (ZTimestamp jdnFromYear: y - 100 month: m day: d) 
        ns: t nanosecondsSinceMidnight ] ].
```
