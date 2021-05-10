---
layout: post
title: My first Rust crate, pretend
---

I'm very happy to publish my first crate on crates.io, [`pretend`](https://crates.io/crates/pretend) :tada:

As a play on word on [Feign](https://github.com/OpenFeign/feign), my source of inspiration, it is a declarative HTTP
and REST client.

## My love for Feign ...

I have written a few Java/Spring based micro-services before discovering feign. To communicate between micro-services,
I used to write my HTTP requests by hand, using the Apache HTTP client. It was a bit tedious, as I had to write a lot
of boilerplate code to build the URL, set the headers, serialize and deserialize bodies etc.

Moreover, the code didn't reflect the REST API I call. Since the code to create and execute a request could be very
big, it's harder to find out at a quick glance which endpoint is being called by just looking at the code.

Then I discovered Feign, and my life changed a bit. Thanks to the annotations, it was easy to describe the REST API
I want to use. All the boilerplate is also gone, as Feign was in charge of creating and executing the request.

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
  
  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);
}

GitHub github = Feign.builder().target(GitHub.class, "https://api.github.com");
```

Feign is also very modular, and allows changing nearly every component it uses, from the HTTP client to "retryers"
or the serialization framework.

## ... and my love for Rust

While I'm mainly an application developer, and "simply" want to put Rust in some odd places [^1], I really wanted to 
contribute to its ecosystem.

Today, I'm facing the same problems I faced when using the Apache HTTP client: boilerplate. While Rust HTTP clients
are really well-designed and often offer a lot of possibilities, I am in search of expressiveness. Fortunately, Rust's
procedural macros is really powerful, and allows replicating what makes Feign great, the annotation system. 

After tinkering with `syn`, `quote` and friends, as well as digging into various HTTP client documentations, I'm
now happy to present `pretend`

## Introducing `pretend`

`pretend` is an async-first HTTP client. By annotating traits with some attributes, it will automatically implement
these traits so that they can be used.

`pretend` is not really an HTTP client as it doesn't perform any request. Just like Feign, it tries to be modular
and uses client implementations. The first release ships integration with 
[`reqwest`](https://docs.rs/reqwest/latest/reqwest/) and [`isahc`](https://docs.rs/isahc/latest/isahc/).

Here is an example of how to use `pretend`. You can learn more by looking at the 
[documentation](https://docs.rs/pretend/0.1.0/pretend/) (when it is ready ...).

```rust
use pretend::{pretend, request, Pretend, Result, Url};
use pretend_reqwest::Client;

#[pretend]
trait HttpBin {
    #[request(method = "POST", path = "/anything")]
    async fn post_anything(&self, body: &'static str) -> Result<String>;
}


#[tokio::main]
async fn main() {
    let client = Client::default();
    let url = Url::parse("https://httpbin.org").unwrap();
    let pretend = Pretend::for_client(client).with_url(url);
    
    // pretend annotated traits are automatically implemented for Pretend
    let response = pretend.post_anything("hello").await.unwrap();
    assert!(response.contains("hello"));
}
```

By reusing clients, I hope to not land in this situation :slightly_smiling_face:

![XKCD: how standards proliferate](https://imgs.xkcd.com/comics/standards.png)

## Feedbacks welcomed !

Feedbacks are warmly welcomed. If this crate is missing some features don't hesitate to contact me on 
[Twitter](https://twitter.com/SfietKonstantin) or to leave an issue on 
[Github](https://github.com/SfietKonstantin/pretend). 

-----

## PS: Other similar crates

[`feignhttp`](https://crates.io/crates/feignhttp) fills a similar role. It doesn't use traits but automatically
implement functions instead. It seems to even shares the same inspiration. 

There is also [`mclient`](https://crates.io/crates/mclient) that also implement functions. 

[^1]: Like in a phone running GNU/Linux
