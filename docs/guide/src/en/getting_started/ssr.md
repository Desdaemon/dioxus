# Server-Side Rendering

The Dioxus VirtualDom can be rendered server-side.

[Example: Dioxus DocSite](https://github.com/dioxusLabs/docsite)

## Multithreaded Support

The Dioxus VirtualDom, sadly, is not currently `Send`. Internally, we use quite a bit of interior mutability which is not thread-safe. This means you can't easily use Dioxus with most web frameworks like Tide, Rocket, Axum, etc.

To solve this, you'll want to spawn a VirtualDom on its own thread and communicate with it via channels.

When working with web frameworks that require `Send`, it is possible to render a VirtualDom immediately to a String – but you cannot hold the VirtualDom across an await point. For retained-state SSR (essentially LiveView), you'll need to create a pool of VirtualDoms.


## Setup

If you just want to render `rsx!` or a VirtualDom to HTML, check out the API docs. It's pretty simple:

```rust
// We can render VirtualDoms
let mut vdom = VirtualDom::new(app);
let _ = vdom.rebuild();
println!("{}", dioxus_ssr::render_vdom(&vdom));

// Or we can render rsx! calls directly
println!( "{}", dioxus_ssr::render_lazy(rsx! { h1 { "Hello, world!" } } );
```

However, for this guide, we're going to show how to use Dioxus SSR with `Axum`.

Make sure you have Rust and Cargo installed, and then create a new project:

```shell
cargo new --bin demo
cd app
```

Add Dioxus and the `ssr` renderer feature:

```shell
cargo add dioxus
cargo add dioxus-ssr
```

Next, add all the Axum dependencies. This will be different if you're using a different Web Framework

```
cargo add tokio --features full
cargo add axum
```

Your dependencies should look roughly like this:

```toml
[dependencies]
axum = "0.4.5"
dioxus = { version = "*" }
dioxus-ssr = { version = "*" }
tokio = { version = "1.15.0", features = ["full"] }
```

Now, set up your Axum app to respond on an endpoint.

```rust
use axum::{response::Html, routing::get, Router};
use dioxus::prelude::*;

#[tokio::main]
async fn main() {
    let addr = std::net::SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("listening on http://{}", addr);

    axum::Server::bind(&addr)
        .serve(
            Router::new()
                .route("/", get(app_endpoint))
                .into_make_service(),
        )
        .await
        .unwrap();
}
```

And then add our endpoint. We can either render `rsx!` directly:

```rust
async fn app_endpoint() -> Html<String> {
    Html(dioxus_ssr::render_lazy(rsx! {
            h1 { "hello world!" }
    }))
}
```

Or we can render VirtualDoms.

```rust
async fn app_endpoint() -> Html<String> {
    fn app(cx: Scope) -> Element {
        cx.render(rsx!(h1 { "hello world" }))
    }
    let mut app = VirtualDom::new(app);
    let _ = app.rebuild();

    Html(dioxus_ssr::render_vdom(&app))
}
```

And that's it!

> You might notice that you cannot hold the VirtualDom across an await point. Dioxus is currently not ThreadSafe, so it _must_ remain on the thread it started. We are working on loosening this requirement.