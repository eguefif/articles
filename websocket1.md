# Writing a WebSocket echo server in Rust

Websocket are a convenient way to maintain a connexion between a server and client using all the existing http infrastructure. In this article, we'll cover the basic of a working websocket echo server. We will cover the handshake and frame concept.
Here is a link to the [rfc 6455](https://datatracker.ietf.org/doc/html/rfc6455)
Here is the repo for the full code and branch step.

## The server Handshake
### A basic TCp server
I like to start with how I'm gonna use the code I will write. In our case, we want to run a server. Let's write our main:
```rust
fn main() -> std::io::Result<()> {
    run_server("127.0.0.1", 8000)?;
    Ok(())
}
```

Starting with the function helps us define our needs. By doing so, we also find our entry point. Let's add to this file the following function.
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

This function creates a [TcpListener](https://doc.rust-lang.org/std/net/struct.TcpListener.html) and anytime a new connection is opened with a client, it spawn a new thread to handled it.

Let's test it with telnet:
```bash
$ telnet 127.0.0.1 8000
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Connection closed by foreign host.
```
It worked meaning that our basic tcp server handle new connection.

### Reading a HTTP request
From now on, we will use Postman to craft WebSocket packet. Here is the download [link](https://www.postman.com/downloads/). The software is free.

Let's read our socket until the end of the Http header. We won't go through the [rfc 9112](https://datatracker.ietf.org/doc/html/rfc9112). We just need to know that a Http header is terminated by the \r\n\r\n. Let's refactor our code a little bit. We will first create a handle_client function and a WebSocket structure.

Here is our new `run_server` function and the new `handle_client` function:
```rust
use std::net::TcpListener;

fn run_server(ip: &str, port: u32) -> std::io::Result<()> {
    let listener = TcpListener::bind(format!("{}:{}", ip, port))?;
    for stream in listener.incoming() {
        match stream {
            Ok(socket) => {
                thread::spawn(move || handle_client(socket));
            }
            Err(e) => eprintln!("Error: {e}"),
        }
    }

    Ok(())
}
```
We want a websocket struct. Let's write how we want to use in our handle_client.
```rust
use std::net::{TcpListener, TcpStream};

fn handle_client(socket: TcpStream) {
    let mut websocket = WebSocket::new(socket);
    loop {
        let payload = websocket.try_read_frame().unwrap(); // Let's keep it simple for now, the unwrap is a good way to focus on the logic first and a reminder of what we need to do.
        websocket.send_frame(payload);
        break
    }
}
```

We now have a look of what we want. Let's write our websocket module in another file. Don't forget to add the module in the main.
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
        Some("Hey".to_string())
    }

    pub fn send_frame(&mut self, payload: String) {
        println!("Sending: {payload}");
    }
}
```
The `try_to_read_frame` and `send_frame` functions will be covered in the frame part of this article. Let's focus on the handshake. We will put the logic in our `new` function.
We want to read the first http request and print it. Let's add some logic that.

Here is the rewritte of our `new` function:
```rust
    pub fn new(mut socket: TcpStream) -> Self {
        let buffer = read_http_request(&mut socket);
        println!("Request:\n{buffer}");
        Self { socket }
    }
```
It's pretty self explanotory. Let's write the function `read_http_request`. This one won't be in our Struct. It's contain in the new function and is not related to the websocket. It's just a convenience function to read socket. Here is the code:
```rust
fn read_http_request(socket: &mut TcpStream) -> String {
    let mut retval = String::new();
    let mut buffer = vec![0; 1024];
    loop {
        if let Ok(_) = socket.read(&mut buffer) { // We don't handle the Err case. Our socket is blocking waiting for the whole request to come in.
            let chunk = String::from_utf8_lossy(&buffer);
            if chunk.contains("\r\n\r\n") { // This is where we check if we reached the end of the header.
                let mut splits = chunk.split("\r\n\r\n"); // What ever is after, we drop it. If the client is a websocket client, it won't send anything else.
                let last_chunk = splits.next().unwrap(); // Unwrap will stay, we know for sure that there is at least one \r\n\r\n
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
When we run the program and use postman to send a request. Here is the output:
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
We can dismiss the error. We can see the basic of a websocket client first request. It has the following lines:
* A legit first line
* header `Sec-WebSocket-Version` mandatory
* header `Sec-WebSocket-Key` mandatory
* header `Connection` mandatory
* header `Upgrade` mandatory
* header `Sec-WebSocket-Extensions`, this one is optional. We won't talk about it.

### Extract what we need and process the key
If we were to make a production grade server, we would have to check headers and return an error in case the client is wrong. We just want to do an echo server. Let's skip that part and go straight to the point. All we want is the client's key. Before going any further, let's write plan a bit aheader. We want to extract the key and then make the header that we will return to the client. Here is our new function:
```rust
use std::io::{Read, Write}; // We need to import the Write trait for write_all
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
    client_request
}

fn build_response(key: String) -> String{
    key
}

    ```

Let's tackle first the key extraction. If we look at what Postman sent to us, we can determined how we will extract the key. Let's get a look at it.
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
Again, we use a lot of `expect` in order to focus on the logic. We iterate thourgh the lines of our request until one of them contains the __Sec-WebSocket-Key__ header.
When we find it, we just splits and return the value. Let's print the key in the next function.
```rust
fn build_response(key: String) -> String {
    println!("{key}");
    key
}
```
If we use Postman to check, we print the key.
```bash
$ cargo run
rk7yy+5Lp5XJG3Q3zfAdlA==
thread '<unnamed>' panicked at src/main.rs:29:50:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### How to process the key for the client
The only information we need to respond is the `Sec-WebSocket-Key`. This is a string. We will talk in the client part how to make it. The server just need to process it a certain way. The rfc explain how to do it with an example. Let's quote it:
>A |Sec-WebSocket-Accept| header field.  The value of this header field is constructed by concatenating /key/, defined above in step 4 in Section 4.2.2, with the string "258EAFA5- E914-47DA-95CA-C5AB0DC85B11", taking the SHA-1 hash of this concatenated value to obtain a 20-byte value and base64-encoding (see Section 4 of [RFC4648]) this 20-byte hash.

To summary, here are a speudo code:
guid = "258EAFA5- E914-47DA-95CA-C5AB0DC85B11"
concatenated_key = concatenate(client_key, guid)
digest = sha1(concatenated_key)
processed_key = bas64.encode(digest)

The rfc explains why we do that. It is to be sure that a server is able to handle websocket. This is not for security reason. The client will check the key returned by the server and if the expected value is right, it will start sending frame.

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
Everything does in the `websocket.rs` file. We added the function we want with some dummy values. Let's run the test first and you'll see that it fails. Let's write the logic now.
To make it work, we need to use to crates. You can add the following to your **cargo.toml**:
```yaml
[dependencies]
base64 = "0.22.1"
sha1 = "0.10.6"
```
We first make the imports we need. `Digest` is a trait that we need for `Sha1::new`.

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
When we run our test, we can see that it works. The first time I tried to do that, it took me something like 4 or 5 hours. I was making a HashMap of all the headers. For some reason, I decided to lowercase every key/value. That was a very bad idea, eventhough my unit test pass, the program was returning the wrong key because I was making the server key from a lowercased version of the client's key. When someone write something in a RFC, you stick to the spelling and not try to be smarter. I thought that using capital letter was error prone. I've learned my lesson. From now on, I will stick to the standard as much as I can.

### Writing our response request.
According to the RFC, the server must respond with the following headers and first line:
```
HTTP/1.1 101 Switching Protocols\r\n
Upgrade: websocket\r\n
Connection: Upgrade\r\n
Sec-WebSocket-Accept: KEY\r\n
\r\n
```

The first line is typical from an Http request. We indicate the HTTP version, the status code and the status code description. The KEY is our processed key. Here is our code:
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

If we try it with Postman, we see that it managed to connect and was disconnected right awayt. 
