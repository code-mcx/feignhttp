# FeignHttp

[![rust](https://img.shields.io/badge/language-rust-c99272.svg)](https://github.com/rust-lang/rust)
[![MIT licensed](https://img.shields.io/github/license/dxx/feignhttp.svg?color=blue)](./LICENSE)

FeignHttp is a declarative HTTP client. Based on rust macros.

## Features

* Easy to use
* Asynchronous request
* Supports `json` and `plain text`
* [Reqwest](https://github.com/seanmonstar/reqwest) of internal use

## Usage

FeignHttp mark macros on asynchronous functions, you need to add [tokio](https://github.com/tokio-rs/tokio) in your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
feignhttp = { version = "0.0.1" }
```

Then add the following code:

```rust
use feignhttp::get;

#[get("https://api.github.com")]
async fn github() -> Result<String, Box<dyn std::error::Error>> {}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = github().await?;
    println!("result: {}", r);

    Ok(())
}
```

The `get` attribute macro specifies get request, `Result<String, Box<dyn std::error::Error>>` specifies the 
return result. It will send get request to `https://api.github.com` and receive a plain text body. You must specify 
`Box<dyn std::error::Error>` to be compatible with all errors returned by the internal call library.

### Making a POST request

For a post request, you should use the `post` attribute macro to specify request method and use a `body` attribute to specify 
a request body.

```rust
use feignhttp::post;

#[post(url = "http://localhost:8080/create")]
async fn create(#[body] text: String) -> Result<String, Box<dyn std::error::Error>> {}
```

The `#[body]` mark a request body. Function parameter `text` is a String type, it will put in the request body as plain text. 
String and &str will be put as plain text into the request body.

### Paths

Using `path` to specify path value:

```rust
use feignhttp::get;

#[get("https://api.github.com/repos/{owner}/{repo}")]
async fn repository(
    #[path] owner: &str,
    #[path] repo: String,
) -> Result<String, Box<dyn std::error::Error>> {}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = repository("dxx", "feignhttp".to_string()).await?;
    println!("repository result: {}", r);
    Ok(())
}
```

`dxx`  will replace `{owner}` and `feignhttp` will replace `{repo}` , the url to be send will be 
`https://api.github.com/repos/dxx/feignhttp`. You can specify a path name like `#[path("owner")]`.

### Query Parameters

Using `param` to specify query parameter:

```rust
use feignhttp::get;

#[get("https://api.github.com/repos/{owner}/{repo}/contributors")]
async fn contributors(
    #[path("owner")] user: &str,
    #[path] repo: &str,
    #[param] page: u32,
) -> Result<String, Box<dyn std::error::Error>> {}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = contributors (
        "dxx",
        "feignhttp",
        1,
    ).await?;
    println!("contributors result: {}", r);

    Ok(())
}
```

The `page` parameter will as query parameter in the url. An url which will be send is `https://api.github.com/repos/dxx/feignhttp?page=1`.

> Note: A function parameter without `param` attribute will as a query parameter by default.

### Headers

Using `header` to specify request header:

```rust
use feignhttp::get;

#[get("https://api.github.com/repos/{owner}/{repo}/commits")]
async fn commits(
    #[header] accept: &str,
    #[path] owner: &str,
    #[path] repo: &str,
    #[param] page: u32,
    #[param] per_page: u32,
) -> Result<String, Box<dyn std::error::Error>> {}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = commits(
        "application/vnd.github.v3+json",
        "dxx",
        "feignhttp",
        1,
        5,
    )
    .await?;
    println!("commits result: {}", r);

    Ok(())
}
```

 A header `accept:application/vnd.github.v3+json ` will be send.

### URL constant

We can use constant to maintain all urls of request:

```rust
use feignhttp::get;

const GITHUB_URL: &str = "https://api.github.com";

#[get(GITHUB_URL, path = "/repos/{owner}/{repo}/languages")]
async fn languages(
    #[path] owner: &str,
    #[path] repo: &str,
) -> Result<String, Box<dyn std::error::Error>> {}
```

Url constant must be the first metadata in get attribute macro. You also can specify metadata key:

```rust
#[get(url = GITHUB_URL, path = "/repos/{owner}/{repo}/languages")]
async fn languages(
    #[path] owner: &str,
    #[path] repo: &str,
) -> Result<String, Box<dyn std::error::Error>> {}
```

### JSON

[Serde](https://github.com/serde-rs/serde) is a framework for serializing and deserializing Rust data structures. When use json, you should add serde in `Carto.toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
```

Here is an example of getting json:

```rust
use feignhttp::get;
use serde::Deserialize;

// Deserialize: Specifies deserialization
#[derive(Debug, Deserialize)]
struct IssueItem {
    pub id: u32,
    pub number: u32,
    pub title: String,
    pub url: String,
    pub repository_url: String,
    pub state: String,
    pub body: Option<String>,
}


const GITHUB_URL: &str = "https://api.github.com";

#[get(url = GITHUB_URL, path = "/repos/{owner}/{repo}/issues")]
async fn issues(
    #[path] owner: &str,
    #[path] repo: &str,
    page: u32,
    per_page: u32,
) -> Result<Vec<IssueItem>, Box<dyn std::error::Error>> {}


#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = issues("octocat", "hello-world", 1, 2).await?;
    println!("issues: {:#?}", r);

    Ok(())
}
```

This issues function return `Vec<IssueItem>`, it is deserialized according to the content of the response.

Send a json request:

```rust
use feignhttp::post;
use serde::Serialize;

#[derive(Debug, Serialize)]
struct User {
    id: i32,
    name: String,
}

#[post(url = "http://localhost:8080/create")]
async fn create(#[body] user: User) -> Result<String, Box<dyn std::error::Error>> {}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let user = User {
        id: 1,
        name: "jack".to_string(),
    };
    let _r = create(user).await?;

    Ok(())
}
```

See [here](./examples/json.rs) for a complete example.

### Using structure

Structure is a good way to manage requests. Define a structure and then define a large number of request methods：

```rust
use feignhttp::feign;

const GITHUB_URL: &str = "https://api.github.com";

struct Github {}

#[feign(url = GITHUB_URL)]
impl Github {
    #[get]
    async fn home() -> Result<String, Box<dyn std::error::Error>> {}

    #[get("/repos/{owner}/{repo}")]
    async fn repository(
        #[path] owner: &str,
        #[path] repo: &str,
    ) -> Result<String, Box<dyn std::error::Error>> {}

    // ...
    
    // Structure method still send request
    #[get(path = "/repos/{owner}/{repo}/languages")]
    async fn languages(
        &self,
        #[path] owner: &str,
        #[path] repo: &str,
    ) -> Result<String, Box<dyn std::error::Error>> {}
}
```

See [here](./examples/struct.rs) for a complete example.

### Timeout configuration

If you need to configure the timeout, use `connect_timeout` and `timeout` to specify connect timeout and read timeout.

Connect timeout:

```rust
#[get(url = "http://xxx.com", connect_timeout = 3000)]
async fn connect_timeout() -> Result<String, Box<dyn std::error::Error>> {}
```

Read timeout:

```rust
#[get(url = "http://localhost:8080", timeout = 3000)]
async fn timeout() -> Result<String, Box<dyn std::error::Error>> {}
```

## License

FeignHttp is provided under the MIT license. See [LICENSE](./LICENSE).
