# yggdrasilctl

A library for accessing [Admin API] of [Yggdrasil Network router].

It supports both sync and async environment. All you need is to provide
socket that implements either `Read` and `Write` traits from `std` for synchronous
operations or `AsyncRead` and `AsyncWrite` traits from an async runtime.
Currently supported runtimes are `tokio` and `futures`. If your favorite
runtime is not in the list, consider creating an issue or pull request.

[Admin API]: https://yggdrasil-network.github.io/admin.html
[Yggdrasil Network router]: https://github.com/yggdrasil-network/yggdrasil-go

# Compatibility

This library is backward-compatible with the most existing versions of [`yggdrasil-go`], and have a prospect of forward-compatibility.
Despite this, potential user should note that [`yggdrasil-go`] is still an experimental software.
And there were seen breaking changes in the certain API calls, notably in versions `0.4.5` (October 2022), `0.5.0` (November 2023).

Nevertheless, this library was successfully tested for compatibility with [`yggdrasil-go`] of versions: `0.4.4`, `0.4.7`, `0.5.1`, `0.5.4`, `0.5.5`, `0.5.6`, `0.5.12`.
You can test compatibility yourself by running the following command in the crate directory.

```sh
$ YGGDRASIL_SOCKET="/run/yggdrasil/yggdrasil.sock" cargo test -p yggdrasilctl
```

[`yggdrasil-go`]: https://github.com/yggdrasil-network/yggdrasil-go

# Basic usage

Add either line to your dependencies in `Cargo.toml`

```toml
# Use `std` (synchronous)
yggdrasilctl = "1"
# Use async runtime
# Available features: "use_tokio" or "use_futures"
yggdrasilctl = { version = "1", default-features = false, features = [ "use_tokio" ] }
```

Next:

```rust,ignore
use yggdrasilctl::Endpoint;
use std::os::unix::net::UnixStream;

// Create socket using your favorite runtime
let socket = UnixStream::connect("/run/yggdrasil/yggdrasil.sock")/*.await*/.unwrap();

// Attach endpoint to a socket
let mut endpoint = Endpoint::attach(socket);

// First you can get I/O or protocol parsing error
let maybe_error = endpoint.get_self()/*.await*/.unwrap();

// Then Admin API can return error (string) to your request
match maybe_error {
    Ok(response) => println!("Yggdrasil address: {}", response.address),
    Err(error) => println!("Admin API returned error: {error}"),
}
```

# Advanced usage

You may also want to perform `debug_*` requests which are deliberately unimplemented in this library.
For this case `yggdrasilctl` allows you to declare a structure of a response you expect to receive.

First, add crates `serde` and `serde_json` to your dependecies

```toml
# Imports derive macros for `Deserialize` trait
serde = { version = "1", features = [ "derive" ] }
# Imports enum `Value` that represents any possible json value
serde_json = "1"
```

Next:

```rust,ignore
use yggdrasilctl::Endpoint;
use serde::Deserialize;
use serde_json::Value;
use std::collections::HashMap;
use std::net::Ipv6Addr;

// Connect endpoint
use std::os::unix::net::UnixStream;
let socket = UnixStream::connect("/var/run/yggdrasil/yggdrasil.sock")/*.await*/.unwrap();
let mut endpoint = Endpoint::attach(socket);
let get_self = endpoint.get_self()/*.await*/.unwrap().unwrap();

// Declare a struct you expect to receive
#[derive(Deserialize)]
struct DebugRemoteGetSelfEntry {
    coords: String,
    key: String,
}
type DebugRemoteGetSelf = HashMap<Ipv6Addr, DebugRemoteGetSelfEntry>;

// Pass arguments to the request
let mut args = HashMap::<String, Value>::new();
args.insert("key".to_string(), Value::from(get_self.key.as_str()));

// Perform the request
let maybe_error = endpoint.request_args::<DebugRemoteGetSelf>("debug_remotegetself", args)/*.await*/.unwrap();

// Parse the request
match maybe_error {
    Ok(response) =>
        println!(
            "Yggdrasil node coordinates: {:?}",
            response[&get_self.address].coords
        ),
    Err(error) => println!("Admin API returned error: {error}"),
}
```
