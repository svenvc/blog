# The complexity of HTTP Benchmarking

Sept 5, 2025

From an engineering standpoint, we should benchmark our HTTP server setup to evaluate our architectural decisions and the parameters that we are using. To measure is to know and all that.

Yes, but HTTP Benchmarking (or any benchmarking really) is quite complex and many aspects influence the results that we observe.

The setup at https://zn.stfx.eu offers a nice framework to highlight the serious variability depending on what we are benchmarking and how we do our benchmarking.

Be warned though: you are about to enter a rabbit hole.

We are going to use `ab`, the Apache HTTP server benchmarking tool. You can experiment yourself against the demo server or a local server in your image. Here is the full output of a single run:

```console
$ ab -k -n 1024 -c 16 https://zn.stfx.eu/small 
This is ApacheBench, Version 2.3 <$Revision: 1913912 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking zn.stfx.eu (be patient)
Completed 102 requests
Completed 204 requests
Completed 306 requests
Completed 408 requests
Completed 510 requests
Completed 612 requests
Completed 714 requests
Completed 816 requests
Completed 918 requests
Completed 1020 requests
Finished 1024 requests


Server Software:        nginx/1.24.0
Server Hostname:        zn.stfx.eu
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-ECDSA-CHACHA20-POLY1305,256,256
Server Temp Key:        ECDH X25519 253 bits
TLS Server Name:        zn.stfx.eu

Document Path:          /small
Document Length:        521 bytes

Concurrency Level:      16
Time taken for tests:   2.101 seconds
Complete requests:      1024
Failed requests:        0
Keep-Alive requests:    1024
Total transferred:      758784 bytes
HTML transferred:       533504 bytes
Requests per second:    487.46 [#/sec] (mean)
Time per request:       32.823 [ms] (mean)
Time per request:       2.051 [ms] (mean, across all concurrent requests)
Transfer rate:          352.74 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2  16.6      0     151
Processing:    18   28   3.3     28      45
Waiting:       18   28   3.3     28      45
Total:         18   30  16.9     28     181

Percentage of the requests served within a certain time (ms)
  50%     28
  66%     29
  75%     30
  80%     31
  90%     33
  95%     34
  98%     42
  99%    154
 100%    181 (longest request)
```

The **basic invocation** is to do 1024 requests with a concurrency level of 16. This simulates 16 HTTP clients each doing 64 requests. Note that we explicitly ask for HTTP/1.1 keepalive to be turned on. That means that each client should open one connection and use it for all its 64 requests.

Before looking at the results it is important to make sure all requests completed successfully and have the content length that we expect. All requests should be keepalive.

The key numbers are requests per second and the time per request. Here we get about 480 reqs/s and 32 ms/req. That is a good, acceptable result for a single low end server running a dynamic language across the internet over a secure connection.

Let’s see how our numbers change when measuring different things differently.

Reducing **the concurrency level** has a big impact on overall throughput, while the time per request remains the same. The relation between these two numbers is almost linear. On the other hand, a lower concurrency level puts less stress on the server.

```console
$ ab -k -n 1024 -c 4 https://zn.stfx.eu/small
Requests per second:    134.33 [#/sec] (mean)
Time per request:       29.776 [ms] (mean)
```

Not using **keepalive** totally kills performance.

```console
$ ab -n 1024 -c 16 https://zn.stfx.eu/small
Requests per second:    34.72 [#/sec] (mean)
Time per request:       460.844 [ms] (mean)
```

The **size of the document** being transferred impacts time on the wire. A 16 bytes document is faster than a 16Kb document, but not that much.

```console
$ ab -k -n 1024 -c 16 https://zn.stfx.eu/bytes/16
Requests per second:    506.10 [#/sec] (mean)
Time per request:       31.614 [ms] (mean)
```

```console
$ ab -k -n 1024 -c 16 https://zn.stfx.eu/bytes/16384
Requests per second:    386.36 [#/sec] (mean)
Time per request:       41.412 [ms] (mean)
```

There is also a difference between a **dynamic** and a **static** document. Our original request https://zn.stfx.eu/small runs a bit of Smalltalk code that dynamically builds an HTML document.

```smalltalk
small: request
	"Answer a small HTML page"

	| page |
	page := self brandedPage: 'Small' do: [ :html |
		html tag: #p with: 'This is a small HTML document' ].
	^ ZnResponse ok: (ZnEntity html: page)
```

The Zinc server demo also contains a static file server with a similar document called `small.html` that is read from a file with the same name.

```console
$ ab -k -n 1024 -c 16 https://zn.stfx.eu/zn/small.html
Requests per second:    510.91 [#/sec] (mean)
Time per request:       31.316 [ms] (mean)
```

We can also try to measure the cost of **proxying** by comparing `small.html` being served directly by nginx vs. zinc serving that file as above.

```console
$ ab -k -n 1024 -c 16 https://stfx.eu/small.html
Requests per second:    520.56 [#/sec] (mean)
Time per request:       30.736 [ms] (mean)
```

These last two variations make only a small difference, because the request handling of the back end Zinc server is relatively fast, as we will see later on, but also because our document is, well, small (about 500 bytes).

The following URLs can be used for a similar test on a larger document (about 8Kb):
- https://zn.stfx.eu/dw-bench
- https://zn.stfx.eu/zn/st-bench.html
- https://stfx.eu/st-bench.html

Now, let’s change the **network** being used. Up until now, the numbers shown were from my home office machine, over a wireless network, residential cable internet maxing out at 300 Mbps down, 30 Mbps up in Belgium to the server which happens to be located in the Netherlands.

Here is a measurement from a cheap shared virtual private server in Germany.

```console
$ ab -k -n 1024 -c 16 https://stfx.eu/small.html
Requests per second:    788.91 [#/sec] (mean)
Time per request:       20.281 [ms] (mean)
```

Switching to a more professional setup with a strong symmetrical internet feed, though running on older hardware and inside a container, the numbers improve again.

```console
$ ab -k -n 1024 -c 16 https://stfx.eu/small.html
Requests per second:    1163.28 [#/sec] (mean)
Time per request:       13.754 [ms] (mean)
```

Now let’s log in to the server itself and run our benchmark there, first using the external URL that we used before.

```console
$ ab -k -n 1024 -c 16 https://stfx.eu/small.html
Requests per second:    1138.93 [#/sec] (mean)
Time per request:       14.048 [ms] (mean)
```

That result seems equivalent to our result from the machine using the better internet feed. The fact that the benchmarking tool with its concurrent clients runs on the same machine as the server probably has an influence too, especially since it is basically a one core machine.

Now that we are on the machine, we can also access the backend zinc server directly, bypassing the proxying and secure connection, i.e. bypassing nginx.

```console
$ ab -k -n 1024 -c 16 http://localhost:8180/small
Requests per second:    3162.00 [#/sec] (mean)
Time per request:       5.060 [ms] (mean)
```

Impressive. But remember that the zinc demo is deployed on pretty basic **hardware** : a VM with 1 virtual CPU core and 1 Gb memory. That is enough for its intended purpose but hardly powerful.

My development machine has an Apple Silicon M2 Pro chip and 16Gb memory, with 10 cores and a clock speed of up to 3.5 GHz. The local results are quite impressive.

```console
$ ab -k -n 1024 -c 16 http://localhost:8180/small
Requests per second:    7531.07 [#/sec] (mean)
Time per request:       2.125 [ms] (mean)
``` 

If there is one thing to remember from all this, it is that you should be really careful when doing benchmarks and be aware of many variable or dimensions.
