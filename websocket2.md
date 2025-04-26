# Writing a WebSocket echo server in Rust 2: handle frame

So far, the server's and client's logic were different enough that we kept them separate. The frame handling is similar. It's time we gather some logic into a library.
## Gathering the websocket logic into a library.

```bash
$ cargo new websocket_rs - lib
```

### Create the files
We will create two files for the client and server:
* websocket_client.rs
* websocket_server.rs
Then we copy and paste what we have in our different projects that were in the websocket files.
Let's write in our lib.rs the module declarations:

```rust
#![allow(dead_code)]
mod websocket_client;
mod websocket_error;
mod websocket_server;
```

I allowed __dead_code__ to silence the warnings at the library level. We don't use these in this library project.

### Handling error correctly
Since we want to provide our client and server with a library, we should handle our errors correctly. A library user does not expect the program to crash when they call our library. We have to let them know when there is a problem. We will use the `Result<Self, Box<dyn error::Error>>` pattern. It means we need to create a data structure that implements the trait Error.
Let's add a file called webosocket_error.rs and put an enum that will be used to return errors.

```rust
use std::fmt;
#[derive(Debug, Clone)]
pub enum WsError {
     InvalidHandshakeKey,
     MissingSecWebSocketAcceptHeader,
}

impl std::error::Error for WsError {}

impl fmt::Display for WsError {
     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
         match self {
         WsError::InvalidHandshakeKey => write!(f, "Invalid key for handhsake"),
         WsError::MissingSecWebSocketAcceptHeader => {
         write!(f, "Server respond without a Sec-WebSocketAccept header")
         }
     }
 }
}
```

There is nothing much to explain, I always look up at a website to remember how to do that.
Let's update our server client's new` function:

```rust
 pub fn new(ip: &str,did port: i32) -> Result<Self, Box<dyn error::Error>> {
 let mut socket = TcpStream::connect(format!("{}:{}", ip, port))?
 let key = generate_key();
 let request = build_request(&key);
 socket.write_all(request.as_bytes())?;
 check_server_response(&mut socket, key)?;
 Ok(Self { socket })
 }
