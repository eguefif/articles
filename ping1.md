# Writing a ping tool 1: basic icmp echo

This article will lay down the basic structure of a ping tool. We will cover the following:
* DNS resolution
* Socket configuration
* Sending a ICMP packet and receiving the echo response.

## ICMP
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
8e 4a 00 01 64 21 0d 68 
```
For the ingress packet:
```bash
00 00 00 00 5d f9
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

##
