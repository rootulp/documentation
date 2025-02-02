---
id: api-middleware
title: API Middleware
sidebar_label: API Middleware
---

## Objective

Since Day 1 of the project Prysm has been using [gRPC](https://grpc.io/) as the API layer for inter-process communication. As an example, all requests from the validator to the beacon node are conducted via gRPC. This technology helped us greatly to make sure our Ethereum client has a well-defined set of APIs, and that we don't introduce backwards incompatibility across versions of Prysm.

As time went on, users started asking if it would be possible to have a JSON-over-HTTP API, which users could query for information about the beacon node, the network state etc. Fortunately it is easy to expose HTTP endpoints for gRPC methods using the [grpc-gateway library](https://github.com/grpc-ecosystem/grpc-gateway). This was enough for our own, Prysm-specific set of APIs.

At some point Ethereum researchers, in cooperation with client devs, decided it would be a good idea to have a standard set of HTTP APIs across the whole network. This led to the [official Ethereum API specification](https://ethereum.github.io/beacon-APIs/). Unfortunately grpc-gateway is not extensible enough to allow implementing the Ethereum API spec in its entirety. For that reason we built a piece of software called the API Middleware, which is a proxy between an HTTP client and grpc-gateway. The proxy has complete control over the request/response lifecycle and can therefore provide functionality to satisfy both HTTP REST as well as gRPC.

## Useful links

- The middleware: https://github.com/prysmaticlabs/prysm/tree/develop/api/gateway/apimiddleware
- Usage in the beacon node: https://github.com/prysmaticlabs/prysm/tree/develop/beacon-chain/rpc/apimiddleware

## Using the middleware

### The `Endpoint`

The central point of the library is the `Endpoint` struct. It represents a single HTTP endpoint that is supported by the proxy, meaning that requests to this endpoint will be routed through the middleware.

```
// Endpoint is a representation of an API HTTP endpoint that should be proxied by the middleware.
type Endpoint struct {
    Path               string          // The path of the HTTP endpoint.
    GetResponse        interface{}     // The struct corresponding to the JSON structure used in a GET response.
    PostRequest        interface{}     // The struct corresponding to the JSON structure used in a POST request.
    PostResponse       interface{}     // The struct corresponding to the JSON structure used in a POST response.
    DeleteRequest      interface{}     // The struct corresponding to the JSON structure used in a DELETE request.
    DeleteResponse     interface{}     // The struct corresponding to the JSON structure used in a DELETE response.
    RequestURLLiterals []string        // Names of URL parameters that should not be base64-encoded.
    RequestQueryParams []QueryParam    // Query parameters of the request.
    Err                ErrorJson       // The struct corresponding to the error that should be returned in case of a request failure.
    Hooks              HookCollection  // A collection of functions that can be invoked at various stages of the request/response cycle.
    CustomHandlers     []CustomHandler // Functions that will be executed instead of the default request/response behaviour.
}
```

We will get back to the details of this struct later, but let's first take a look at the `EndpointFactory` interface.

```
// EndpointFactory is responsible for creating new instances of Endpoint values.
type EndpointFactory interface {
    Create(path string) (*Endpoint, error)
    Paths() []string
    IsNil() bool
}
```

As the name implies, this is where `Endpoint`s come from. If you want to register an API endpoint, you firstly need to have a factory implementation. In Prysm we already have this interface satisfied by the `BeaconEndpointFactory` struct. We will skip `IsNil` in this text because it's not related to the middleware per se.

```
type BeaconEndpointFactory struct {
}

func (f *BeaconEndpointFactory) Paths() []string {
    // some code here
}

func (f *BeaconEndpointFactory) Create(path string) (*apimiddleware.Endpoint, error) {
    // some code here
}
```

This factory will allow us to implement a new beacon node API endpoint. Let's say the Ethereum APIs define this endpoint:

> Description: Returns random data based on block root and optional nonce
> Method: GET
> URL: /eth/v1/beacon/random/{block_id}
> Query parameters: nonce (uint)
> Example request: /eth/v1/beacon/random/0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2?nonce=123456
> 
> Response fields: A (hex string, 4 bytes), B (string)
> Example response: { "A": "0x23456789", "B": "345678" }

We will also assume that we have already created protocol buffer definitions for the request and the response.

```
message RandomRequest {
    bytes block_id = 1;
    uint64 nonce = 2;
}

message RandomResponse {
    bytes a = 1;
    uint64 b = 2;
}
```

Armed with this knowledge, let's think how our `Endpoint` should look like step by step:
- `Path` - This is obvious. The path is `/eth/v1/beacon/random/{block_id}`. We must not include query parameters in the path.
- `GetResponse`- This is a GET request with a response JSON, so we obviously need some representation of the response. So far we don't have anything suitable to put in this field.
- `PostRequest` - It's not a POST request, so we don't care about this field.
- `PostResponse` - Again, it's not a POST request. We can skip this.
- `RequestURLLiterals` - Let's stop here for a minute. Do we have any URL parameters that we should **not** encode into base64? We have one parameter called `{block_id}`, and the example request tells us that it needs to be a hex string. gRPC expects base64-encoded data for each `bytes` field, which means our parameter should **not** be passed literally. This leads us to the conclusion that we don't need this field.
- `RequestQueryParams` - We have one query parameter called `nonce`, therefore we will need to use this field when preparing our endpoint.
- `Err` - The API specification does not tell us to return any custom errors, so we will use a default one, which is provided by the library.
- `Hooks` - For the sake of this example, let's say that our gRPC method returns a 32-byte long value for `A`, but we need only the first 4 bytes. This will require custom logic and we will use a hook to match our needs.
- `CustomHandlers` - We don't want to overwrite the whole request-response logic with our custom code. We leave this field alone.

### Registering our endpoint

The first thing that has to be done for every endpoint is to add its path to the array inside the factory's `Paths` method. This is very straightforward.

```
return []string{
    // all other endpoints
    "/eth/v1/beacon/random/{block_id}",
}
```

Once we have our path registered, we need to tell our factory what `Endpoint` to return once we hit this path in our proxy middleware. We add a new `case` statement at the bottom of the factory's `Create` method.

```
case "/eth/v1/beacon/random/{block_id}":
```

We don't bother with setting the `Path` and `Err` fields because the path is filled out automatically at the bottom of `Create` and the default error is present in the default `Endpoint` (at the very top of `Create` a default `Endpoint` is created, so we only need to amend it). We therefore start with `GetResponse`. Next to the factory file we have a `structs.go` file with a lot of structs defined for already existing endpoints. We add a JSON representation of our response.

```
type randomResponseJson struct {
    A string `json:"a" hex:"true"`
    B string `json:"b"` 
}
```

Note the `hex:"true"` struct tag. This tells the middleware that a JSON field is a hex string, so that it can transform it between hex and base64. This allows us to use both representations in one request, satisfying both the HTTP specification and the gRPC requirements.

Once we have our struct defined, we can use it inside the `case` statement.

```
case "/eth/v1/beacon/random/{block_id}":
    endpoint.GetResponse = &randomResponseJson{}
```

Now it's time for `RequestQueryParams`. Looking at already existing endpoints, it's not hard to figure out what has to be done here.

```
case "/eth/v1/beacon/random/{block_id}":
    endpoint.GetResponse = &randomResponseJson{}
    endpoint.RequestQueryParams = []apimiddleware.QueryParam{{Name: "nonce"}}
```

At this point we can execute the example request `/eth/v1/beacon/random/0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2?nonce=123456` and we will get a response. Our new endpoint is working! 

### Custom logic

There is one more thing we need to do before we can call it a day. The specification says that the field `A` must consist of exactly 4 bytes, but grpc-gateway passes 32 bytes into our proxy middleware. To handle this we firstly need to identify which part of the `handleApiPath` function in `api_middleware.go` would need to be changed to fit our needs. After some inspection we come to the conclusion that it's `SerializeMiddlewareResponseIntoJson` because this is where the grpc-gateway's response is transformed into the JSON sent to the HTTP client. We need to somehow inject our code before the response gets transformed: fetch the response, replace the contents of `A` by removing everything except the first 4 bytes, and transform it into JSON as usual.

Let's inspect the `HookCollection` type, which is a part of the `Endpoint` definition. We see it contains a function field called `OnPreSerializeMiddlewareResponseIntoJson`, which is exactly what we need. We need to write a function that satisfies the field's signature and implements the required logic. Notice the `RunDefault` return parameter. It tells the middleware whether it should still execute the default code after the `Pre` function completes. In our case we don't have to serialize the response into JSON ourselves, so we want to set this value to `true` upon successfully returning from the function. 

All in all, `prepareA` can look something like below (the second return value can be used when we don't want to run the default; we can return the JSON directly from here).

```
func prepareA(response interface{}) (apimiddleware.RunDefault, []byte, apimiddleware.ErrorJson) (apimiddleware.RunDefault, []byte, apimiddleware.ErrorJson) {
    // type assert parameter into our response type
    randomResponse := (...)
    
    // set the new value
    randomResponse.A = (...)
    
    // run the default
    return true, nil, nil
}
```

`custom_hooks.go` contains several examples of such hooks: pre, post, with and without running the default logic.

After implementing the hook we have to register it inside the factory.

```
case "/eth/v1/beacon/random/{block_id}":
    endpoint.GetResponse = &randomResponseJson{}
    endpoint.RequestQueryParams = []apimiddleware.QueryParam{{Name: "nonce"}}
    endpoint.Hooks = apimiddleware.HookCollection{
        OnPreSerializeMiddlewareResponseIntoJson: prepareA,
    }
```

That's it!

## A few more things to consider

The example above did not cover everything that is possible with the library. There are some other things that may be of interest when creating a new `Endpoint`:
- Currently only a few hooks are possible to implement. This is to prevent the `HookCollection` from having a lot of fields with most of them never used. If you need a new pre or post hook, either create a new wrapped version of the middleware function (in case it has no hooks yet) or amend the existing wrapped function (in case it already has either a pre or post hook). See `(...)Wrapped` in `api_middleware.go` for examples.
- Instead of implementing single hooks we can replace the whole response/request with one or more custom handlers, which we register in the factory. See `custom_handlers.go` for examples. This may be tricky to get right, but essentially allows to manipulate the request in any desirable way.
- `ProcessRequestContainerFields`, `ProcessMiddlewareResponseFields` and `process_field.go` are responsible for field translations e.g. hex to base64 and vice versa. If you need to process a field in a new way, you will need to register the struct tag and the processing function in one of the aforementioned functions. Notice that this is a global change, but it does not matter as long as you use a new tag. Because none of the existing structs contain this tag, their fields will not be processed in this new way.