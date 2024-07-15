# Exploring Solana Actions

## Introduction

Recently, Solana announced the [Solana Actions and Blinks]. These new features promise to enhance the user experience and broaden the capabilities of decentralized applications (dApps) on the Solana network. In this blog, we’ll delve into what Actions and Blinks are, how they work.

[Solana Actions and Blinks]: https://decrypt.co/236854/solana-foundation-launches-new-feature-social-media

## What are Solana Actions?

Solana Actions are a new feature designed to streamline the way users interact with decentralized applications. They serve as predefined, reusable functions that can be called upon to execute specific tasks within dApps. This modular approach simplifies the development process, allowing developers to build more complex applications with greater ease and efficiency.

You can check out [this youtube] to imagine how to interact.

[this youtube]: https://www.youtube.com/watch?v=m_feBl0ROik

## Creating custom Solana actions

I found [Actions and Blinks] on Solana official document how to get started to develop a Solana actions. and also if you are youtube lover, you can check out [How to Build Solana Actions (using the @solana/actions SDK)].
The document uses a JS to create REST API server, however since I like Rust, I decided to build by Rust instead of JS.

[Actions and Blinks]: https://solana.com/docs/advanced/actions
[How to Build Solana Actions (using the @solana/actions SDK)]: https://www.youtube.com/watch?v=kCht01Ycif0

### Cargo new

To create a new cargo package, you can run below command. If you have not installed Rust yet, you can follow [Installing Rust] to install.

```bash
cargo new solana-actions-server
```

[Installing Rust]: https://www.rust-lang.org/learn/get-started

### Install some dependencies

Rust have some popular web frameworks such as [actix-web], [rocket], [axum]. You can use any frameworks to build Solana actions. In this blog, I use axum because I have never used before.
Let's install it.

```bash
cargo add axum
```

and, make the server asynchronously, we need one of the popular async crate tokio.

```bash
cargo add tokio --features full
```

In order to serialize, deserialize the parameter and response, we need serde create

```bash
cargo add serde --features derive
```

```bash
cargo add serde-json
```

[actix-web]: https://docs.rs/actix-web/latest/actix_web/
[rocket]: https://docs.rs/rocket/latest/rocket/
[axum]: https://docs.rs/axum/latest/axum/

### Checks axum working correctly

First of all, we build basic API to check working correctly. 
We would borrow codes from [axum README]

```rs
use axum::{
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    // build our application with a single route
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    // run our app with hyper, listening globally on port 3000
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

then run the server.

```bash
cargo r

# If you request `http://localhost:3000/`, you can see the text `Hello, World!`
```

[axum README]: https://docs.rs/axum/latest/axum/



## Resources
- https://solana.com/docs/advanced/actions
- https://github.com/solana-developers/awesome-blinks?tab=readme-ov-file
