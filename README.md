node-request-caching
====================

## HTTP and HTTPS requests with caching for node.js

### Features

- Zero configuration
- Convenience methods for GET / POST requests with parameters (querystring / request body)
- Automatic key generation based on request signature
- Memory and Redis adapters for cache storage

### Installation

As the module is not yet on npm registry, install with:

```
npm install https://github.com/matteoagosti/node-request-caching/tarball/master
```

If you want to run tests you first have to install `mocha` and the from the module directory run:

```
npm test
```

### Usage

You can find a simple example into `examples/simple.js`

```javascript
var RequestCaching = require('../lib/request-caching');

// Cache into Memory
var rc = new RequestCaching();

for (var i = 0; i < 10; i++) {
  setTimeout(function() {
    rc.get(
      'https://graph.facebook.com/facebook',  // URI
      {fields: 'id,name'},                    // query string params
      1,                                      // TTL in seconds
      function(err, res, body, cache) {
        console.log('Response', res);         // response params object (headers, statusCode)
        console.log('Body', body);            // response body as string
        console.log('Cache', cache);          // cache info object (hit, key)
      }
    );
  }, i * 1000);
}
```

### API

#### RequestCaching(options)

Every instance has its own shared cache storage adapter.

This is the structure for the `options` parameter (with defaults values included):

```javascript
{
  store: {                    // STORE config, shared among requests from the same instance
    adapter: 'method',        // can be either memory or redis
    options: {                // any additional options for the adapter (e.g. redis config)
      ...
    }
  },
  request: {                  // any defaults for node HTTP.request method
    method: 'GET',
    ...
  },
  caching: {                  // CACHING config
    ttl: 60*60,               // default TTL in seconds, used when not specified in request
    prefix: 'requestCaching'  // prefix to append before each key, if set keys will be prefix:key
  }
}
```

#### request(options, callback)

Issues an HTTP / HTTPS request, optionally caching its result.

`options` must be an object conformig to the following schema:

```javascript
{
  uri: 'http[s]://...',       // Optional string containing remote uri. If specified
                              // it will be used for building HTTP.request options
  
  params: {                   // Optional parameters that will be querystringified and
    param1: 'value1',         // appended to GET querystring or added to POST request body.
    ...                       // If uri contains already a query string, its param=value pairs 
                              // will be merged with params, without overwrite them
  },
  
  request: {                  // HTTP.request method options
    method: 'GET',            // default request method is GET
    hostname: '...',
    port: 80,
    path: '/',
    auth: '...',
    headers: {                // If params is given, headers will contain the following:
      'key': 'value',         // 'Content-Type': 'application/x-www-form-urlencoded',
      ...                     // 'Content-Length': querystring.stringify(params)
                              // However, if you specify them, they won't get overwritten
    }
  },

  caching: {                  // CACHING config (if not specified will take instance's defaults)
    ttl: 60*60,               // TTL in seconds
    prefix: 'requestCaching', // prefix to append before key, if set final key will be prefix:key
    key: '...'                // Optional parameter containing the cache key. If not specified
                              // will be autogenerated by MD5 hashing the JSON.stringify of
                              // [querystring.stringify(options.params), options.request]
  }
}
```

`callback(err, res, body, cache)` gets invoked whenever error or response occurs. Function arguments are:
- `err`: the error message, `null` if everything is ok
- `res`: object containing some properties of the HTTP.response:
```javascript
{
  headers: {
    'key': 'value',
    ...
  },
  statusCode: ...
}
```

- `body`: response's body as string
- `cache`: object containing some cache properties:
```javascript
{
  hit: true/false,   // true if content was fetched from cache
  key: '...',        // the key (useful when using automatic key generation)
}
```

#### get(uri, params, ttl, callback)

Convenience method over `request(options, callback)`. Issues a `GET` request to the given `uri`, adding `params` to the query string, storing into cache for `ttl` seconds, invoking `callback` once done (both when error or success, according to the same schema of `request(options, callback)`. If `uri` already includes a query string, its value get added to `params`, but without overriding what's already defined in `params`.

#### post(uri, params, ttl, callback)

The same as previously mentioned `get(uri, params, ttl, callback)`, but issuing a `POST` request, adding `params` to the request body and including the following request headers:

```
'Content-Type': 'application/x-www-form-urlencoded',
'Content-Length': querystring.stringify(params)
```

### Additional notes

Right now the TTL is specified in seconds, despite the `Memory` adapter can work with milliseconds resolution. I went for it as until `Redis 2.6` will be out, the current `Redis` adapter can't go below second precision; for consistency reasons I preferred to leave everything in seconds. In addition, `Redis` key's expire precision is in the order of half a second (more or less), so pay attention when storing keys with a TTL of 1, as it may happen that when reading them after 1.5 seconds you'll still get the cached entry.