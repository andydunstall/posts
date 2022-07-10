# UDP Implementation Using Raw Sockets in Golang
This article implements a write-only UDP ‘socket’ in Golang using raw sockets, which provide the application direct access to the IP layer. Note the kernel does not forward received UDP datagrams to raw sockets so to receive UDP in the application would require accessing the link layer directly

## UDP
UDP (User Datagram Protocol) is a very simple transport-layer on top of IP providing datagram-oriented transport which preserves message boundaries (unlike TCP which is stream-oriented), and typically provides error detection with a checksum in the UDP header. It provides no reliability, ordering, duplicate elimination, flow control or congestion control.

As defined in RFC 768, UDP simply prefixes the application data with an 8 byte header that includes:
* The destination port required for demultiplexing between sockets on the receiver,
* An optional source port which may be used for the return address by the application,
* The datagram length (including the size of the header) which is redundant when sent over IP since IP already includes the datagram length. I’m guessing this is to avoid making assumptions about the underlying network layer protocol (though the RFC states that it assumes it runs on top of IP),
* An optional checksum covering the UDP header, UDP payload and select IP fields (the protocol, source IP address and destination IP used to verify on the receiver that the datagram was routed correctly). Note although the checksum is optional it's almost always provided.

UDP has a few benefits over TCP:
* Less overhead: TCP would require a three-way handshake to setup the connection, buffers allocated for each connection, acknowledgements for each data message, and a further 4 packets to terminate the connection (assuming no packets piggyback one another). This is why simple protocols like DNS typically use UDP as it improves latency with fewer round trips and allows the server to handle more clients without the extra resources required to run over TCP.
* More control: Some applications like multimedia and real-time communication can tolerate loss of packets and may add redundant data to conceal packet loss, rather than wait for resends provided by TCP.
* No congestion control

## Raw Sockets
Raw sockets allow the application to directly access the IP layer. This enables:
* Reading and writing ICMP as used by the ping application.
* Reading and writing IPv4 datagrams when the IPv4 protocol field is not handled by the kernel. Note the kernel will not forward IP packets containing TCP and UDP to raw sockets so these protocols cannot be received, though can be sent as shown below. This makes sense as it cannot demultiplex between sockets without a port.
* Writing IPv4 header using `IP_HDRINCL` socket option, which if set the application is expected to write the whole IPv4 datagram rather than just the payload (though the kernel still writes a few IP fields), which is not needed in this case to send UDP. When `IP_HDRINCL` is not set the kernel will prefix the IPv4 header and use the protocol specified when creating the socket (UDP in this case) as the protocol field.

Bind can be called to set the interface, though given the IP layer has no concept of ports only the IP address is used. The same occurs with connect where this can set the destination IP meaning the application can use send rather than sendto but does not set the port.

When receiving data the kernel will check all processes for raw sockets that match the datagram received. This will only forward datagrams that have a protocol field the kernel doesn't recognise, and ICMP/IGMP, which means it will never forward TCP or UDP to raw sockets as this is processed inside the kernel. It will also check if the raw socket has called bind or connect, which if called will filter based on the IP datagrams destination and source IP to match those in bind and connect respectively. Also if the raw socket specified a protocol this must match the protocol field in the datagram.

Note only the superuser can create raw sockets, which makes sense to avoid snooping on other users packets, since if bind and connect are not called, and the protocol is unset it will receive all IPv4 datagrams except for TCP and UDP.

## Golang Implementation
Since UDP is such a simple protocol it's trivial to implement using raw sockets. However as mentioned above the kernel will not forward UDP or TCP packets to raw sockets so this is send only. Also this doesn’t add the UDP checksum as this is optional (though in practice obviously you'd usually want it).

A very simple implementation in Golang is given in [udpsock.go](https://gist.github.com/dunstall/d9ae1b9bdb5eacc33a617e0d645f6edb).

This opens a raw socket with the IPv4 protocol fields set to UDP using `net.DialIP(“ip4:udp”, laddr, raddr)`, which internally uses `syscall.SOCK_RAW` to create a raw socket. If `laddr` is given this will call bind with that address, and if `raddr` is given will call connect with that address. As this implementation is send-only, bind is redundant so only `raddr` is given. This also specifies “ip4” rather than “ip” since this is incompatible with IPv6.

To send data into the created raw socket the user data is just prefixed with the UDP header described above and sends it into the raw socket (since it's already called connect within `DialIP` it can just use `Send` rather than `SendTo`).

```go
package main

import (
    "encoding/binary"
    "fmt"
    "log"
    "net"
)

const (
    headerLen = 8
)

// UDP implements a UDP connection using raw sockets (send IP datagrams).
type UDPConn struct {
    ipConn   *net.IPConn
    srcPort  uint16
    destPort uint16
}

func Dial(ip string, srcPort uint16, destPort uint16) (*UDPConn, error) {
    ipConn, err := net.DialIP("ip4:udp", nil, &net.IPAddr{
        IP: net.ParseIP(ip),
    })
    if err != nil {
        return nil, err
    }
    return &UDPConn{
        ipConn:   ipConn,
        srcPort:  srcPort,
        destPort: destPort,
    }, nil
}

func (c *UDPConn) Write(b []byte) (int, error) {
    if len(b) > 0xffff-headerLen {
        return 0, fmt.Errorf("datagram payload too large (max 0xffff - 8)")
    }
    enc := c.header(uint16(len(b)))
    enc = append(enc, b...)
    return c.ipConn.Write(enc)
}

// header returns the encoded UDP datagram header. This header has format:
//  0      7 8     15 16    23 24    31
// +--------+--------+--------+--------+
// |     Source      |   Destination   |
// |      Port       |      Port       |
// +--------+--------+--------+--------+
// |                 |                 |
// |     Length      |    Checksum     |
// +--------+--------+--------+--------+
// Note includes a checksum of 0 to indicate no checksum.
func (c *UDPConn) header(payloadLen uint16) []byte {
    b := make([]byte, headerLen)
    binary.BigEndian.PutUint16(b[0:], c.srcPort)
    binary.BigEndian.PutUint16(b[2:], c.destPort)
    binary.BigEndian.PutUint16(b[4:], headerLen+payloadLen)
    // No checksum.
    binary.BigEndian.PutUint16(b[6:], 0)
    return b
}
```
