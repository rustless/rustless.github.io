---
layout: default
---

## What is Rustless?

Rustless is a REST-like API micro-framework for Rust. It's designed to provide a simple DSL to easily develop RESTful APIs. It has built-in support for common conventions, including multiple formats, subdomain/prefix restriction, content negotiation, versioning and much more.

Rustless in a port of [Grape] library from Ruby world and is still mostly **in progress** (that mean that API and features in
**experimental** in Rust's terms). Based on [hyper] - an HTTP library for Rust.

[Grape]: https://github.com/intridea/grape
[hyper]: https://github.com/hyperium/hyper

## Basic Usage

Below is a simple example showing some of the more common features of Rustless.

~~~rust
#![feature(phase)]

#[phase(plugin)]
extern crate rustless;
extern crate rustless;
extern crate serialize;

use std::io::net::ip::Ipv4Addr;
use serialize::json::Json;
use rustless::{
    Rustless, Application, Valico, Api, Client, NS, 
    PathVersioning, AcceptHeaderVersioning, HandleResult
};

fn main() {

    let api = box Api::build(|api| {
        // Specify API version and versioning strategy
        api.version("v1", AcceptHeaderVersioning("chat"));
        api.prefix("api");

        // Create API for chats
        let chats_api = box Api::build(|chats_api| {
            // Add namespace
            chats_api.namespace("chats/:id", |chat_ns| {
                
                // Valico settings for this namespace
                chat_ns.params(|params| { 
                    params.req_typed("id", Valico::u64())
                });

                // Create endpoint for POST /chats/:id/users/:user_id
                chat_ns.post("users/:user_id", |endpoint| {
                
                    // Add description
                    endpoint.desc("Update user");

                    // Valico settings for endpoint params
                    endpoint.params(|params| { 
                        params.req_typed("user_id", Valico::u64());
                        params.req_typed("name", Valico::string())
                    });

                    // Set-up handler for endpoint, note that we return
                    // of macro invocation.
                    edp_handler!(endpoint, |client, params| {
                        client.json(params)
                    })
                });

            });
        });

        api.mount(chats_api);
    });

    let mut app = Application::new();
    app.mount(api);

    let rustless: Rustless = Rustless;
    rustless.listen(
        app,
        Ipv4Addr(127, 0, 0, 1),
        3000
    );

    println!("Rustless server started!");
}
~~~

## Parameter Validation and Coercion

You can define validations and coercion options for your parameters using a DSL block in Endpoint and Namespace definition. See [Valico] for more info about things you can do.

[Valico]: https://github.com/rustless/valico

## Query strings

Rustless is intergated with [rust-query] to allow smart query-string parsing 
end decoding (even with nesting, like `foo[0][a]=a&foo[0][b]=b&foo[1][a]=aa&foo[1][b]=bb`). See [rust-query] for more info.

[rust-query]: https://github.com/rustless/rust-query

## Versioning

There are three strategies in which clients can reach your API's endpoints: 

    * PathVersioning
    * AcceptHeaderVersioning
    * ParamVersioning

### Path

~~~rust
api.version("v1", PathVersioning);
~~~

Using this versioning strategy, clients should pass the desired version in the URL.

    curl -H http://localhost:3000/v1/chats/

### Header

~~~rust
api.version("v1", AcceptHeaderVersioning("chat"));
~~~

Using this versioning strategy, clients should pass the desired version in the HTTP `Accept` head.

    curl -H Accept:application/vnd.chat.v1+json http://localhost:3000/chats

Accept version format is the same as Github (uses)[https://developer.github.com/v3/media/].

### Param

~~~rust
api.version("v1", ParamVersioning("ver"));
~~~

Using this versioning strategy, clients should pass the desired version as a request parameter in the URL query.

    curl -H http://localhost:9292/statuses/public_timeline?ver=v1

## HTTP Status Code

By default Rustless returns a 200 status code for `GET`-Requests and 201 for `POST`-Requests. You can use `status` and `set_status` to query and set the actual HTTP Status Code

~~~rust
client.set_status(NotFound);
~~~

## Parameters

Request parameters are available through the `params` struct. This includes `GET`, `POST` and `PUT` parameters, along with any named parameters you specify in your route strings.

The request:

~~~
curl -d '{"text": "hello from echo"}' 'http://localhost:3000/echo' -H Content-Type:application/json -v
~~~

The Rustless endpoint:

~~~rust
api.post("", |endpoint| {
    edp_handler!(endpoint, |client, params| {
        client.json(params)
    })
});
~~~

In the case of conflict between either of:

* route string parameters
* `GET`, `POST` and `PUT` parameters
* the contents of the request body on `POST` and `PUT`

route string parameters will have precedence.

## Redirecting

You can redirect to a new url temporarily (302) or permanently (301).

~~~rust
client.redirect("http://google.com");
~~~

~~~rust
client.redirect_permanent("http://google.com");
~~~

## Raising Exceptions

You can abort the execution of an API method by raising errors with `error`.

~~~rust
client.error(CustomError);
~~~

## Before and After

Blocks can be executed before or after every API call, using `before`, `after`,
`before_validation` and `after_validation`.

Before and after callbacks execute in the following order:

1. `before`
2. `before_validation`
3. _validations_
4. `after_validation`
5. _the API call_
6. `after`

Steps 4, 5 and 6 only happen if validation succeeds.

E.g. using `after`:

~~~rust
chats_api.after(callback!(|client| {
    client.set_status(NotFound);
    Ok(())
}));
~~~

The block applies to every API call within and below the current nesting level.
