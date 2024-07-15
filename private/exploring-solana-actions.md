# Exploring Solana Actions

## Introduction

Recently, Solana announced the [Solana Actions and Blinks]. These new features promise to enhance the user experience and broaden the capabilities of decentralized applications (dApps) on the Solana network. In this blog, we’ll delve into what Actions and Blinks are, how they work.

[Solana Actions and Blinks]: https://decrypt.co/236854/solana-foundation-launches-new-feature-social-media

## What are Solana Actions?

Solana Actions are a new feature designed to streamline the way users interact with decentralized applications. They serve as predefined, reusable functions that can be called upon to execute specific tasks within dApps. This modular approach simplifies the development process, allowing developers to build more complex applications with greater ease and efficiency.

You can check out [Solana Actions and Blinks] to imagine how to interact.

[Solana Actions and Blinks]: https://www.youtube.com/watch?v=m_feBl0ROik

## Creating custom Solana actions

I found [Actions and Blinks] on Solana official document how to get started to develop a Solana actions. The document uses a JS to create REST API server, however since I like Rust, I decided to build by Rust instead of JS. 
If you are youtube lover, you can check out [How to Build Solana Actions (using the @solana/actions SDK)].


[Actions and Blinks]: https://solana.com/docs/advanced/actions
[How to Build Solana Actions (using the @solana/actions SDK)]: https://www.youtube.com/watch?v=kCht01Ycif0

### Cargo new

First of all, create a new cargo package, you can run below command. If you have not installed Rust yet, you can follow [Installing Rust] to install.

```bash
cargo new solana-actions-server
```

[Installing Rust]: https://www.rust-lang.org/learn/get-started

### Install some dependencies

Rust have some popular web frameworks such as [actix-web], [rocket], [axum]. You can use any frameworks to build Solana actions. In this blog, I use `axum` because I have never used before.
Let's add it.

```bash
cargo add axum
```

and, make the server asynchronously works, we will add one of the popular async crate `tokio`.

```bash
cargo add tokio --features full
```

In order to serialize, deserialize the parameter and response, we will also add `serde` crate

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

After installing dependencies, we build basic API to check `axum` works correctly. 
We would borrow codes from [axum README], then paste inside `main.rs`

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

After pasting it, run the app.

```bash
cargo r

# If you request `http://localhost:3000/`, you are probably to able to see the text `Hello, World!`
```

[axum README]: https://docs.rs/axum/latest/axum/

### Endpoints

We need to define the endpoint first.
Following the document, we will have at least 3 endpoints. 

> ensure your application has a valid actions.json file at the root of your domain
> build an API endpoint for the GET request that returns the metadata about your Action
> create an API endpoint that accepts the POST request and returns the signable transaction for the user

- actions.json
- `GET` 
- `POST` 


So let's rewrite the routing logic like this.

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

> ensure your application responds with the required Cross-Origin headers on all Action endpoints, including the actions.json file

To satisfy this requirements, we need to specify the `cors`.

We install `tower-http` to handle `cors`

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

We need to define the `rules` filed to allow the application to map a set of a website's relative route paths to a set of other paths.

- `pathPattern`: A pattern that matches each incoming pathname.
- `apiPath`: A location destination defined as an absolute pathname or external URL.

```rs
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

Let's rewrite get method of `/actions.json`.

```rs
    let app = Router::new()
        .route("/actions.json", get(get_request_actions_json))
        .route("/api/actions/transfer-sol", get())
        .route("/api/actions/transfer-sol", post());
```

### Get request handler

When get request happens, we need to return the corresponding field.

- `icon`: The value must be an absolute HTTP or HTTPS URL of an icon image. The file must be an SVG, PNG, or WebP image, or the client/wallet must reject it as malformed.
- `title`: The value must be a UTF-8 string that represents the source of the action request. For example, this might be the name of a brand, store, application, or person making the request.
- `description`: The value must be a UTF-8 string that provides information on the action. The description should be displayed to the user.
- `label`: The value must be a UTF-8 string that will be rendered on a button for the user to click. All labels should not exceed 5 word phrases and should start with a verb to solidify the action you want the user to take. For example, "Mint NFT", "Vote Yes", or "Stake 1 SOL".
- `disabled`: The value must be boolean to represent the disabled state of the rendered button (which displays the label string). If no value is provided, disabled should default to false (i.e. enabled by default). For example, if the action endpoint is for a governance vote that has closed, set disabled=true and the label could be "Vote Closed".
- `error`: An optional error indication for non-fatal errors. If present, the client should display it to the user. If set, it should not prevent the client from interpreting the action or displaying it to the user. For example, the error can be used together with disabled to display a reason like business constraints, authorization, the state, or an error of external resource.
- `links.actions`: An optional array of related actions for the endpoint. Users should be displayed UI for each of the listed actions and expected to only perform one. For example, a governance vote action endpoint may return three options for the user: "Vote Yes", "Vote No", and "Abstain from Vote".

