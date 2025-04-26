# Writing a ping tool 1: basic icmp echo

This article will lay down the basic structure of a ping tool. We will cover the following:
* DNS resolution
* Socket configuration
* Sending a ICMP packet and receiving the echo response.

## What is ICMP and how does it works with ethernet and IP

ICMP is used in complement to the Internet Protocol to manage some problematic situation. If a host cannot be reached for example, the node in the network that was supposed to forward the IP packet return a ICMP message to the senderto report the problem. In our case, we will use ICMP to make echo message.

The [RFC 792](https://datatracker.ietf.org/doc/html/rfc792) is used as a reference. The part that we are interested in is __Echo or Echo Reply Message__ page 13. I'm not super good at understanding RFC. I need to have some representation or concrete data to understand the thing. Especially, the RFC says:
> CMP messages are sent using the basic IP header.  The first octet of the data portion of the datagram is a ICMP type field...

While I was programming the thing for the first time in C, my understanding was that the ICMP data comes before IP header. When I had to parse the receiving data, I realized that was the opposite. Let's see how a packet is arriving. We will use tshark to sniff ICMP packet.
```bash
sudo tshark -f "icmp" -x
```
* -f : this option tells tshark to discriminate icmp packet
* -x : this ask tshark to display raw data in hexadecimal.

Let's ping Cloudflare dns server.
```bash
ping 1.1.1.1
```

Here is the tshark output:
```bash
Capturing on 'wlo1'
0000  bb bb bb bb bb bb aa aa aa aa aa aa 08 00 45 00   ................
0010  00 54 af d7 40 00 40 01 53 b5 c0 a8 74 72 01 01   .T..@.@.S...tr..
0020  01 01 08 00 db 5e 8e 4a 00 01 64 21 0d 68 00 00   .....^.J..d!.h..
0030  00 00 5d f9 00 00 00 00 00 00 10 11 12 13 14 15   ..].............
0040  16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25   .......... !"#$%
0050  26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35   &'()*+,-./012345
0060  36 37                                             67

0000  aa aa aa aa aa aa bb bb bb bb bb bb 08 00 45 28   ...............(
0010  00 54 02 6c 00 00 32 01 4e f9 01 01 01 01 c0 a8   .T.l..2.N.......
0020  74 72 00 00 e3 5e 8e 4a 00 01 64 21 0d 68 00 00   tr...^.J..d!.h..
0030  00 00 5d f9 00 00 00 00 00 00 10 11 12 13 14 15   ..].............
0040  16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25   .......... !"#$%
0050  26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35   &'()*+,-./012345
0060  36 37                                             67
```
The first paragraph is the packet we sent. The second, the packet we received. In this, we have part of the ethernet protocol, the IP header and ICMP packet. Let's find out what is what.
I started to look at how the ethernet protocol is composed but, it seems that we don't have the preamble. If we look at the first six bytes of our packet, we recognized our MAC address. We can verify that by writing `ip address show` in the terminal. If we look at the ethernet protocol, they says that (I put what we have for the first packet):
* the first 8 bytes are for preamble and SFD, (we clearly don't have these with the Tshark output).
* 6 bytes: destination (bb bb bb bb bb bb)
* 6 bytes: source (aa aa aa aa aa aa which is my mac address)
* type/length: (08 i suspect it to be the type more than the length)
* data (...)
I changed my mac address and the destination address

The data part contains the IP header and our ICMP data. We can recongnized the IP header begining. It starts with 45. The 4 stands for IPV4 and 5 is the minium length of a IP header. The length is expressed in 32-bits words. It means here that the IP header is 5 * 32-bits words. In byte, it is 5 * 4 bytes which makes the header 20 bytes long. We can count them to eliminate the IP header and go straight to the ICMP header.
Here is the ICMP data:
For the egress packet
```bash
08 00 db 5e 8e 4a 00 01 
```
For the ingress packet:
```bash
00 00 e3 5e 8e 4a 00 01 
```
If we look at the ICMP rfc, we can confirm that an echo packet starts with 8 and an echo reply starts with 0.
```bash
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Identifier          |        Sequence Number        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Data ...
+-+-+-+-+-
```
We have confirmation that the IP header comes first and the ICMP comes next. If we pay more attention to the IP header, we can recognize that, 4 bytes before the ICMP, we have our source and desctination address:
```bash
01 01 01 01 # which is the Cloudflare dns
c0 a8 74 72 # which is a local network IP address starting by 192.168
```
Next, we'll check what C has to offer to send this kind of packet.

## The C socket API

The main way we create network program is by using the `socket` function.
```c
int socket(int domain, int type, int protocol);
```
It returns a file description for the socket and takes the following parameters:
* domain => specify what kind of socket, in our case, internet protocol V4 type
* type => what kind of socket, if you want a classic TCP, you'll choose `SOCK_STREAM`, in our case it will be `SOCK_RAW`
* protocol => in our case, it will be IPPROTO_ICMP

It's the first time I use `SOCK_RAW`. Anytime you launch the program, you need to be root or use sudo. I forgot that aspect when I first finished working on that program and it would not configure my socket correctly.

Here is our plan:
* get the ip destination
* initialize the socket and configure it
* build and send a simple ICMP packet
* receive and parse the response.

Let's write our main
```c
#include <stdio.h>

struct sockaddr_in get_ip_target(char *ip);
int init_socket(struct sockaddr_in addr);
void send_ping(int sockfd);
void handle_echo(int sockfd);

int main(int argc, char **argv) {
    if (argc != 2) {
        printf("Usage: sudo ./ping IP_ADDRESS\n");
        return (1);
    }

    // struct sockaddr_in ip = get_ip_target(argv[1]);
    //  int sockfd = init_socket(ip);
    //  send_ping(sockfd);
    //  handle_echo(sockfd);
}
```
I commented out the part that are not yet written. It gives us an overall idea of the flow. Let's start with the `get_ip_target`

### Get the sockaddr_in ip structure
When we work with ip socket in c, we need to manipulate some struct. It's easy to be confused here. Most of the functions we will manipulate take as a parameters a `sockaddr` struct. It's the case for example for `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`. Here is how the man page define `sockaddr`:
```c
struct sockaddr {
           sa_family_t     sa_family;      /* Address family */
           char            sa_data[];      /* Socket address */
       };
```
You cannot define things this way, it would be to tedious. Instead we use some specialized struct such as `sockaddr_in`:
```c
       struct sockaddr_in {
           sa_family_t     sin_family;     /* AF_INET */
           in_port_t       sin_port;       /* Port number */
           struct in_addr  sin_addr;       /* IPv4 address */
       };
```
It's easier, we can assign the family, the port and ip address using the struct field.

And sockd_addr_in6 for IPv6:
```c
       struct sockaddr_in6 {
           sa_family_t     sin6_family;    /* AF_INET6 */
           in_port_t       sin6_port;      /* Port number */
           uint32_t        sin6_flowinfo;  /* IPv6 flow info */
           struct in6_addr sin6_addr;      /* IPv6 address */
           uint32_t        sin6_scope_id;  /* Set of interfaces for a scope */
       };
```

I have the feeling that C programmer loves to type cast and make structure. I have the feeling that it is some kind of polymorphism. `sockaddr` would be the abstract struct and `sockaddr_in` and `sockaddr_in6` would be concrete class.

Let's write the function that will build our `sockaddr_in` struct:
```c
struct sockaddr_in get_ip_target(char *ip) {
    struct sockaddr_in addr;

    addr.sin_family = AF_INET;
    addr.sin_port = htons(0);
    if (inet_pton(AF_INET, ip, &(addr.sin_addr)) != 1) {
        fprintf(stderr, "Error: ip address is not formatted correctly: %s \n",
                ip);
        exit(1);
    }
    return addr;
}
```
AF_INET is a maccro that represent IPv4.
`htons`, according to the man page, do the following:
> The  htons()  function converts the unsigned short integer hostshort from host byte order to network byte or‐

And finally, `inet_pton` will build the address structure from the char array for us.

We have a usable `sockaddr_in` struct, we then need to initialize a socket for ICMP.

### The socket initialization

We first need to create a socket. Here is the code we'll used:
```c
int sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
```

We already know about `AF_INET`, it tells the operating system that we want an IPv4 socket. We also want s `SOCK_RAW` socket. We will be able to manipulate the ICMP header and receive the IP header. Then we precise the protocol by writing `IPPROTO_ICMP`.

When we run the ping program, we can see different information:
```bash
$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=50 time=97.5 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=50 time=122 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=50 time=142 ms
```
We see the icmp_seq which is the ping number. We see the time it takes for the round trip. We also see ttl=50. It is a requirement for the IP header. This is the time to live when it reaches 0, the datagram is discarded. It is to avoid having packet running around forever. We need to set a TTL for our socket. Here is the code:
```c
int ttl = 64;
if (setsockopt(sockfd, SOL_IP, IP_TTL, &ttl, sizeof(ttl)) != 0) {
    fprintf(stderr, "Error: impossible to set ttl for socket\n");
    exit(1);
}
```
The `setsockopt` function takes the following argument:
* the socket
* the level of the option, in our case, this is `SOL_IP`, the configuration is on the IP header
* the actual value
* the size of the value

We then need to set some options to our socket. We are not serving anything, so we want our socket to block when sending something and waiting for something to return. But we don't want the program to hang forever if there are no response. We want then to set a timeout that will stop the program to wait from the system the response packets. Here is the code.
```c
struct timeval timeout;
timeout.tv_sec = 1000000;
timeout.tv_usec = 0;

if (setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, (void *)&timeout,
               sizeof(struct timeval)) != 0) {

    fprintf(stderr, "Error: impossible to set timeout for socket\n");
    exit(1);
}
```
This is the same function call but we different value. You may have noticed that the nature of the option is different, therefore, the fourth argument has a different type. This is why we pass a pointer to the function along the size of the parameters. The function will be able to read the option value by knowing the option number and the size of the value. You also notice that we use `SOL_SOCKET` because this option is to be applied at the socket level. If you want to dig more on the matter, here is a website that has more information on this function interface: [setsockopt](https://pubs.opengroup.org/onlinepubs/000095399/functions/setsockopt.html).

Here is the complete code:
```c
int init_socket(struct sockaddr_in addr) {
    int sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);

    int ttl = 64;
    if (setsockopt(sockfd, SOL_IP, IP_TTL, &ttl, sizeof(ttl)) != 0) {
        fprintf(stderr, "Error: impossible to set ttl for socket\n");
        exit(1);
    }

    struct timeval timeout;
    timeout.tv_sec = 1000000;
    timeout.tv_usec = 0;

    if (setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, (void *)&timeout,
                   sizeof(struct timeval)) != 0) {

        fprintf(stderr, "Error: impossible to set timeout for socket\n");
        exit(1);
    }
    return sockfd;
}
```

### Building and sending our packet
In order to build an ICMP packet, we need to use the `icmphdr` struct. I found the definition on this [website](https://sites.uclouvain.be/SystInfo/usr/include/netinet/ip_icmp.h.html).
We add to that a message randomly genereated. Here is the code for our packet initializer.

```c
typedef struct {
    struct icmphdr header;
    char message[64 - sizeof(struct icmphdr)];
} Packet;

Packet init_packet() {
    Packet packet;

    bzero(&packet, sizeof(Packet));
    packet.header.type = ICMP_ECHO;
    packet.header.un.echo.sequence = getpid();
    packet.header.un.echo.sequence = 0;
    packet.header.checksum = calculate_checksum(&packet);

    for (int i = 0; i < 10; i++) {
        packet.message[i] = (char)i + ' ';
    }

    return packet;
}
```

It's pretty straightforwar. The `un` in `packet.header.un` is the union that you can see in the struct source code. The same memory space is used either for ICMP echo or for MTU. It's not possible to do both.

We don't have a `calculate_checksum` function yet. This one was tricky. The best explanation I found was on the actual [ping source code](https://github.com/dgibson/iputils/blob/8f6403b0f7589d664f5764e9de3af15dd6e45aa8/ping.c#L903). There is [a video](https://www.youtube.com/watch?v=EmUuFRMJbss) I found that explain the maths behind the checksum. I won't explain the algorithm.

```c
uint16_t calculate_checksum(void *packet) {
    uint16_t *buffer = packet;
    uint16_t sum = 0;
    int max = sizeof(Packet);

    for (int i = 0; i < max; i += 2)
        sum += *buffer++;
    if (max % 2 != 0) {
        sum += *(unsigned char *)buffer;
    }

    sum = (sum >> 16) + (sum & 0xFFFF);
    sum += (sum >> 16);

    return ~sum;
}
```
After building our packet, we need to send it. We did not use the `connect` function. We need to tell the system the destination. We will use `sendto`. Here is the code.

```c
uint16_t calculate_checksum(void *packet);
Packet init_packet();

void send_ping(int sockfd, struct sockaddr_in addr) {
    Packet packet = init_packet();

    if (sendto(sockfd, &packet, sizeof(Packet), 0, (struct sockaddr *)&addr,
               sizeof(addr)) == -1) {
        fprintf(stderr, "Error: failed to send packet\n");
        exit(EXIT_FAILURE);
    }
}
```
