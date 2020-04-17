---
title: "Extending Envoy With Wasm and Rust"
date: 2020-04-17T16:45:18+03:00
draft: false
---

# Extending Envoy with WASM and Rust



---


*If you already know what Istio, Envoy, WASM and Rust have in common and just want to get started building your filter - feel free to skip straight to [part 2 - Building the Filter](#part2)*


---



# Part 1: The Evolution of the Mesh

As I’ve mentioned multiple times over the last couple of years - service meshes are the next stage in the evolution of cloud native infrastructure. 

The service mesh innovation is advancing on steroids. That’s exactly why [Layer5.io](https://layer5.io/) - the service mesh community that I’m proud to be a part of - is focused on making this uncontrollable sprawl of new exciting tech more manageable by creating the service mesh landscape, defining a performance spec and building [tools](https://meshery.io) for multi-service mesh management. 

## The History of Istio Extensibility

Still  - each of the mesh implementations continues developing at its own speed with **Istio** - being backed by Google and IBM  - running ahead of competition on everything related to functionality. But alas - until version 1.5 Istio was also known for performance issues caused by a number of architectural decisions taken early on in project development. One of the main culprits for performance bottlenecks was the component called Mixer. Its main responsibilities in the mesh included enforcing traffic policy and collecting telemetry. Mixer did all this with the help of Adapters - an extension mechanism which conveniently allowed one to integrate Istio with third-party policy and telemetry systems. The downside to this was that Mixer resided in each network request’s data path and caused insufferable latencies when it became overloaded. Check out t[his benchmark report](https://kinvolk.io/blog/2019/05/performance-benchmark-analysis-of-istio-and-linkerd/) to see how Istio before version 1.5 used to perform on high traffic volumes. 

The Istio team has been working hard to resolve the performance issues. One of the difficult decisions they had to take was tearing the Mixer out and moving telemetry and policy setting functionality into the mesh proxies. In Istio’s case these proxies are [Envoy](https://www.envoyproxy.io/) instances. But what about the Mixer adapters? In order to not give up the extensibility in favour of throughput - a new solution had to be designed. It did take long to arrive - and  an exciting one too! Envoy proxy now allows loading WebAssembly modules for use in its network filter chains. 

## With WASM cometh Rust

I won’t waste your time describing what WebAssembly is and how it is the future of secure lightweight computing workloads. There are other blog posts about that. I’ll just say that right now WASM support in Envoy means that one can write filters in either C++, [AssemblyScript](https://github.com/AssemblyScript/assemblyscript) or Rust. And it’s the last option that excites me the most - because I’m very fond of Rust and have been looking for a practical project to use it on.

## There Is A Hub

If you want to quickly get started with creating your own WASM-based Envoy filters - 
the wonderful folks at Solo.io already provide some tooling to help you get off the ground. Their [WebAssemblyHub](https://webassemblyhub.io/) and the **wasme** command-line utility allow one to create filters, package them and deploy to Gloo, Istio or stand-alone Envoy. But as it happens - they still don’t have Rust support built-in. 

In general it looks like the proxy-sdk for Rust is somewhat lagging behind its C++ and [AssemblyScript](https://github.com/AssemblyScript/assemblyscript) counterparts. At one point when trying to make my Rust-based filter work I even thought that the SDK is not ready yet. But thanks to [Victor Charypar](https://twitter.com/charypar) who was incredibly nice to respond to my Github issue I finally realised it’s my version of Envoy that was at fault. 

Anyway - in the end I got my filter working and what follows in part 2 of this post is the description of the steps you need to take in order to build your own WASM-based Envoy filter:

# Part 2: Building The Filter {#part2}


## Get the Toolbox Ready

Your mileage with Rust may vary - so just in case you still don’t have Rust and Cargo (the wonderful Rust package manager) installed - go ahead and get them. On Linux and macOS systems, this is done as follows:


```bash
    curl https://sh.rustup.rs -sSf | sh
```


If everything goes well, you’ll see this appear:


```bash
    Rust is installed now. Great!
```


Remember - we’re dealing with bleeding edge functionality here, so the basic Rust installation isn’t sufficient. We’ll also need to install the Rust nightly toolchain and the support for WASM compilation target. Thankfully this is very easy. All you need to is run:


```bash
    rustup toolchain install nightly
    rustup target add wasm32-unknown-unknown
```


Now we have all the tools ready - let’s initialize our project:

## Create the Library

A WASM filter is compiled from a Rust library project, so first we need to create that project:


```bash
    cargo new --lib my-wasm-filter
```


This will create a template library project in ‘my-wasm-filter’ directory.  You’ll find a lib.rs file in the src/ directory and a Cargo.toml file that tells Cargo how to build your project.


## Setting the Crate Type

The resulting library is loaded from Envoy’s C++ code so there’s no need to include any Rust-specific information in it. Therefore we’ll set the crate type to ‘cdylib’ as defined [here](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/cdylib-crates-for-c-interoperability.html). This will also produce a smaller binary. To do this - open your Cargo.toml file and under the [lib] section add:


```bash
    [lib]
    crate-type = ["cdylib"]
```


## Proxy WASM SDK

Envoy filters need to be based on the SDKs provided by the [proxy-wasm](https://github.com/proxy-wasm) project.  Specifically the [proxy-wasm-rust](https://github.com/proxy-wasm/proxy-wasm-rust-sdk) The first version of the SDK is already published on Crates.io [here](https://crates.io/crates/proxy-wasm/0.1.0). So we’ll need to add it to our Cargo.toml as a dependency:


```bash
    [dependencies]
    proxy-wasm = "0.1.0"
```



And now we’re ready to code. 

## Let’s Start Coding

At the time of writing this - no detailed documentation for writing Envoy WASM filters exists. My code is based on the examples found in the [proxy-sdk](https://github.com/proxy-wasm/proxy-wasm-rust-sdk) project and on this [example](https://github.com/envoyproxy/envoy-wasm/tree/master/api/wasm/cpp) for a CPP-based filter in the[ envoy-wasm](https://github.com/envoyproxy/envoy-wasm) project. 

Basically what we’ll need to do is:



*   Implement a [base Context trai](https://github.com/proxy-wasm/proxy-wasm-rust-sdk/blob/master/src/traits.rs#L19)t for our filter.
*   Implement an [HttpContext trait](https://github.com/proxy-wasm/proxy-wasm-rust-sdk/blob/master/src/traits.rs#L154) which inherits the base context trait.
*   Override cont[ext ](https://github.com/envoyproxy/envoy-wasm/blob/e8bf3ab26069a387f47a483d619221a0c482cd13/examples/wasm/envoy_filter_http_wasm_example.cc#L14)API methods to handle corresponding initialization and http headers/events from host.
*   Initialize our context by calling `proxy_wasm::set_http_context`

Note: our example only takes care of HTTP headers. Should we want to handle other stream and initialization events - we’ll need to implement StreamContext or RootContext accordingly. 


## What Will Our Filter Do?

I decided to implement a very simple imaginary scenario where each request directed to our service needs to be authorized by sending a token which is then checked for validity by the filter. If the token is validated - the request is passed on to the service. Otherwise - 403 response is returned to the caller. 

The validity check for the token is quite dumb - we check if the token is a prime number. In order to do that - let’s add another dependency in our Cargo.toml - the primes crate:


```bash
    [dependencies]
    proxy-wasm = "0.1.0"
    primes = "0.3.0"
```


## The Implementation:

Let’s create our Context:


```rust
    struct PrimeAuthorizer {
        context_id: u32,
    }
```


And implement the Context class for it:


```rust
    impl Context for PrimeAuthorizer {}
```


Note - we don’t need to implement any methods for Context as we’re doing all the work at L7 level - only processing HTTP headers. But we still need to have this in our code because our Context has to implement the base Context trait.

Now to the actual work - let’s implement the HttpContext. We’ll actually only need to implement one method: `HttpContext::on_http_request_headers` - to validate the headers as they arrive:


```rust

    impl HttpContext for PrimeAuthorizer {
        fn on_http_request_headers(&mut self, _: usize) -> Action {
            for (name, value) in &self.get_http_request_headers() {
                trace!("In WASM : #{} -> {}: {}", self.context_id, name, value);
            }

            match self.get_http_request_header("token") {
                Some(token) if is_prime(token.parse().unwrap()) => {
                    self.resume_http_request();
                    Action::Continue
                }
                _ => {
                    self.send_http_response(
                        403,
                        vec![("Powered-By", "proxy-wasm")],
                        Some(b"Access forbidden.\n"),
                    );
                    Action::Pause
                }
            }
        }
    }
```


As you can see  -  the method calls self.get_http_request_header to find the header named ‘token’ and then checks it for primacy. In case the token is a prime number - self.resume_http_request() passes the request further to the target cluster. Otherwise a 403 response is returned saying “Acess forbidden”. The method returns an [Action](https://github.com/proxy-wasm/proxy-wasm-rust-sdk/blob/master/src/types.rs#L34)  enum which tells Envoy whether to continue the request processing or to pause and wait for the next request.


## Testing The Filter

While building the filter was quite easy - verifying that it works proved to be a bit painful. The official Envoy binaries still don’t have WASM support built-in. But I assumed that if I build my own Envoy binary with WASM enabled - the filter should get loaded fine. The binary I built was based on [this github repo](https://github.com/envoyproxy/envoy-wasm), but alas - Envoy was crashing with “Failed to load WASM module due to a missing import: env.proxy_get_configuration”

Next step - I decided to use the Envoy image used by[ wasme CLI utility](https://docs.solo.io/web-assembly-hub/latest/tutorial_code/getting_started/) to test filters it helps build. This is the one: <code>[https://quay.io/repository/solo-io/gloo-envoy-wasm-wrapper](https://quay.io/repository/solo-io/gloo-envoy-wasm-wrapper) </code>

But this time Envoy was crashing with “"WASM missing malloc/free" . While trying to work around this I opened a [github issue ](https://github.com/proxy-wasm/proxy-wasm-rust-sdk/issues/3)on the proxy-wasm-rust project. And thanks to [Victor Charypar](https://twitter.com/charypar) I finally realized I should’ve been using the Envoy binary from the official Istio proxy image: [https://hub.docker.com/r/istio/proxyv2](https://hub.docker.com/r/istio/proxyv2) . Specifically the 1.5.0 tag. Once I built my image based on that - everything clicked! Hooray, my fitler does what I expect it to do!

My final project complete with the testing setup in a docker-compose file can be found here:

[https://github.com/otomato-gh/proxy-wasm-rust](https://github.com/otomato-gh/proxy-wasm-rust) 

All you need to do is:



1. Clone the repo
2. cargo +nightly build --target=wasm32-unknown-unknown --release

    This will create the file ‘myenvoyfilter.wasm’ in &lt;your-clone-directory>/target/wasm32-unknown-unknown. The name of the WASM file is defined by this configuration in Cargo.toml:

```bash
    [lib]
    name = "myenvoyfilter"
```

3. docker-compose up --build

    This will build a new envoy image based on istio/proxyv2 and pull an image for hasicorp/http-echo that is used as the target service 


    You should see something like: 

```bash
    proxy_1        | [2020-04-17 13:15:36.931][14][debug][wasm] [external/envoy/source/extensions/common/wasm/wasm.cc:285] Thread-Local Wasm created 4 now active


    proxy_1        | [2020-04-17 13:15:36.935][14][debug][upstream] [external/envoy/source/common/upstream/cluster_manager_impl.cc:1084] membership update for TLS cluster web_service added 1 removed 0
```

Note that Envoy is set up to listen on port 18000 of your host machine.
And that our newly-built filter is loaded from our build target:


```yaml
     proxy:
      ..
       volumes:
         - ./envoy/envoy.yaml:/etc/envoy.yaml
         - ./target/wasm32-unknown-unknown/release/myenvoyfilter.wasm:/etc/myenvoyfilter.wasm
        ..
       ports:
         - "18000:80"
```

Now go to another prompt and verify everything works as expected by sending curl requests:


```bash
    curl  -H "token":"323232" 0.0.0.0:18000
    Access forbidden.

    curl  -H "token":"32323" 0.0.0.0:18000
    "Welcome to WASM land"
```


Feel free to try this with other prime and non-prime numbers or strings. Let me know if you succeed in breaking it. :)


## Summary:



*   WASM filters allow extending Envoy functionality
*   We can build WASM filters for Envoy with Rust
*   Currently one needs the Envoy binary from Istio1.5+ to successfully execute such a filter.

Next time I get to write about this we’ll see how to deploy our Rusty filters to an Istio cluster and hopefully - we’ll be doing it with [Meshery](https://meshery.io) once the support for this is ready.

Looking forward to your questions and comments.