```rs
#[derive(Serialize)]
struct ActionGetResponse {
    title: String,
    icon: String,
    description: String,
    links: Links,
}

#[derive(Serialize)]
struct Links {
    actions: Vec<ActionLink>,
}

#[derive(Serialize)]
struct ActionLink {
    label: String,
    href: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    parameters: Option<Vec<Parameter>>,
}

#[derive(Serialize)]
struct Parameter {
    name: String,
    label: String,
    required: bool,
}

async fn get_request_handler() -> impl IntoResponse {
    let base_href = "/api/actions/transfer-sol?";
    let response = ActionGetResponse {
        title: "Actions Example - Transfer Native SOL".into(),
        icon: "https://solana-actions.vercel.app/solana_devs.jpg".into(),
        description: "Transfer SOL to another Solana wallet".into(),
        links: Links {
            actions: vec![
                ActionLink {
                    label: "Send 1 SOL".into(),
                    href: format!("{}amount=1", base_href),
                    parameters: None,
                },
                ActionLink {
                    label: "Send 5 SOL".into(),
                    href: format!("{}amount=5", base_href),
                    parameters: None,
                },
                ActionLink {
                    label: "Send 10 SOL".into(),
                    href: format!("{}amount=10", base_href),
                    parameters: None,
                },
                ActionLink {
                    label: "Send SOL".into(),
                    href: format!("{}amount={{amount}}", base_href),
                    parameters: Some(vec![Parameter {
                        name: "amount".into(),
                        label: "Enter the amount of SOL to send".into(),
                        required: true,
                    }]),
                },
            ],
        },
    };

    (StatusCode::OK, Json(response))
}
```

### Post request handler

The client (e.g. wallet, browser extension, etc) makes an HTTP `POST` JSON requests to the action URL with a payload

- account: The value must be the base58-encoded public key of an account that may sign the transaction.

After getting base58-encoded public key, we will implement the logic of transfering SOL to random pubkey. The response would be like this.

- transaction: The value must be a base64-encoded serialized transaction. The client must base64-decode the transaction and deserialize it.
- message: The value must be a UTF-8 string that describes the nature of the transaction included in the response. The client should display this value to the user. For example, this might be the name of an item being purchased, a discount applied to a purchase, or a thank you note.


```rs
#[derive(Deserialize)]
struct QueryParams {
    amount: f64,
}

#[derive(Deserialize)]
struct PostRequest {
    account: String,
}

#[derive(Serialize)]
struct PostResponse {
    transaction: String,
    message: String,
}

async fn post_request_handler(
    Query(params): Query<QueryParams>,
    Json(payload): Json<PostRequest>,
) -> Result<impl IntoResponse, (StatusCode, Json<Value>)> {
    let account = Pubkey::from_str(&payload.account).map_err(|_| {
        (
            StatusCode::BAD_REQUEST,
            Json(json!({"error": "Invalid 'account' provided"})),
        )
    })?;
    let to_pubkey = Keypair::new().pubkey();
    let lamports = (params.amount * LAMPORTS_PER_SOL as f64) as u64;

    let rpc_client = RpcClient::new_with_commitment(
        "https://api.devnet.solana.com".to_string(),
        CommitmentConfig::confirmed(),
    );

    let recent_blockhash = rpc_client.get_latest_blockhash().map_err(|e| {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({"error": format!("Failed to get latest blockhash: {}", e)})),
        )
    })?;

    let instruction = transfer(&account, &to_pubkey, lamports);
    let mut transaction = Transaction::new_with_payer(&[instruction], Some(&account));
    transaction.message.recent_blockhash = recent_blockhash;

    let serialized_transaction = serialize(&transaction).map_err(|_| {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({"error": "Failed to serialize transaction"})),
        )
    })?;

    Ok(Json(PostResponse {
        transaction: STANDARD.encode(serialized_transaction),
        message: format!("Send {} SOL to {}", params.amount, to_pubkey),
    }))
}
```

If you are getting lost, you can see the [full code](https://github.com/aoikurokawa/solana-actions-server/tree/main).

## Test action

Run the app.

```bash
cargo r
```

After running app correctly, we follow [https://dial.to], then paste `http://localhost:3000/api/actions/transfer-sol`

Now you can see the UI.

[https://dial.to]: https://dial.to


## Conclusion

Solana’s Actions and Blinks represent a significant leap forward in the blockchain space. By enhancing efficiency, speed, and user experience, these features are poised to make Solana an even more attractive platform for developers and users alike. Whether you’re a developer looking to build the next big dApp or a user seeking the best blockchain experience, Solana’s latest innovations are definitely worth exploring.

## Resources
- https://solana.com/docs/advanced/actions
- https://github.com/solana-developers/awesome-blinks?tab=readme-ov-file
