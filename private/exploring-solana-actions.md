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

### Endpoints

Following the document, we need three endpoints to handle `GET` and `POST` request and one for returning json file. 

> build an API endpoint for the GET request that returns the metadata about your Action
> create an API endpoint that accepts the POST request and returns the signable transaction for the user
> ensure your application has a valid actions.json file at the root of your domain

So let's rewrite the routing.

```rs
#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/actions.json", get())
        .route("/api/actions/transfer-sol", get())
        .route("/api/actions/transfer-sol", post());

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Cors

To satisfy this requirements, we need to specify the `cors`.

> ensure your application responds with the required Cross-Origin headers on all Action endpoints, including the actions.json file

We install `tower-http` to define `cors`

```bash
cargo add tower-http --features full
```

```rs
#[tokio::main]
async fn main() {
    let cors = CorsLayer::new()
        .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
        .allow_headers([
            CONTENT_TYPE,
            AUTHORIZATION,
            CONTENT_ENCODING,
            ACCEPT_ENCODING,
        ])
        .allow_origin(Any);

    let app = Router::new()
        .route("/actions.json", get())
        .route("/api/actions/transfer-sol", get())
        .route("/api/actions/transfer-sol", post());

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Get request actions json

> The purpose of the actions.json file allows an application to instruct clients on what website URLs support Solana Actions and provide a mapping that can be used to perform GET requests to an Actions API server.

```
async fn get_request_actions_json() -> impl IntoResponse {
    Json(json!({
        "rules": [
            {
                "pathPattern": "/*",
                "apiPath": "/api/actions/*",
            },
            {
                "pathPattern": "/api/actions/**",
                "apiPath": "/api/actions/**",
            },
        ],
    }))
}

```

### Get request handler

### Post request handler



## Resources
- https://solana.com/docs/advanced/actions
- https://github.com/solana-developers/awesome-blinks?tab=readme-ov-file
