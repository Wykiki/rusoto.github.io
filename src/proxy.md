# HTTP Proxy

Rusoto does not support `*_PROXY` environment variables, but you can overload the
HTTP Connector it uses.

**NOTE** : The example below does not take `NO_PROXY` env var into account, even
if it parses it from environment.

```toml
[dependencies]
anyhow = "1.0.28"
rusoto_core = "0.44"
rusoto_credential = "0.44"
rusoto_acm = "0.44"
hyper = { version = "0.13.6", features = ["runtime"] }
hyper-proxy = "0.8"
hyper-tls = "0.4"
```

```rust
extern crate anyhow;
extern crate hyper;
extern crate hyper_proxy;
extern crate hyper_tls;
extern crate rusoto_core;
extern crate rusoto_credential;

use hyper::client::{Client, HttpConnector};
use hyper::Uri;
use hyper_proxy::{Intercept, Proxy, ProxyConnector};
use hyper_tls::HttpsConnector;
use rusoto_acm::AcmClient;
use rusoto_core::request::HttpClient;
use rusoto_core::Region;
use rusoto_credential::ChainProvider;

fn main() {
    let credentials_provider = ChainProvider::new();
    let acm_client = AcmClient::new_with(
        create_client().unwrap(),
        credentials_provider.clone(),
        Region::EuWest3,
    );
}

fn create_client() -> anyhow::Result<HttpClient<ProxyConnector<HttpsConnector<HttpConnector>>>> {
    let http_proxy = std::env::var("HTTP_PROXY")
        .or_else(|_| std::env::var("http_proxy"))
        .ok();
    let https_proxy = std::env::var("HTTPS_PROXY")
        .or_else(|_| std::env::var("https_proxy"))
        .ok()
        .or_else(|| http_proxy.clone());
    let _no_proxy = std::env::var("NO_PROXY")
        .or_else(|_| std::env::var("no_proxy"))
        .ok();
    let mut proxies: Vec<Proxy> = Vec::new();
    if let Some(prox) = http_proxy {
        proxies.push(Proxy::new(
            Intercept::Http,
            prox.parse::<Uri>().expect("Malformed HTTP_PROXY env var"),
        ));
    }
    if let Some(prox) = https_proxy {
        proxies.push(Proxy::new(
            Intercept::Https,
            prox.parse::<Uri>().expect("Malformed HTTPS_PROXY env var"),
        ));
    }
    let https_connector = HttpsConnector::new();
    let proxy_connector = match !proxies.is_empty() {
        true => {
            let mut proxy_connector =
                ProxyConnector::from_proxy(https_connector, proxies.pop().unwrap())?;
            while !proxies.is_empty() {
                proxy_connector.add_proxy(proxies.pop().unwrap());
            }
            proxy_connector
        }
        false => ProxyConnector::new(https_connector)?,
    };
    let mut hyper_builder = Client::builder();

    // disabling due to connection closed issue
    hyper_builder.pool_max_idle_per_host(0);
    Ok(rusoto_core::HttpClient::from_builder(
        hyper_builder,
        proxy_connector,
    ))
}
```
