# HTTP Cache API

The buck client has two distinct protocols to communicate with the buck HTTP cache servers:

* [Thrift Over HTTP Protocol](https://buck.build/concept/http_cache_api.html#thrift_http)
* [Binary HTTP Protocol](https://buck.build/concept/http_cache_api.html#binary_http)

## Thrift Over HTTP Protocol

The goal behind this protocol is to make it easier to extend in the future. Binary protocols with custom serialization/deserialization always result in backwards compatibility problems. We overcome this by using thrift serialization to represent our requests and responses.

### The HTTP Headers

Each request and response should set the following HTTP headers:

|HTTP Header	|Possible Values (comma separated)	|
|---	|---	|
|Content-Type	|application/x-hybrid-thrift-binary	|
|---	|---	|
|X-Thrift-Protocol	|compact, binary, json, simple_json	|

The *X-Thrift-Protocol* is intended to specify the thrift protocol used to serialize the Thrift Metadata Section of the body.

### The HTTP Request/Response Format

The body of each Request and Response is always composed of:

1. **Metadata Size Bytes Section: **4 bytes representing an int32 in big-endian byte-order that corresponds to the number of bytes N of the *Thrift Metadata Section*.
2. **Thrift Metadata Section: **N bytes representing the *Thrift Metadata Section*.
3. **Payload Section: **The rest of the bytes in the body correspond to the *Payload Section*.

All HTTP requests made using this protocol should result in a 200 HTTP Status Code. Information about cache miss, invalid formats or backend problems should all come as part of the *BuckCacheResponse* thrift type described below.

### Thrift Metadata Section

The Thrift Metadata Section for HTTP Requests is always of thrift type *BuckCacheRequest*. In the case of HTTP Responses, the thrift type of this section will be BuckCacheResponse.
All thrift types involved in this protocol can be found in the following source file:

* [buck/src/com/facebook/buck/artifact_cache/thrift/buckcache.thrift](https://github.com/facebook/buck/blob/master/src/com/facebook/buck/artifact_cache/thrift/buckcache.thrift)

### Payload Section

This sections contains the raw bytes corresponding to the contents of the cache artifact being operated on. For cache fetch Requests and cache store Responses, this payload will contain zero bytes (ie, it is not used).

## Binary HTTP Protocol

Buck makes two types of requests to the cache. Note that while there is a distinction between metadata and data, the two are combined in the same way in both types of request, so they can be stored as a single blob by the server. However, while metadata is unique to each set of keys, data can be duplicated between keys, so storing them separately allows for deduplication.

### Fetch an artifact from the cache.

`GET /artifacts/key/[key]?target=[buildTarget]`
If the artifact is cached, the response should be:
status `200`
content-type `application/octet-stream`
32 bit big endian signed integer denoting the number of length in bytes of the metadata
(1) bytes of metadata
The artifact's data
If the artifact is not cached, the response should be:
status `404`
The `target` `GET` parameter is the fully qualified target name (eg `//foo:bar#default,static`) associated with the rulekey. The parameter is optional and will only be included in situations where the target is known to Buck.

### Store an artifact in the cache.

`PUT /artifacts/key`
status `202`
content-type `application/octet-stream`
32 bit big endian signed integer denoting the number of keys
(1) strings, one for each key. Each string is represented as a 16 bit big endian unsigned integer followed by the bytes of the string encoded in UTF-8
32 bit big endian signed integer denoting the number of length in bytes of the metadata
(3) bytes of metadata

1. The artifact's data

