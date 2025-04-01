# Writing a WebSocket echo server in Rust: the handshake
Websockets are a convenient way to maintain a connection between a server and a client using all the existing HTTP infrastructure. In this article, we'll cover the basics of a working WebSocket echo server. We will cover the handshake and frame concept.

Here are some links:
- [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)
- [complete project code](https://github.com/eguefif/ws-echoserver-rust)

## The server Handshake
### A basic TCP server
I like to start by thinking about how I'll use the code I write. In our case, we want to run a server. Let's write our server's main:

```rust
fn main() -> std::io::Result<()> {
    run_server("127.0.0.1", 8000)?;
    Ok(())
}
```

By doing so, we get an idea of what our entry point looks like and the parameters it needs. Let's add the following function to this file.

```rust
use std::net::TcpListener;
use std::thread;

pub fn run_server(ip: &str, port: u32) -> std::io::Result<()> {
    let listener = TcpListener::bind(format!("{}:{}", ip, port))?;
    for stream in listener.incoming() {
        match stream {
            Ok(socket) => {
                thread::spawn(move || println!("Handling new connection: {:?}", socket));
            }
            Err(e) => eprintln!("Error: {e}"),
        }
    }

    Ok(())
}
```

This function creates a [TcpListener](https://doc.rust-lang.org/std/net/struct.TcpListener.html) and anytime a new connection is opened with a client, it spawns a new thread to handle it.

Let's test it with telnet:
```bash
$ telnet 127.0.0.1 8000
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Connection closed by foreign host.
```

It worked, meaning that our basic TCP server handled a new connection.

### Reading an HTTP request
From now on, we will use Postman to craft WebSocket packets. Here is the download [link](https://www.postman.com/downloads/). The software is free.

Let's read our socket until the end of the HTTP header. We won't go through the [RFC 9112](https://datatracker.ietf.org/doc/html/rfc9112). We need to know that an HTTP header is terminated by the `\r\n\r\n`. Let's refactor our code a little bit. We will first create a handle_client function and a WebSocket structure.

Here is our new `run_server` function and the new `handle_client` function:

```rust
use std::net::TcpListener;

fn run_server(ip: &str, port: u32) -> std::io::Result<()> {
    let listener = TcpListener::bind(format!("{}:{}", ip, port))?;
    for stream in listener.incoming() {
        match stream {
            Ok(socket) => {
                // Instead of printing, we handle the client
                thread::spawn(move || handle_client(socket));
            }
            Err(e) => eprintln!("Error: {e}"),
        }
    }

    Ok(())
}

use std::net::{TcpListener, TcpStream};

fn handle_client(socket: TcpStream) {
    let mut websocket = WebSocket::new(socket);
    loop {
        // Let's keep it simple for now, the unwrap is a good 
        // way to focus on the logic first and a reminder of 
        // what we need to do.
        let payload = websocket.try_read_frame().unwrap(); 
        websocket.send_frame(payload);
        break
    }
}

```

Now that we know what we need, let's create our WebSocket module in a separate file. Don't forget to add the module in the main.

```rust
use std::net::TcpStream;

pub struct WebSocket {
    socket: TcpStream,
}

impl WebSocket {
    pub fn new(socket: TcpStream) -> Self {
        Self { socket }
    }

    pub fn try_to_read_frame(&mut self) -> Option<String> {
        todo!();
    }

    pub fn send_frame(&mut self, payload: String) {
        todo!();
    }
}
```

The `try_to_read_frame` and `send_frame` functions will be covered in the next article. Let's focus on the handshake. We will put the logic in our `new` function.
We want to read the first HTTP request and print it. Let's add some logic to that. Here is the rewrite of our `new` function.

```rust
    pub fn new(mut socket: TcpStream) -> Self {
        let buffer = read_http_request(&mut socket);
        println!("Request:\n{buffer}");
        Self { socket }
    }
```

It's pretty self-explanatory. Let's write the function `read_http_request`. This one won't be in our Struct. It's not directly related to the WebSocket. It does not need to manipulate any of the `WebSocket` struct attributes. It's just a convenience function to read a socket.

```rust
fn read_http_request(socket: &mut TcpStream) -> String {
    let mut retval = String::new();
    let mut buffer = vec![0; 1024];
    loop {
        // We don't handle the Err case. Our socket is blocking,
        // waiting for the whole request to come in.
        if let Ok(_) = socket.read(&mut buffer) { 
            let chunk = String::from_utf8_lossy(&buffer);
            // This is where we check if we reached the end of the header.
            if chunk.contains("\r\n\r\n") { 
                // We discard whatever is after the double \r\n. If the client is a WebSocket 
                //client, it won't send anything else.
                let mut last_chunk = chunk.split("\r\n\r\n");

                // Unwrap will stay, we know for sure that there is at least one \r\n\r\n
                let last_chunk = splits.next().unwrap(); 

                retval.push_str(last_chunk);
                break;
            } else {
                retval.push_str(&chunk);
            }
        }
    }
    retval
}
```

When we run the program and use Postman to send a request. Here is the output:

```bash
Request:
GET / HTTP/1.1
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: 7hKoimIDBLiE9aYdyn8amA==
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Host: 127.0.0.1:8000
thread '<unnamed>' panicked at src/main.rs:29:50:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

We can dismiss the error. We can see the basics of a WebSocket client's first request. It has the following lines:
* A legit first line
* header `Sec-WebSocket-Version` mandatory according to the RFC.
* header `Sec-WebSocket-Key` mandatory.
* header `Connection` mandatory.
* header `Upgrade` mandatory.
* header `Sec-WebSocket-Extensions`, this one is optional. We won't talk about it.

### Extract what we need and process the key
If we were to create a production-grade server, we would need to check headers and return an error if the client is incorrect. We just want to do an echo server. Let's skip that part and go straight to the point. All we want is the client's handshake key. Before going any further, let's write a plan a bit ahead. We want to extract the key and then craft an HTTP header that we will return to the client.

```rust
// We need to import the Write trait for write_all
use std::io::{Read, Write}; 

impl WebSocket{
    pub fn new(mut socket: TcpStream) -> Self {
        let client_request = read_http_request(&mut socket);
        let key = extract_client_key(client_request);
        let response = build_response(key);
        socket.write_all(response.as_bytes()).unwrap();
        Self { socket }
    }
    //...
}

fn extract_client_key(client_request: String) -> String {
    todo!()
}

fn build_response(key: String) -> String{
    todo!()
}

```

First, let's tackle the key extraction. If we look at what Postman sent us, we can determine how to extract the key. Let's get a look at it.

```bash
GET / HTTP/1.1
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: 7hKoimIDBLiE9aYdyn8amA==
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

It's straightforward, we get the lines and then split on `:`.

```rust
fn extract_client_key(client_request: String) -> String {
    for line in client_request.lines() {
        if line.contains("Sec-WebSocket-Key") {
            let mut splits = line.split(":");
            splits.next().expect("Error: header wrong format");
            return splits
                .next()
                .expect("Error: no value for websocket key")
                .trim()
                .to_string();
        }
    }
    panic!("Error: not a valid websocket upgrade request")
}
```

Again, we use a lot of `expect` to focus on the logic. We iterate through the lines of our request until one of them contains the __Sec-WebSocket-Key__ header.
When we find it, we split and return the value. Let's print the key in the next function.


```rust
fn build_response(key: String) -> String {
    println!("{key}");
    todo!()
}
```

If we use Postman to check the key against what we print, it works.

```bash
$ cargo run
rk7yy+5Lp5XJG3Q3zfAdlA==
thread '<unnamed>' panicked at src/main.rs:29:50:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### How to process the key for the client
The only information we need to respond is the `Sec-WebSocket-Key`. This is a string. We will discuss the client part in the next section on how to make it. The server just needs to process it a certain way. The RFC explains how to do it with an example. Let's quote it:
>A |Sec-WebSocket-Accept| header field.  The value of this header field is constructed by concatenating /key/, defined above in step 4 in Section 4.2.2, with the string "258EAFA5- E914-47DA-95CA-C5AB0DC85B11", taking the SHA-1 hash of this concatenated value to obtain a 20-byte value and base64-encoding (see Section 4 of [RFC4648]) this 20-byte hash.

To summarize, here is a pseudo code:
guid = "258EAFA5- E914-47DA-95CA-C5AB0DC85B11"
concatenated_key = concatenate(client_key, guid)
digest = sha1(concatenated_key)
processed_key = bas64.encode(digest)

The RFC explains why we do that. It is to be sure that a server can handle WebSockets. This is not for security reasons. The client will check the key returned by the server, and if the expected value is correct, it will start sending frames.

The RFC gives an example. Let's use it as a test case for our code. The client key is dGhlIHNhbXBsZSBub25jZQ==. At the end of the process, it will be: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=.

```rust
fn process_key(key: &str) -> String {
    "place holder".to_string()
}

#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn it_should_processed_key() {
        let client_key = "dGhlIHNhbXBsZSBub25jZQ==";
        let expected = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo=";

        let result = process_key(client_key);
        assert_eq!(result, expected);
    }
}
```

Everything is done in the `websocket.rs` file. We added the function we want with some dummy values. Let's run the test first, and you'll see that it fails. Let's write the logic now.
To make it work, we need to use the base64 and the sha1 crates. You can add the following to your **cargo.toml**:

```yaml
[dependencies]
base64 = "0.22.1"
sha1 = "0.10.6"
```
Let's write our key processing function.

```rust
use sha1::{Digest, Sha1};
use std::io::Read;

fn process_key(key: &str) -> String {
    let guid = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
    let concatenated_key = format!("{}{}", key, guid);
    let mut hasher = Sha1::new();
    hasher.update(concatenated_key);
    let digest = hasher.finalize();
    BASE64_STANDARD.encode(digest)
}

```

When we run our test, we can see that it works. The first time I tried to do that, it took me something like 4 or 5 hours. I was making a HashMap of all the headers. For some reason, I decided to lowercase every key/value entry. That was a very bad idea, even though my unit test passed, the program was returning the wrong key because I was making the server key from a lowercased version of the client's key. When someone writes something in an RFC, you stick to the spelling and do not try to be smarter. I thought that using capital letters was error-prone. I've learned my lesson. From now on, I will stick to the standard as much as I can.

### Writing our response request.
According to the RFC, the server must respond with an HTTP packet that contains a response line and headers ended by `\r\n\r\n`.

```
HTTP/1.1 101 Switching Protocols\r\n
Upgrade: websocket\r\n
Connection: Upgrade\r\n
Sec-WebSocket-Accept: KEY\r\n
\r\n
```

The first line is typical of an HTTP request. We indicate the HTTP version, the status code, and the status code description. The `KEY` is our processed key.

```rust
fn build_response(key: String) -> String {
    let server_key = process_key(&key);
    format!(
"HTTP/1.1 101 Switching Protocols\r\n\
Upgrade: websocket\r\n\
Connection: Upgrade\r\n\
Sec-WebSocket-Accept: {}\r\n\r\n
",
        server_key
    )
}
```

If we try it with Postman, we see that it managed to connect and was disconnected right away because our handle client thread stops. 

## The client handshake
The code for the client is to be written in another project.

### Implementing a basic TCP connection handler
Let's write how we would like to use the WebSocket client in our main.

```rust
use crate::websocketclient::WebSocketClient;
mod websocketclient;

fn main() {
    let mut websocket = WebSocketClient::new("127.0.0.1", 8000);
    websocket.send_frame("Hello, World");
    let response = websocket.read_frame();
    println!("Response");
}
```

First, we want to create a WebSocket and initialize it in the `new` function. Then we would like to send and read something. Let's write our basic WebSocketClient.


```rust
use std::net::TcpStream;

pub struct WebSocketClient {
    socket: TcpStream,
}

impl WebSocketClient {
    pub fn new(ip: &str, port: i32) -> Self {
        let socket = TcpStream::connect(format!("{}:{}", ip, port))
            .expect("Error: impossible to connect to remote");
        Self { socket }
    }

    pub fn send_frame(&mut self, payload: &str) {
        todo!()
    }

    pub fn read_frame(&mut self) -> String {
        todo!()
    }
}
```

The `send_frame` and `read_frame` will be covered in the frame section. Let's focus on how to make the handshake on the client side.

### Writing the client's handshake overall flow 
We want to send an HTTP request to the server with the client's key. We can study what the postman sent to us when we were working on the server side.


```bash
GET / HTTP/1.1
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: 7hKoimIDBLiE9aYdyn8amA==
Connection: Upgrade
Upgrade: websocket
# Extensions are optional
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Host: 127.0.0.1:8000
```

The header contains a hard-coded value. The `Connection: Upgrade` indicates to the server that we want to upgrade the current HTTP connection to a WebSocket connection. It is useful when interacting with a full-fledged HTTP server. When receiving the `Upgrade` and `Upgrade: Websocket`, the server will know what to do. We also need to craft a key. Let's write the code of our flow with some basic function definitions.

```rust
use std::io::Write;

impl WebSocket{
    pub fn new(ip: &str, port: i32) -> Self {
        let socket = TcpStream::connect(format!("{}:{}", ip, port))
            .expect("Error: impossible to connect to remote");
        let key = generate_key();
        let request = build_request(key);
        socket.write_all(request.as_bytes()).unwrap();
        check_server_response(&mut socket);
        Self { socket }
    }
    //...
}

fn generate_key() -> String {
    todo!()
}

fn build_request(key: &str) -> String {
    todo!()
}

fn check_server_response(socket: &mut TcpStream) {
    todo!()
}
```

### How to generate the key
Let's quote the RFC first:
> The request MUST include a header field with the name |Sec-WebSocket-Key|. The value of this header field MUST be a nonce consisting of a randomly selected 16-byte value that has been base64-encoded (see Section 4 of [RFC4648]). The nonce MUST be selected randomly for each connection.

In pseudo code, the generation looks like this.

```rust
let nonce = generate_16_byte_long_string();
base64.encode(nonce)
```

First, we will need the base64 library we use for the server and the rand library. Let's add that in our `cargo.toml`

```toml
rand = "0.9.0"
base64 = "0.22.1"
```

If we transform the pseudo code into actual Rust code, it will look like the following.

```rust
fn generate_key() -> String {
    let mut rng = rand::rng();
    let mut key = String::new();

    for _ in 0..16 {
        key.push(rng.random::<char>());
    }
    BASE64_STANDARD.encode(key)
}

```

 ### Sending the handshake init request
We need to take the HTTP header we saw earlier, remove the optional header, and put our freshly generated key.

```rust
fn build_request(key: &str) -> String {
    let request = format!(
"GET / HTTP/1.1\r\n\
Sec-WebSocket-Version: 13\r\n\
Sec-WebSocket-Key: {key}\r\n\
Connection: Upgrade\r\n\
Upgrade: websocket\r\n\
Host: 127.0.0.1:8000\r\n\r\n"
    )
}
```

We will use our own server to interact with. It makes things easier. We don't have to respect the HTTP RFC. Let's write the part that reads the server response.

```rust
fn check_server_response(socket: &mut TcpStream) {
    let response = read_server_http_response(socket).unwrap();
    todo!("Check response")
}

fn read_server_http_response(socket: &mut TcpStream) -> Option<String> {
    let mut buffer = vec![0; 1024];
    let mut response = String::new();
    loop {
        if let Ok(_) = socket.read(&mut buffer) {
            let chunk = String::from_utf8_lossy(&buffer);
            if chunk.contains("\r\n\r\n") {
                let chunk = chunk.split("\r\n\r\n").next().unwrap();
                response.push_str(&chunk);
                return Some(response);
            }
            response.push_str(&chunk);
        } else {
            break;
        }
    }
    None
}
```

We took the same logic we use in the server to parse the HTTP request. We read the stream until we find `\r\n\r\n` and we return the response.
Here is the server's response to our upgrade request.

```bash
$ cargo run
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: LfBTW9X7bjL2jHlPdymxQ8kJ7tk=
```

We recognize the header we wrote earlier. We're right on track. Our client needs to check if the server's key is correct.
Let's first write how we want the logic to work, according to what we know. We want first to extract the key. Then we use the key we sent to the server to compare with what the server returns. Thus, we need to transform the key we sent the same way the server did and compare it with the server's result. Let's replace the `todo!()`.

```rust
fn check_server_response(socket: &mut TcpStream, key: String) {
    let response = read_server_http_response(socket).unwrap();
    let client_key = extract_server_key(&response);
    let control_key = get_control_key(&key);
    if client_key != control_key {
        panic!("Error: wrong Sec-WebSocket-Accept key");
    }
}

fn extract_server_key(response: &str) -> String {
    todo!()
}

fn get_control_key(key: &str) -> String {
    todo!()
}
```

First, we want to extract the server key. We already know how to do that.

```rust
fn extract_server_key(response: &str) -> String {
    for line in response.lines() {
        if line.contains("Sec-WebSocket-Accept") {
            let mut splits = line.split(":");
            splits.next().expect("Error: header wrong format");
            return splits
                .next()
                .expect("Error: no value for websocket key")
                .trim()
                .to_string();
        }
    }
    panic!("Error: not a valid websocket upgrade response")
}
```

Be careful, we are looking for `Sec-WebSocket-Accept`, which is a different header name than the one our client used to send the key.
The control key follows the same logic as the server. We need to add the `sha` library to our' cargo. toml'.

```toml
[dependencies]
rand = "0.9.0"
base64 = "0.22.1"
sha1 = "0.10.6"
```

Here is the code.

```rust
use sha1::{Digest, Sha1};

fn get_control_key(key: &str) -> String {
    let guid = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
    let concatenated_key = format!("{}{}", key, guid);
    let mut hasher = Sha1::new();
    hasher.update(concatenated_key);
    let digest = hasher.finalize();
    BASE64_STANDARD.encode(digest)
}
```

Let's add a print in our main to see if it works. If it doesn't, the program will crash because the control key is not like the server key.

```rust
fn main() {
    let mut websocket = WebSocketClient::new("127.0.0.1", 8000);
    println!("Handshake is done");
    ...
}
```

Let's try it.

```bash
$ cargo run
Handshake done
```

It worked. Nice job. We just made a working WebSocket handshake. From now on, our server and client will only communicate using frames. We'll see what it looks like in the next article.

## Conclusion

The huge benefit of WebSocket is that, once the handshake is done, the TCP/TLS connection stays open. These two protocols carry some overhead. They have a handshake on their own, and the TLS adds some encryption. When we use WebSockets, we don't have to redo the TCP and TLS handshake, which saves time. It's faster.