```

We replace the `unwrap()` and `expect(…)` functions with an error forwarding operator `?`. We need to handle errors in `check_server_response`. Let's have a look.

```rust
fn check_server_response(socket: &mut TcpStream, key: String) -> Result<(), Box<dyn error::Error>> {
 let response = read_server_http_response(socket)?;
 let client_key = extract_client_key(&response);
 let control_key = get_control_key(&key);
 if client_key == control_key {
 Ok(())
 } else {
 Err(Box::new(WsError::InvalidHandshakeKey))
 }
}
```

The new version returns a `Result<(), Box<dyn error::Error>`. The error can return our new WsError enum or an IO error.
We are not done with this part, the `extract_client_key` can return an error in case of an invalid HTTP header in the server's response. Here is the new version:

```rust
fn extract_client_key(response: &str) -> Result<String, Box<dyn error::Error>> {
 for line in response.lines() {
 if line.contains("Sec-WebSocket-Accept") {
 let splits = line.split(":").collect::<Vec<&str>>();
 if splits.len() != 2 {
 return Err(Box::new(WsError::Header(line.to_string())));
 }
 return Ok(splits[1].trim().to_string());
 }
 }
 Err(Box::new(WsError::MissingSecWebSocketAcceptHeader))
}
```

We check the validity of our header by counting the number of splits we have. There is a problem. If the value of the header contains a :, this check will return an error. Let's improve that.

```rust
fn extract_client_key(response: &str) -> Result<String, Box<dyn error::Error>> {
 for line in response.lines() {
 if line.contains("Sec-WebSocket-Accept") {
 if let Some((_, value)) = line.split_once(":") {
 return Ok(value.trim().to_string());
 } else {
 return Err(Box::new(WsError::Header(line.to_string())));
 }
 }
 }
 Err(Box::new(WsError::MissingSecWebSocketAcceptHeader))
}
```

The fix is easy, we use `split_once` which returns an option with a tuple of `&str`. Rust has some very good utility functions for parsing! Let's grep the file one last time for `unwrap` and `expect`. Everything is fine. I leave you as an exercise to transform the server file with the error handling system we put in place.
Now that we have a nice library, let's start working on the frame handling.

## Handle frame

We will test our frame handling logic using Postman. We just need to connect to our server and send messages. The connection won't hold because we don't handle Ping and Pong. It's not a big deal. First, we need to add our library to our server project.

### Import the library locally

Let's edit our `cargo.toml` in the server project and add to the dependencies the following:

```toml
websocket_rs = { path="../websocket_rs" }
```

Let's refactor our `main`:

```rust
use std::net::{TcpListener, TcpStream};
use std::thread;
use websocket_rs::websocket_server::WebSocketServer;
//…
fn handle_client(socket: TcpStream) {
     let mut websocket = WebSocketServer::new(socket).unwrap();
     loop {
         let payload = websocket.try_read_frame().unwrap();
         websocket.send_frame(payload).unwrap();
         break;
     }
}
```

Rust Analyzer tells us that we have an error: module `websocket_server` is private. We need to make our modules public. Let's modify `lib.rs`:

```rust
pub mod websocket_client;
pub mod websocket_error;
pub mod websocket_server;
```

### Read a frame

The information is all written in the [rfc](https://datatracker.ietf.org/doc/html/rfc6455). Let's see what a frame looks like:

```bash
 0 1 2 3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+ - - - -+-+ - - - - - - -+ - - - - - - - - - - - - - - - -+
 |F|R|R|R| opcode|M| Payload len | Extended payload length |
 |I|S|S|S| (4) |A| (7) | (16/64) |
 |N|V|V|V| |S| | (if payload len==126/127) |
 | |1|2|3| |K| | |
 +-+-+-+-+ - - - -+-+ - - - - - - -+ - - - - - - - - - - - - - - - +
 | Extended payload length continued, if payload len == 127 |
 + - - - - - - - - - - - - - - - + - - - - - - - - - - - - - - - -+
 | |Masking-key, if MASK set to 1 |
 + - - - - - - - - - - - - - - - -+ - - - - - - - - - - - - - - - -+
 | Masking-key (continued) | Payload Data |
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 : Payload Data continued … :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 | Payload Data continued … |
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -+
 ```

Each column represents a bit. Here is a basic explanation:
* First bit: the first one is set to 0 if this is the last frame of a multi-frame transfer. Otherwise 1.
* The next 3 bits: we can ignore them for now, they are for extension handling.
* Opcode: it's the type of frame we have. 0b0001 is for a text frame, for example.
* Payload length: This is a tricky one. The length can be encoded on multiple bytes depending on this value.
* Mask: Every client payload has to be masked. The server does not need to mask.
* Payload: frame content.

We will proceed iteratively. First, we'll take a naive approach, where we read all the bytes in a buffer, assuming the whole frame is in it. We will lay the ground for the frame parsing. In a second time, we will implement a more robust approach to handle frames that would be sent over multiple packets.

### A naive approach

We need to create some data structure. We want a WebSocket struct that will handle read and write over a TCP socket. We also need a Frame struct to hold information about the current frame.

```rust
use std::net::TcpStream;
pub struct WebSocket {
    socket: TcpStream,
}

