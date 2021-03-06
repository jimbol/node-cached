# cached

A simple caching library, inspired by the [Play cache API](http://www.playframework.com/documentation/2.2.x/ScalaCache) and biased towards [showing stale data instead of dog piling](http://highscalability.com/strategy-break-memcache-dog-pile).
The interface only exposes very limited functionality, there's no multi-get or deletion of cached data.
The library is designed to support different caching backends, though right now only memcached is implemented.

It supports both promise- and callback-based usage.

## Install

`npm install --save cached`

## Usage

More detailed API docs are in the next section.

### Getting and setting

```js
cached = require('cached');

kittens = cached('kittens');

// Set a key using a plain value
kittens.set('my.key', 'Hello World');

// Set a key using a lazily created promise (or value)
kittens.set('my.key', function() {
  return cache.get('other.key');
});

// Set a key using a callback-style function
kittens.set('my.key', cached.deferred(function(done) {
  done(null, 'Hello World');
}));

kittens.getOrElse('my.key', function() {
  // This will store "Hello World" for key "my.key" if
  // "my.key" was not found
  return 'Hello World';
}).then(function(data) {
  console.log(data);
});

kittens.get('my.key', function(err, data) {
  // Handle it the callback way
});

// Handle it the promise way
kittens.get('my.key').then(
  function(data) {
    console.log(data);
  },
  function(err) {
    throw err;
  }
);
```

## Supported backends

### Memcached

A thin wrapper around [memcached](https://github.com/3rd-Eden/node-memcached).
You can either provide a readily configured client or a combination of hosts and additional options.
Without any additional options it will default to a local memcached on `11211`.

#### Custom client instance

```js
var Memcached = require('memcached');

cached('myStuff', { backend: {
  type: 'memcached',
  client: new Memcached('192.168.0.102:11212', { poolSize: 15 })
}});
```

#### Let cached create the instance

This will create the same cache as above.

```js
cached('myStuff', { backend: {
  type: 'memcached',
  hosts: '192.168.0.102:11212',
  poolSize: 15
}});
```

## API

### cached(name: string, options) -> Cache

Creates a new named cache or returns a previously initialized cache.

* **name:** A meaningful name for what is in the cache, default: `"default"`. This will also be used as a key-prefix. If the name is `"cars"`, all keys will be prefixed with `"cars:"`
* **options:**
  * **backend:** An object that has at least a `type` property. If no backend is configured, the cache will run in "noop"-mode, not caching anything. All other properties are forwarded to the backend, see [using different backends](#supported-backends) for which backend types exist and what options they support.
  * **defaults:** Defaults to apply for all cache operations. See `Cache.setDefaults`

### cached.createCache(options) -> Cache

This allows you to circumvent the global named caches. The options are the same as above, just `name` is also part of the `options` object when using this function.

### cached.dropNamedCache(name: string) -> cached

Drop the given named cache.

### cached.dropNamedCaches() -> cached

Drop all named caches.

### cached.deferred(fn) -> () -> Promise

Convert a node-style function that takes a callback as its first parameter into a parameterless function that generates a promise.
In other words: this is what you'd want to wrap your node-style functions in when using them as value arguments to `set` or `getOrElse`.

#### Example:

```js
var f = cached.deferred(function(cb) {
  var req = require('http').get(myUrl, function(res) {
    cb(null, res.statusCode);
  });
  req.once('error', cb);
});
// f can now be called and the return value will be a promise
f().then(function(statusCode) { console.log(statusCode); });
// More importantly it can be passed into cache.set
cached('myStuff').set('someKey', f);
```

### Cache.setDefaults(defaults) -> Cache.defaults

Extends the current defaults with the provided defaults.
The two important ones are `freshFor` and `expire`:

* `expire` is the time in seconds after which a value should be deleted from the cache (or whatever expiring natively means for the backend). Usually you'd want this to be `0` (never expire).
* `freshFor` is the time in seconds after which a value should be replaced. Replacing the value is done in the background and while the new value is generated (e.g. data is fetched from some service) the stale value is returned. Think of `freshFor` as a smarter `expire`.

### Cache.set(key, value, opts, cb) -> Promise[Value]

Cache store operation. `key` has to be a string, for possible `opts` see `Cache.setDefaults`.
The value can be any of the following:

a. Anything that can be converted to JSON
b. A Promise of (a)
c. A function returning (a) or (b)

The callback will be called with the resolved value, following node conventions (error, value).

### Cache.get(key, cb) -> Promise[Value]

Cache retrieve operation.
`key` has to be a string.
Cache misses are generally treated the same as retrieving `null`, errors should only be caused by transport errors and connection problems.
If you want to cache `null`/`undefined` (e.g. 404 responses), you may want to wrap it or choose a different value, like `false`, to represent this condition.

### Cache.getOrElse(key, value, opts, cb) -> Promise[Value]

This is the function you'd want to use most of the time.
It takes the same arguments as `set` but it will check the cache first.
If a value is already cached, it will return it directly (respond as fast as possible).
If the value is marked as stale (generated `n` seconds ago with `n > freshFor`), it will replace the value in the cache.
When multiple `getOrElse` calls concurrently encounter the same stale value, it will only replace the value once.
This is done on a per-instance level, so if you create many cache instances reading and writing the same keys, you are asking for trouble.
If you don't, the worst case is every process in your system fetching the value at once.
Which should be a smaller number than the number of concurrent requests in most cases.
