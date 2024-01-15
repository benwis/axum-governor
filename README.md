 A Tower service and layer that provides a rate-limiting backend by [governor](https://github.com/antifuchs/governor). Based heavily on the work done for [actix-governor](https://github.com/AaronErhardt/actix-governor). Works with Axum, Hyper, Tonic, and anything else based on Tower!

 # Features:

 + Rate limit requests based on peer IP address, IP address headers, globally, or via custom keys
 + Custom traffic limiting criteria per second, or to certain bursts
 + Simple to use
 + High customizability
 + High performance
 + Robust yet flexible API


 # How does it work?

 Each governor middleware has a configuration that stores a quota.
 The quota specifies how many requests can be sent from an IP address
 before the middleware starts blocking further requests.

 For example if the quota allowed ten requests a client could send a burst of
 ten requests in short time before the middleware starts blocking.

 Once at least one element of the quota was used the elements of the quota
 will be replenished after a specified period.

 For example if this period was 2 seconds and the quota was empty
 it would take 2 seconds to replenish one element of the quota.
 This means you could send one request every two seconds on average.

 If there was a quota that allowed ten requests with the same period
 a client could again send a burst of ten requests and then had to wait
 two seconds before sending further requests or 20 seconds before the full
 quota would be replenished and he could send another burst.

 # Example
 ```rust,no_run
use axum::{error_handling::HandleErrorLayer, routing::get, BoxError, Router};
use std::net::SocketAddr;
use std::time::Duration;
use tokio::net::TcpListener;
use tower::ServiceBuilder;
use tower_governor::{errors::display_error, governor::GovernorConfigBuilder, GovernorLayer};

async fn hello() -> &'static str {
    "Hello world"
}

#[tokio::main]
async fn main() {
    // Configure tracing if desired
    // construct a subscriber that prints formatted traces to stdout
    let subscriber = tracing_subscriber::FmtSubscriber::new();
    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber).unwrap();

    // Allow bursts with up to five requests per IP address
    // and replenishes one element every two seconds
    // We Box it because Axum 0.6 requires all Layers to be Clone
    // and thus we need a static reference to it
    let governor_conf = Box::new(
        GovernorConfigBuilder::default()
            .per_second(2)
            .burst_size(5)
            .finish()
            .unwrap(),
    );

    let governor_limiter = governor_conf.limiter().clone();
    let interval = Duration::from_secs(60);
    // a separate background task to clean up
    std::thread::spawn(move || {
        loop {
            std::thread::sleep(interval);
            tracing::info!("rate limiting storage size: {}", governor_limiter.len());
            governor_limiter.retain_recent();
        }
    });

    // build our application with a route
    let app = Router::new()
        // `GET /` goes to `root`
        .route("/", get(hello))
        .layer(GovernorLayer {
                    // We can leak this because it is created once and then
                    config: Box::leak(governor_conf),
        });

    // run our app with hyper
    // `axum::Server` is a re-export of `hyper::Server`
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);
    let listener = TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())
        .await
        .unwrap();
}
 ```

 # Configuration presets

 Instead of using the configuration builder you can use predefined presets.

 + [`GovernorConfig::default()`](https://docs.rs/tower_governor/latest/tower_governor/governor/struct.GovernorConfig.html#method.default): The default configuration which is suitable for most services. Allows bursts with up to eight requests and replenishes one element after 500ms, based on peer IP.

 + [`GovernorConfig::secure()`](https://docs.rs/tower_governor/latest/tower_governor/governor/struct.GovernorConfig.html#method.secure): A default configuration for security related services.
 Allows bursts with up to two requests and replenishes one element after four seconds, based on peer IP.

 For example the secure configuration can be used as a short version of this code:

 ```rust
 use tower_governor::governor::GovernorConfigBuilder;

 let config = GovernorConfigBuilder::default()
     .per_second(4)
     .burst_size(2)
     .finish()
     .unwrap();
 ```

 # Customize rate limiting key

 By default, rate limiting is done using the peer IP address (i.e. the IP address of the HTTP client that requested your app: either your user or a reverse proxy, depending on your deployment setup).
 You can configure a different behavior which:
 1. can be useful in itself
 2. allows you to setup multiple instances of this middleware based on different keys (for example, if you want to apply rate limiting with different rates on IP and API keys at the same time)

 This is achieved by defining a [KeyExtractor] and giving it to a [Governor] instance.
 Three ready-to-use key extractors are provided:
 - [PeerIpKeyExtractor]: this is the default, it uses the peer IP address of the request.
 - [SmartIpKeyExtractor]: Looks for common IP identification headers usually provided by reverse proxies in order(x-forwarded-for,x-real-ip, forwarded) and falls back to the peer IP address.
 - [GlobalKeyExtractor]: uses the same key for all incoming requests

 Check out the [custom_key_bearer](https://github.com/benwis/tower-governor/blob/main/examples/src/custom_key_bearer.rs) example for more information.


 # Add x-ratelimit headers

 By default, `x-ratelimit-after` is enabled but if you want to enable `x-ratelimit-limit`, `x-ratelimit-whitelisted` and `x-ratelimit-remaining` use the [`.use_headers()`](https://docs.rs/tower_governor/latest/tower_governor/governor/struct.GovernorConfigBuilder.html#method.use_headers) method on your GovernorConfig.


 # Error Handling

 This crate surfaces a GovernorError with suggested headers, and includes a [`display_error`]: crate::errors::display_error() function that will turn those errors into a Response. Feel free to provide your own error handler that takes in a BoxError and returns a [`Response`](https://docs.rs/http/latest/http/response/struct.Response.html). Tower Layers require that all Errors be handled, or it will fail to compile. 

 # Common pitfalls

 1. Do not construct the same configuration multiple times, unless explicitly wanted!
 This will create an independent rate limiter for each configuration! Instead pass the same configuration reference into [`Governor::new()`](https://docs.rs/tower_governor/latest/tower_governor/governor/struct.Governor.html#method.new), like it is described in the example.

 2. Be careful to create your server with [`.into_make_service_with_connect_info::<SocketAddr>`](https://docs.rs/axum/latest/axum/struct.Router.html#method.into_make_service_with_connect_info) instead of `.into_make_service()` if you are using the default PeerIpKeyExtractor. Otherwise there will be no peer ip address for Tower to find!