impl WebSocket {
     pub fn new(socket: TcpStream) -> Self {
        Self { socket }
     }
}
```

We can replace the socket attribute in our WebSocketServer and client struct with a WebSocket struct. We also need to define the module in our lib.rs file.

```rust
// lib.rs
mod websocket;
pub mod websocket_client;
pub mod websocket_error;
pub mod websocket_server;
// In websocket_client.rs
use crate::websocket::WebSocket;
pub struct WebSocketClient {
 websocket: WebSocket,
}
impl WebSocketClient {
 pub fn new(ip: &str, port: i32) -> Result<Self, Box<dyn error::Error>> {
     /…
     Ok(Self {
     websocket: WebSocket::new(socket),
     })
 }
// In websocket_server.rs
use crate::websocket::WebSocket;
pub struct WebSocketServer {
    websocket: WebSocket,
}

impl WebSocketServer {
    pub fn new(mut socket: TcpStream) -> Result<Self, Box<dyn error::Error>> {
         //…
         Ok(Self {
         websocket: WebSocket::new(socket),
     })
 }
```

We first need to read the socket and then get the payload from the frame. Let's add that to our code.

```rust
impl WebSocket {
     pub fn new(socket: TcpStream) -> Self {
     Self { socket }
 }

pub fn read_frame(&mut self) -> Result<String, Box<dyn Error>> {
     let mut buffer = [0u8; 10_000];
     let _ = self.socket.read(&mut buffer)?;
     let payload = get_payload(buffer);
     Ok(payload)
     }
}

fn get_payload(buffer: [u8; 10_000]) -> String {
    todo!()
}
```

The parsing will happen in `get_payload`. I like to start by writing all the function calls. They do not exist, but it gives an idea of what we need. If we go back to the frame structure, we can ignore the first byte. We need to parse the length, get the mask, and unmask the payload. It looks like the following.

```rust
fn get_payload(buffer: [u8; 10_000]) -> String {
 let len = get_payload_len(&buffer);
 let mask = get_mask(&buffer);
 let payload = get_payload(&buffer, len);
 unmask_data(payload, mask)
}
```

This is the minimum that we need to get the payload. In this implementation, we assume that the client sends us text. We don't check the opcode. We don't handle ping and pong. It's bare. Let's implement these functions, and we'll iterate on the rest later.
Let's start with the `get_payload_len` function. First, let's write some tests.
The length is tricky. It all depends on byte 2, the payload length. If it is less than 126, the length will only be the payload length. If this byte equals 126, the length is encoded in the next two bytes. If it's 127, length is encoded in the next eight bytes. Therefore, we need to test these three cases.
Let's see the first and second cases.

```rust
#[cfg(test)]
mod test {
    use super::*;
    #[test]
     fn it_parse_basic_len_correctly() {
         // This frame carry: Hello
         // I built it using Postman and print them in the server 
         let buffer = [129, 133, 166, 51, 46, 40, 238, 86, 66, 68, 201];
         let len = get_len(&buffer);
         assert_eq!(len, 5)
     }
    #[test]
     fn it_parse_basic_len_correctly_double_size() {
         // The length is 128 and here is the payload.
         // Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Aenean commodo ligula eget dolor. Aenean massa. Cum sociis natoque pe
         let buffer = [
         129, 254, 0, 128, 234, 188, 150, 161, 166, 211, 228, 196, 135, 156, 255, 209, 153, 201,
         251, 129, 142, 211, 250, 206, 152, 156, 229, 200, 158, 156, 247, 204, 143, 200, 186,
         129, 137, 211, 248, 210, 143, 223, 226, 196, 158, 201, 243, 211, 202, 221, 242, 200,
         154, 213, 229, 194, 131, 210, 241, 129, 143, 208, 255, 213, 196, 156, 215, 196, 132,
         217, 247, 207, 202, 223, 249, 204, 135, 211, 242, 206, 202, 208, 255, 198, 159, 208,
         247, 129, 143, 219, 243, 213, 202, 216, 249, 205, 133, 206, 184, 129, 171, 217, 248,
         196, 139, 210, 182, 204, 139, 207, 229, 192, 196, 156, 213, 212, 135, 156, 229, 206,
         137, 213, 255, 210, 202, 210, 247, 213, 133, 205, 227, 196, 202, 204, 247, 202,
         ];
        let len = get_len(&buffer);
         assert_eq!(len, 128)
         }
}
```

I created these buffers using Postman and printed the bytes I received from the server. I also tested if these tests were failing. To make sure I have a failing test, I changed the length value. You may have noticed that we don't test when the payload is encoded in eight bytes. We tested the lower end of our two-bytes-long length. Let's test the upper end. We don't need to copy and paste all the frame's bytes. We need the first 4. I generated a 35,000-character-long text using a text generator and used Postman to send it to our server. Here are the first bytes that Postman sent:

```bash
[129, 254, 136, 184, 10, 8, 221, 131, 70, 103, 175, 230]
```

Our length is defined by 136 and 184. I made a test to see if those two were well-read. Here is the test:

```rust
 #[test]
 fn it_parse_basic_len_correctly_double_size_large() {
     // The following string is the payload. Length = 35000 sent by Postman.
     let buffer = [129, 254, 136, 184, 10, 8, 221, 131, 70, 103, 175, 230];
     let expected = 0b10001000_00000000 + 0b10111000;
     let len = get_len(&buffer);
     assert_eq!(len, expected);
     assert_eq!(len, 35000);
 }
```

It works! Perfect. Let's try to test an eight-long length.
We don't need to check for an 8-byte number. A 3-byte is enough. Here we can see an easy way to check by building the number using a bit representation. Let's craft this test by hand.
We need to craft the frame first. We don't care about the first byte. The second byte has to be 255 or 127. Let's use 255, it's interesting. If we choose 255, we get the following binary representation: 0b1111_1111. The first byte indicates that we mask the payload. The remaining bits are equal to 127, meaning that we encoded the length over 8 bytes.
The following two bytes are used to represent the 2-byte length. If we reuse the number from the last test and we turn to 1 the bit number 17, we have to represent our length in three bytes. Two bytes are not enough. The frame will use an 8-byte encoding. It looks like this:
[129, 255, 0, 0, 0, 0, 0, 1, 136, 184]
We are just interested in the 1, 136, and 184. Let's build the expected value using the pattern from the last test. We don't need to know the size; we just need to calculate the size and check if our function is able to decode the size when it's encoded on 8 bytes.

```rust
// this is our 1 136 184
let expected = 0b1_00000000_00000000 + 0b10001000_00000000 + 0b1011_1000;
```

We padded with zero accordingly to simulate the bit representation in memory. Here is the whole test:

```rust
 #[test]
 fn it_parse_basic_len_correctly_big_size() {
 let buffer = [129, 255, 0, 0, 0, 0, 0, 1, 136, 184, 175, 230];
 let expected = 0b1_00000000_00000000 + 0b10001000_00000000 + 0b10111000;
 let len = get_len(&buffer);
 assert_eq!(len, expected );
 }
```

 Perfect, we know that our function can handle big numbers without needing actual data!
I already explained how to decode the length. It all depends on the first 7 bits of byte 2. The way we read it is by getting the bytes one by one and shifting them according to their position to obtain their positional value. Let's see when the length is two bytes. Let's use the second test buffer as an example.

```bash
[129, 254, 136, 184]
```

The first byte is to be ignored. It does not count for the length. The second byte will tell us how to read the length. We need to extract the first 7 bits. This is done by ` 0b0111_1111 & buffer[1]`. In our case, the result is 126. We then read the next two bytes, which are 136 and 184. Let's use a binary representation of our number:
136 = 0b1000_1000
184 = 0b1011_1000
We know that this length is encoded in two bytes. It's a 16-bit integer. We therefore need to shift the first number by eight bits on the left and add the other number.
0b1000_1000_0000_0000 + 0b1011_1000 = 0b1000_1000_1011_1000
 136 * 2⁸ + 184 = 35_0000
The eight-byte length follows the same principle.

```rust
fn get_payload_len(buffer: &[u8]) -> usize {
     // We mask the first byte as it is not part of the length
     let size_byte = 0b01111111 & buffer[1];
     match size_byte {
     126 => ((buffer[2] as usize) << 8) + buffer[3] as usize,
     127 => {
     ((buffer[4] as usize) << (8 * 5))
     + ((buffer[5] as usize) << (8 * 4))
     + ((buffer[6] as usize) << (8 * 3))
     + ((buffer[7] as usize) << (8 * 2))
     + ((buffer[8] as usize) << (8 * 1))
     + (buffer[9] as usize)
     }
     _ => size_byte as usize, // easy case
     }
}
```

Now, let's get the mask. It will depend on the type of length we have. We know that it is a 4-byte-long number. We will return an array. It will be easier to use an array for the unmask step. Here is the function. I did a bunch of tests. You can check them out in the repo. I won't put them here to keep the article short.

```rust
fn get_mask(buffer: &[u8]) -> &[u8] {
     let byte_size = 0b0111_1111 & buffer[1];
     match byte_size {
     126 => &buffer[4..8],
     127 => &buffer[10..14],
     _ => &buffer[2..6],
     }
}
```
Then we get the payload. It will also depend on the length and size.
```rust
fn extract_payload(buffer: &[u8], len: usize) -> &[u8] {
     let byte_size = 0b0111_1111 & buffer[1];
     let start = match byte_size {
     126 => 2 + 2 + 6 + 4,
     127 => 2 + 2 + 4,
     _ => 2 + 4,
 };
 &buffer[start..(start + len)]
```

The last part is the unmasking process. I will quote the RFC.
> Octet i of the transformed data ("transformed-octet-i") is the XOR of
> octet i of the original data ("original-octet-i") with octet at index
> i modulo 4 of the masking key ("masking-key-octet-j"):
> j = i MOD 4
> transformed-octet-i = original-octet-i XOR masking-key-octet-j
Let's write a test first. We already have the buffer for a message that carry "Hello" let's reused it and check if our flow works.

```rust
 #[test]
 fn it_parse_a_basic_message() {
     // This frame carry: Hello
     let buffer = [129, 133, 166, 51, 46, 40, 238, 86, 66, 68, 201];
     let payload = get_payload(&buffer);
     assert_eq!(payload.as_str(), "Hello")
 }
```

Let's run the test and… it fails! Nice. Sometimes, you run your test and it passes although it's supposed to fail…
And now the code!

```rust
fn unmask_payload(payload: &[u8], mask: &[u8]) -> String {
     let mut unmasked_payload: Vec<u8> = Vec::new();
     for (i, char) in payload.into_iter().enumerate() {
         let j = i % 4;
         unmasked_payload.push(*char ^ mask[j]);
     }
     String::from_utf8_lossy(&unmasked_payload).to_string()
}
```

Let's run cargo test and… it passes. We successfully read a frame! You can go ahead and try with Postman. The `write_frame` will print the payload.

## Write frame

The first thing is to edit our `websocket_server struct.write_frame()` function. Let's call a function from our websocket struct

```rust
 pub fn send_frame(&mut self, payload: String) -> Result<(), Box<dyn error::Error>> {
     self.websocket.write_frame(payload)?;
     Ok(())
 }
```

The sending will be easier. Because we are working on a server first, we don't need to mask the payload. If we were to mask the payload, we could use the same function as unmask (we'd renamed it apply_mask()). The same operation works for masking and unmasking. We would just have to tell the function what we want to do. For now, let's skip that.
Let's see what the code looks like.

```rust
 //…
pub fn write_frame(&mut self, payload: String) -> Result<(), Box<dyn Error>> {
     let payload_bytes = payload.as_bytes();
     let mut frame: Vec<u8> = Vec::new();
    encode_fin(&mut frame); // fin is the first bit
     encode_opcode(&mut frame);
     encode_length(&payload_bytes, &mut frame);
     add_payload(&mut frame, &payload_bytes);
    self.socket.write_all(&frame)?;
     Ok(())
}

fn encode_fin(frame: &mut Vec<u8>) {
     // A frame can be sent in multiple fragment.
     // The first bit tells the client if this is the last fragment of a frame.
     // Because we have a one fragment long frame, this is also the first.
     frame.push(0b1000_0000);
}

fn encode_opcode(frame: &mut Vec<u8>) {
     let first_byte = frame.pop().unwrap();
     // This tells the client the payload is text.
     frame.push(0b1000_0001 | first_byte);
}

fn encode_length(payload_bytes: &[u8], frame: &mut Vec<u8>) {
    let payload_len = payload_bytes.len();
    if payload_len < 126 {
         frame.push(payload_len as u8);
     } else if payload_len <= 65_535 {
         frame.push(126);
         frame.extend_from_slice(&(payload_len as u16).to_be_bytes());
     } else {
         frame.push(127);
         frame.push(0);
         frame.push(0);
         frame.extend_from_slice(&(payload_len as u64).to_be_bytes());
     }
}
fn add_payload(frame: &mut Vec<u8>, payload_bytes: &[u8]) {
    frame.extend_from_slice(payload_bytes);
}
```

### Fix the bug

If you follow along here, it does not work. We have a bug. It took me two hours to find out why. Maybe you already know. I'm just gonna show you my debugging process.
If you tried to use Postman, you most likely encountered the following error(maybe you didn't because you implemented it correctly).
Here is the error: `Error: Invalid WebSocket frame: FIN must be set`.
To debug it, I used tshark, the command-line version of Wireshark. It allows you to look at the network traffic. It's a neat tool if you want to check what's happening on your local interface, and intercept only the websocket frame. you can run it using:

```bash
    tshark -i lo -Y "websocket.opcode == 10" -x
```

* -i lo => interface lo.
* -Y filter packet.
* -x displays raw bytes.
Here is what I had when I ran it while using Postman.

```bash
Frame (77 bytes):
0000 00 00 00 00 00 00 00 00 00 00 00 00 08 00 45 00 …………..E.
0010 00 3f 04 6a 40 00 40 06 38 4d 7f 00 00 01 7f 00 .?.j@.@.8M……
0020 00 01 82 44 1f 40 6a 55 c9 7a db b4 44 4d 80 18 …@jU.z..DM">D.@jU.z..DM..
0030 02 00 fe 33 00 00 01 01 08 0a 79 e7 7b 9d 79 e7 …3……y.{.y.
0040 77 26 81 85 f5 5e 53 e4 bd 3b 3f 88 9a w&…^S..;?..
Unmasked data (5 bytes):
0000 48 65 6c 6c 6f Hello
Frame (73 bytes):
0000 00 00 00 00 00 00 00 00 00 00 00 00 08 00 45 00 …………..E.
0010 00 3b ab 5a 40 00 40 06 91 60 7f 00 00 01 7f 00 .;.Z@.@..`……
0020 00 01 1f 40 82 44 db b4 44 4d 6a 55 c9 85 80 18 …@.D..DMjU….
0030 02 00 fe 2f 00 00 01 01 08 0a 79 e7 80 b9 79 e7 …/……y…y.
0040 7b 9d 81 05 48 65 6c 6c 6f {…Hello
Reassembled TCP (8 bytes):
0000 0a 81 05 48 65 6c 6c 6f …Hello
Unmasked data (1 byte):
0000 69 i
```

The first packet is sent by Postman. The second packet is our server. You need to look at the `Reassembled TCP (8bytes):`.
` 0000 0a 81 05 48 65 6c 6c 6f …Hello`
The 81 is the hexadecimal for 0b1000_0001. This is supposed to be the beginning of our frame. Instead, we have a 0a. At some point, I realised that 0a is ASCII for return to line. I forgot to escape the last return to line of our handshake string in `websococket_server`.

```rust
 format!(
     "HTTP/1.1 101 Switching Protocols\r\n\
    Upgrade: websocket\r\n\
    Connection: Upgrade\r\n\
    Sec-WebSocket-Accept: {}\r\n\r\n// This is where we have a problem. There is a hidden return to line.
     ", 
     server_key
 )
```

Here is the corrected code:

```rust
 format!(
     "HTTP/1.1 101 Switching Protocols\r\n\
    Upgrade: websocket\r\n\
    Connection: Upgrade\r\n\
    Sec-WebSocket-Accept: {}\r\n\r\n\ // We fix it by escaping using \
     ",
     server_key
 )
```

If you use Postman again, it works!

## Conclusion

We finally made an echo server. There is a lot we did not implement here. We covered the basics of WebSocket, and it gave us a look at how it works.
