RateLimit.js
============
[![Build
Status](https://travis-ci.org/dudleycarr/ratelimit.js.svg)](https://travis-ci.org/dudleycarr/ratelimit.js) [![npm version](https://badge.fury.io/js/ratelimit.js.svg)](http://badge.fury.io/js/ratelimit.js)

A NodeJS library for efficient rate limiting using sliding windows stored in Redis.

Features
--------
* Uses a sliding window for a rate limit rule
* Multiple rules per instance
* Multiple instances of RateLimit side-by-side for different categories of users.
* Includes Express middleware

Background
----------
See this excellent articles on how the sliding window rate limiting with Redis
works:

* [Introduction to Rate Limiting with Redis Part 1](http://www.dr-josiah.com/2014/11/introduction-to-rate-limiting-with.html)
* [Introduction to Rate Limiting with Redis Part 2](http://www.dr-josiah.com/2014/11/introduction-to-rate-limiting-with_26.html)

Install
-------

```
npm install ratelimit.js
```

Usage
-----

Basic example:

```javascript
var RateLimit = require('ratelimit.js').RateLimit;
var redis = require('redis');

var client = redis.createClient();

var rules = [
  {interval: 1, limit: 5},
  {interval: 3600, limit: 1000}
  ];
var limiter = new RateLimit(client, rules);

var showRateLimited = function(err, isRateLimited) {
  if (err) {
    return console.log("Error: " + err);
  }

  console.log("Is rate limited? " + isRateLimited);
};

// Exceed rate limit.
for(var i = 0; i < 10; i++) {
  limiter.incr('127.0.0.1', showRateLimited);
}
```


Output:
```
Is rate limited? false
Is rate limited? false
Is rate limited? false
Is rate limited? false
Is rate limited? false
Is rate limited? true
Is rate limited? true
Is rate limited? true
Is rate limited? true
Is rate limited? true
```

Express Middleware Usage
------------------------

Construct rate limiter and middleware instances:

```javascript
var RateLimit = require('ratelimit.js').RateLimit;
var ExpressMiddleware = require('ratelimit.js').ExpressMiddleware;
var redis = require('redis');

var rateLimiter = new RateLimit(redis.createClient(), [{interval: 1, limit: 10}]);

var options = {
  ignoreRedisErrors: true; // defaults to false
};
var limitMiddleware = new ExpressMiddleware(rateLimiter, options);
```

Rate limit every endpoint of an express application:

```javascript
app.use(limitMiddleware.middleware(function(req, res, next) {
  res.status(429).json({message: 'rate limit exceeded'});
}));
```

Rate limit specific endpoints:

```javascript
var limitEndpoint = limitMiddleware.middleware(function(req, res, next) {
  res.status(429).json({message: 'rate limit exceeded'});
});

app.get('/rate_limited', limitEndpoint, function(req, res, next) {
  // request is not rate limited...
});

app.post('/another_rate_limited', limitEndpoint, function(req, res, next) {
  // request is not rate limited...
});
```

Don't want to deny requests that are rate limited? Not sure why, but go ahead:

```javascript
app.use(limitMiddleware.middleware(function(req, res, next) {
  req.rateLimited = true;
  next();
}));
```

Use a custom IP extraction function:

```javascript
function extractIps(req) {
  return req.ips;
}

app.use(limitMiddleware.middleware(extractIps, function(req, res, next) {
  res.status(429).json({message: 'rate limit exceeded'});
}));
```

Note: this is helpful if your application sits behind a proxy (or set of proxies).
[Read more about express, proxies and req.ips here](http://expressjs.com/guide/behind-proxies.html).

ChangeLog
---------
* **1.3.0**
  * Add options to ExpressMiddleware constructor and support ignoring redis level errors
* **1.2.0**
  * Remove `checkRequest` and `trackRequests` from middleware in favor of single `middleware` function
* **1.1.0**
  * Add Express middleware
  * Updated README
  * Added credits on Lua code
* **1.0.0**
  * Initial RateLimit support

Authors
-------

* [Dudley Carr](https://github.com/dudleycarr)
* [Josh Gummersall](https://github.com/joshgummersall)