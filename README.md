# httpserver

A zero-dependency implementation of the JDK com.sun.net.httpserver.HttpServer specification with a few significant enhancements.

It adds websocket support using modified source from nanohttpd.

It has basic server-side proxy support using [ProxyHandler](https://github.com/robaho/httpserver/blob/main/src/main/java/robaho/net/httpserver/extras/ProxyHandler.java).

ProxyHandler also supports tunneling proxies using CONNECT for https.

It supports Http2 [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113.html)

All async functionality has been removed. All synchronized blocks were removed in favor of other Java concurrency concepts.

The end result is an implementation that easily integrates with Virtual Threads available in JDK 21 - simply set a virtual thread based ExecutorService.

Improved performance by more than **10x** over the JDK implementation, using http pipelining, optimized String parsing, etc.

Designed for embedding with only a 90kb jar and zero dependencies.

## background

The JDK httpserver implementation has no support for connection upgrades, so it is not possible to add websocket support.

Additionally, the code still has a lot of async - e.g. using SSLEngine to provide SSL support - which makes it more difficult to understand and enhance.

The streams based processing and thread per connection design simplifies the code substantially.

## testing

Nearly all of the tests were included from the JDK so this version should be highly compliant and reliable.

## using

Set the default HttpServer provider when starting the jvm:

<code>-Dcom.sun.net.httpserver.HttpServerProvider=robaho.net.httpserver.DefaultHttpServerProvider</code>

or instantiate the server directly using [this](https://github.com/robaho/httpserver/blob/main/src/main/java/robaho/net/httpserver/DefaultHttpServerProvider.java#L33).

or the service loader will automatically find it when the jar is placed on the class path when using the standard HttpServer service provider.

## performance

** updated 11/22/2024: retested using JDK 23. The results for the JDK version dropped dramatically because I was able to resolve the source of the errors (incorrect network configuration) - and now the robaho version is more than 10x faster.

This version performs more than **10x** better than the JDK version when tested using the [Tech Empower Benchmarks](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/Java/httpserver) on an identical hardware/work setup with the same JDK 23 version.<sup>1</sup>

The frameworks were also tested using [go-wrk](https://github.com/tsliwowicz/go-wrk)<sup>2</sup>

<sup>1</sup>_The robaho version has been submitted to the Tech Empower benchmarks project for 3-party confirmation._<br>
<sup>2</sup>_`go-wrk` does not use http pipelining so, the large number of connections is the limiting factor._

**robaho tech empower**
```
robertengels@macmini go-wrk % wrk -H 'Host: imac' -H 'Accept: text/plain,text/html;q=0.9,application/xhtml+xml;q=0.9,application/xml;q=0.8,*/*;q=0.7' -H 'Connection: keep-alive' --latency -d 60 -c 64 --timeout 8 -t 2 http://imac:8080/plaintext -s ~/pipeline.lua -- 16
Running 1m test @ http://imac:8080/plaintext
  2 threads and 64 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.20ms    9.22ms 404.09ms   85.37%
    Req/Sec   348.78k    33.28k  415.03k    71.46%
  Latency Distribution
     50%    0.98ms
     75%    1.43ms
     90%    0.00us
     99%    0.00us
  41709198 requests in 1.00m, 5.52GB read
Requests/sec: 693983.49
Transfer/sec:     93.98MB
```

**jdk 23 tech empower**
```
robertengels@macmini go-wrk % wrk -H 'Host: imac' -H 'Accept: text/plain,text/html;q=0.9,application/xhtml+xml;q=0.9,application/xml;q=0.8,*/*;q=0.7' -H 'Connection: keep-alive' --latency -d 60 -c 64 --timeout 8 -t 2 http://imac:8080/plaintext -s ~/pipeline.lua -- 16
Running 1m test @ http://imac:8080/plaintext
  2 threads and 64 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.91ms   12.01ms 405.70ms   63.71%
    Req/Sec   114.30k    18.07k  146.91k    87.10%
  Latency Distribution
     50%    4.06ms
     75%    0.00us
     90%    0.00us
     99%    0.00us
  13669748 requests in 1.00m, 1.72GB read
Requests/sec: 227446.87
Transfer/sec:     29.28MB

```

**robaho go-wrk**
```
robertengels@macmini go-wrk % ./go-wrk -c=1024 -d=30 -T=100000 http://imac:8080/plaintext
Running 30s test @ http://imac:8080/plaintext
  1024 goroutine(s) running concurrently
3252278 requests in 30.118280233s, 387.70MB read
Requests/sec:		107983.52
Transfer/sec:		12.87MB
Overall Requests/sec:	105891.53
Overall Transfer/sec:	12.62MB
Fastest Request:	83µs
Avg Req Time:		9.482ms
Slowest Request:	1.415359s
Number of Errors:	0
10%:			286µs
50%:			1.018ms
75%:			1.272ms
99%:			1.436ms
99.9%:			1.441ms
99.9999%:		1.442ms
99.99999%:		1.442ms
stddev:			35.998ms
```

**jdk 23 go-wrk**
```
robertengels@macmini go-wrk % ./go-wrk -c=1024 -d=30 -T=100000 http://imac:8080/plaintext
Running 30s test @ http://imac:8080/plaintext
  1024 goroutine(s) running concurrently
264198 requests in 30.047154195s, 29.73MB read
Requests/sec:		8792.78
Transfer/sec:		1013.23KB
Overall Requests/sec:	8595.99
Overall Transfer/sec:	990.55KB
Fastest Request:	408µs
Avg Req Time:		116.459ms
Slowest Request:	1.930495s
Number of Errors:	0
10%:			1.166ms
50%:			1.595ms
75%:			1.725ms
99%:			1.827ms
99.9%:			1.83ms
99.9999%:		1.83ms
99.99999%:		1.83ms
stddev:			174.373ms

```

## server statistics

The server tracks some basic statistics. To enable the access endpoint `/__stats`, set the system property `robaho.net.httpserver.EnableStatistics=true`.

Sample usage:

```shell
$ curl http://localhost:8080/__stats
Connections: 4264
Active Connections: 2049
Requests: 2669256
Requests/sec: 73719
Handler Exceptions: 0
Socket Exceptions: 0
Mac Connections Exceeded: 0
Idle Closes: 0
Reply Errors: 0
```

The counts can be reset using `/__stats?reset`. The `requests/sec` is calculated from the previous statistics request. 

## maven

```xml
<dependency>
  <groupId>io.github.robaho</groupId>
  <artifactId>httpserver</artifactId>
  <version>1.0.13</version>
</dependency>
```
## enable Http2

Http2 support is enabled via Java system properties.

Use `-Drobaho.net.httpserver.http2OverSSL=true` to enable Http2 only via SSL connections.

Use `-Drobaho.net.httpserver.http2OverNonSSL=true` to enable Http2 on Non-SSL connections (which requires prior knowledge). The Http2 upgrade mechanism was deprecated in RFC 9113 so it is not supported.

See the additional Http2 options in `ServerConfig.java`

The http2 implementation passes all specification tests in [h2spec](https://github.com/summerwind/h2spec)

## Http2 performance

Http2 performance has not yet been optimized, but an unscientific test shows the http2 implementation to have greater than 2x better throughput than the Javalin/Jetty 11 version.

Still, the http2 version is almost 3x slower than the http1 version. I expect this to be the case with most http2 implementations.

The Javalin/Jetty project is available [here](https://github.com/robaho/javalin-http2-example)

<details>
    <summary>performance details</summary>

All tests were run on the same hardware with the same JDK23 version.

Using `h2load -n 1000000 -m 1000 -c 16 [--h1] http://localhost:<port>` 

Jetty 11 http2
```
starting benchmark...
spawning thread #0: 16 total client(s). 1000000 total requests
Application protocol: h2c
finished in 5.25s, 190298.69 req/s, 6.72MB/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1000000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 35.29MB (37003264) total, 7.63MB (8002384) headers (space savings 90.12%), 10.49MB (11000000) data
                     min         max         mean         sd        +/- sd
time for request:      160us     52.24ms      7.76ms      3.94ms    67.73%
time for connect:      235us      8.82ms      4.73ms      2.68ms    62.50%
time to 1st byte:    11.16ms     33.62ms     20.95ms      9.28ms    50.00%
req/s           :   11894.25    12051.63    11957.08       58.94    56.25%
```

Jetty 11 http1
```
starting benchmark...
spawning thread #0: 16 total client(s). 1000000 total requests
Application protocol: http/1.1
finished in 3.67s, 272138.02 req/s, 35.56MB/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1000000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 130.65MB (137000000) total, 86.78MB (91000000) headers (space savings 0.00%), 10.49MB (11000000) data
                     min         max         mean         sd        +/- sd
time for request:      831us    189.78ms     57.30ms     21.98ms    71.20%
time for connect:      152us      4.21ms      2.19ms      1.24ms    62.50%
time to 1st byte:     4.85ms     11.73ms      7.11ms      2.29ms    81.25%
req/s           :   17010.42    17843.23    17334.96      260.43    50.00%
```

robaho http2
```
starting benchmark...
spawning thread #0: 16 total client(s). 1000000 total requests
Application protocol: h2c
finished in 2.20s, 453632.21 req/s, 19.04MB/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1000000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 41.96MB (44000480) total, 5.72MB (6000000) headers (space savings 76.92%), 10.49MB (11000000) data
                     min         max         mean         sd        +/- sd
time for request:      347us     51.17ms     16.98ms     10.52ms    59.21%
time for connect:      228us      8.77ms      4.02ms      2.44ms    62.50%
time to 1st byte:     9.46ms     22.61ms     12.61ms      4.81ms    81.25%
req/s           :   28353.29    29288.55    28542.35      229.27    87.50%
```

robaho http1
```
starting benchmark...
spawning thread #0: 16 total client(s). 1000000 total requests
Application protocol: http/1.1
finished in 802.36ms, 1246317.13 req/s, 103.41MB/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1001066 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 82.97MB (87000000) total, 46.73MB (49000000) headers (space savings 0.00%), 10.49MB (11000000) data
                     min         max         mean         sd        +/- sd
time for request:      860us     35.46ms     12.61ms      3.33ms    75.21%
time for connect:       92us      4.06ms      2.06ms      1.21ms    62.50%
time to 1st byte:     4.68ms     18.67ms     10.85ms      4.88ms    50.00%
req/s           :   77913.01    80438.10    78458.60      721.68    81.25%
```

</details>
