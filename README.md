# Limiter

[![Build Status](https://travis-ci.org/ulule/limiter.svg)](https://travis-ci.org/ulule/limiter)

*Dead simple rate limit middleware for Go.*

* Simple API (your grandmother can use it)
* "Store" approach for backend
* Redis support (but not tied too)
* Middlewares: HTTP and [go-json-rest][2]

## Installation

```bash
$ go get github.com/ulule/limiter
```

## Usage

In five steps:

* Create a `limiter.Rate` instance (the number of requests per period)
* Create a `limiter.Store` instance (see [store_redis](https://github.com/ulule/limiter/blob/master/store_redis.go) for Redis or [store_memory](https://github.com/ulule/limiter/blob/master/store_memory.go) for in-memory)
* Create a `limiter.Limiter` instance that takes store and rate instances as arguments
* Create a middleware instance using the middleware of your choice
* Give the limiter instance to your middleware initializer

Example:

```go
// Create a rate with the given limit (number of requests) for the given
// period (a time.Duration of your choice).
rate := limiter.Rate{
    Period: 1 * time.Hour,
    Limit:  int64(1000),
}

// You can also use the simplified format "<limit>-<period>"", with the given
// periods:
//
// * "S": second
// * "M": minute
// * "H": hour
//
// Examples:
//
// * 5 reqs/second: "5-S"
// * 10 reqs/minute: "10-M"
// * 1000 reqs/hour: "1000-H"
//
rate, err := limiter.NewRateFromFormatted("1000-H")
if err != nil {
    panic(err)
}

// Then, create a store. Here, we use the bundled Redis store. Any store
// compliant to limiter.Store interface will do the job.
store, err := limiter.NewRedisStore(pool, "prefix_for_keys")
if err != nil {
    panic(err)
}

// Or use a in-memory store with a goroutine which clear expired keys every 30 seconds
store := limiter.NewMemoryStore("prefix_for_keys", 30*time.Second)

// Then, create the limiter instance which takes the store and the rate as arguments.
// Now, you can give this instance to any supported middleware.
limiterInstance := limiter.NewLimiter(store, rate)
```

See middleware examples:

* [HTTP](https://github.com/ulule/limiter/tree/master/examples/http)
* [go-json-rest](https://github.com/ulule/limiter/tree/master/examples/gjr)

## How it works

The ip address of the request is used as a key in the store.

If the key does not exist in the store we set a default
value with an expiration period.

You will find two stores:

* RedisStore: rely on [TTL](http://redis.io/commands/ttl) and incrementing the rate limit on each request
* MemoryStore: rely on [go-cache](https://github.com/pmylund/go-cache) with a goroutine to clear expired keys using a default interval

When the limit is reached, a ``429`` HTTP code is sent.

## Why Yet Another Package

You could ask us: why yet another rate limit package?

Because existing packages did not suit our needs.

We tried a lot of alternatives:

1. [Throttled][1]. This package uses the generic cell-rate algorithm. To cite the
documentation: *"The algorithm has been slightly modified from its usual form to
support limiting with an additional quantity parameter, such as for limiting the
number of bytes uploaded"*. It is brillant in term of algorithm but
documentation is quite unclear at the moment, we don't need *burst* feature for
now, impossible to get a correct `After-Retry` (when limit exceeds, we can still
make a few requests, because of the max burst) and it only supports ``http.Handler``
middleware (we use [go-json-rest][2]). Currently, we only need to return `429`
and `X-Ratelimit-*` headers for `n reqs/duration`.

2. [Speedbump][3]. Good package but maybe too lightweight. No `Reset` support,
only one middleware for [Gin][4] framework and too Redis-coupled. We rather
prefer to use a "store" approach.

3. [Tollbooth][5]. Good one too but does both too much and too less. It limits by
remote IP, path, methods, custom headers and basic auth usernames... but does not
provide any Redis support (only *in-memory*) and a ready-to-go middleware that sets
`X-Ratelimit-*` headers. `tollbooth.LimitByRequest(limiter, r)` only returns an HTTP
code.

4. [ratelimit][6]. Probably the closer to our needs but, once again, too
lightweight, no middleware available and not active (last commit was in August
2014). Some parts of code (Redis) comes from this project. It should deserve much
more love.

There are other many packages on GitHub but most are either too lightweight, too
old (only support old Go versions) or unmaintained. So that's why we decided to
create yet another one.

## Contributing

* Ping us on twitter [@oibafsellig](https://twitter.com/oibafsellig), [@thoas](https://twitter.com/thoas)
* Fork the [project](https://github.com/ulule/limiter)
* Fix [bugs](https://github.com/ulule/limiter/issues)

Don't hesitate ;)

[1]: https://github.com/throttled/throttled
[2]: https://github.com/ant0ine/go-json-rest
[3]: https://github.com/etcinit/speedbump
[4]: https://github.com/gin-gonic/gin
[5]: https://github.com/didip/tollbooth
[6]: https://github.com/r8k/ratelimit
