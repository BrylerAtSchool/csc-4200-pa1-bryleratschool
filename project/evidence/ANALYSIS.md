# PA1 Capture Analysis

_Your capture, `evidence/pa1-capture.pcap`, holds two ETP sessions from your own
programs: a PUT, then a GET of the same file. This analysis is about the **GET
session** unless a question says otherwise. Use Wireshark (or tshark) to read the
capture. The numbers must come from your capture: they are checked against it._

## How we captured it

PUT then a GET of `evidence_payload.bin` (262,144) with a 200-byte snap length.

tcpdump -Z root -i lo -s 200 -U -w /tmp/pa1-capture-final.pcap 'tcp port 45678'
./etp-server --port 45678 --root server-files
./etp-client --host 127.0.0.1 --port 45678 --name evidence-client put scratch/evidence_payload.bin evidence_payload.bin
./etp-client --host 127.0.0.1 --port 45678 --name evidence-client get evidence_payload.bin downloads/evidence_payload.bin
ss -tan state time-wait '( sport = :45678 or dport = :45678 )'

## Facts

_Fill in every value from **your** capture, for the **GET** session. Integers in
decimal. Sequence numbers are the **absolute** (raw) values — in Wireshark, turn
off "relative sequence numbers" (Edit > Preferences > Protocols > TCP) or read
the "Sequence Number (raw)" field. With tshark, add
`-o tcp.relative_sequence_numbers:FALSE`. Do not change the key names or the
`etp-facts` label: a script reads this block._
```etp-facts
server_port: 45678
client_port: 33508
client_isn: 1082458682            # raw sequence number in the client's SYN
server_isn: 2414250255            # raw sequence number in the server's SYN/ACK
client_mss: 65495                 # MSS option value in the client's SYN, in bytes
server_to_client_payload_bytes: 262178  # HELLO_OK + GET_OK + BYE_OK TCP payload bytes
server_to_client_data_segments: 12      # server-to-client frames with TCP payload
first_fin_from: server
```

## Q1. The handshake

_Give the frame number, flags, and raw sequence and acknowledgment numbers of the
three handshake segments of the GET session. Explain why the acknowledgment number
in the third segment is what it is. What MSS did the client advertise, and what on
your machine determines that value?_

The GET connection is between client 127.0.0.1:33508 and server 127.0.0.1:45678.

| Frame | Flags | Raw sequence number | Raw acknowledgment number |
|---|---|---:|---:|
| 38 | SYN | 1082458682 | 0 |
| 39 | SYN, ACK | 2414250255 | 1082458683 |
| 40 | ACK | 1082458683 | 2414250256 |

The third frame acknowledges the server's sequence number plus one (2414250255 + 1). This is because SYN uses one sequence number.

The client advertised 65,495 bytes of MSS. This is based on the loopback network's MTU of 65,536 bytes, minus the 20-byte IPv4 header and 20-byte TCP header.

## Q2. Your protocol on the wire

_Find the segment that carries your client's GET request. Give its frame number,
paste the first 8 payload bytes in hex, and decode each header field (VERSION,
TYPE, STATUS, RESERVED, LENGTH). Do the same for the header of the server's
GET_OK. Does GET_OK's LENGTH equal the size of your file? Show the arithmetic that
connects `server_to_client_payload_bytes` to the three responses the server sent in
this session._

The client's GET header appears in frame 49. Its first eight payload bytes are 01 03 00 00 00 00 00 14
VERSION is 1
TYPE is 0x03 (GET)
STATUS is 0
RESERVED is 0
LENGTH is 0x00000014 (20 bytes)

The filename payload, evidence_payload.bin, follows in frame 51.

The server's GET_OK header is in frame 53: 01 83 00 00 00 04 00 00
VERSION is 1
TYPE is 0x83 (GET_OK)
STATUS is 0
RESERVED is 0
LENGTH is 0x00040000 (262,144 bytes). this matches the file size.

The server's TCP payload byte total is 18 + 262,152 + 8 = 262,178:
HELLO_OK has an 8-byte header + a 10-byte body + GET_OK has an 8-byte header + the 262,144-byte file + BYE_OK has an 8-byte header.

## Q3. A byte stream, not messages

_How many segments carried the **GET_OK response** (not counting the segments
that carried HELLO_OK and BYE_OK), and what were their payload sizes? Were they
all the same size? Were they all the MSS? Did any one segment carry the
end of one ETP message and the start of the next, or did one ETP message span
several segments? Explain what this means for code that reads an ETP message, and
point to the place in your own code that deals with it._

The GET_OK response was carried across nine TCP segments:
- Frame 53 had the 8-byte header
- Frames 54, 56, 57, 59, 60, 62, 63, and 65 each carried 32,768 bytes of data

They were not all the same size and the 32,768-byte data segments were smaller than the advertised MSS of 65,495 bytes.

No segment contained the end of one ETP message and the start of another, and yes GET_OK spanned 9 segments

This matters because TCP sends data as a continouts stream. An app can't assume that one TCP segment is one complete message.

Our code handles this using etp_read_header() in etp.c, read_response() in client.c, and transport_tcp.c

## Q4. Teardown and TIME-WAIT

_Which side sent the first FIN in the GET session, and why — cite the rule in the
specification. Paste the relevant line(s) from `evidence/time-wait.txt`. Which end
of the connection (address and port) is in TIME-WAIT, why that end, and roughly how
long Linux keeps it there? Connect this to the SO_REUSEADDR requirement._

According to the ETP/1 specification (§8.2), the server must send BYE_OK and then close the connection.

The server sent the first FIN in frame 68 at 0.148602 seconds. The client sent its FIN in frame 69 at 0.148603 seconds. Because the two FINs crossed during shutdown, both sides entered TIME-WAIT.

The connection endpoints are:
- Client: 127.0.0.1:33508
- Server: 127.0.0.1:45678

The evidence/time-wait.txt file shows:
127.0.0.1:33508  127.0.0.1:45678
127.0.0.1:45678  127.0.0.1:33508

Both endpoints appear because the FIN packets crossed during shutdown.

Linux typically keeps connections in TIME-WAIT for about 60 seconds. SO_REUSEADDR allows the server to reuse its listening address and port while an older connection is still in TIME-WAIT. It does not remove TIME-WAIT itself.

## Q5. What an observer learns

_Suppose someone else obtained this capture file. List specifically what they
learn about your session (names, sizes, timing, anything else). You captured with a
200-byte snap length: what does that hide, and what does it not hide? Which kind of
protection would stop a real on-path observer from learning these things, and what
would it still not hide?_

Someone with access to this capture could learn several details about both the PUT and GET sessions:
- Client name: evidence-client
- Filename: evidence_payload.bin
- Client and server IP addresses and ports
- ETP message types
- File size: 262,144 bytes
- Connection timing and duration
- TCP sequence numbers and MSS
- Connection shutdown information
- PUT_OK digest
- Parts of the unencrypted file contents

The GET connection lasted about 51 milliseconds from frame 38 to frame 71.

The capture used a 200-byte snap length, meaning it saved only the first 200 bytes of each packet. This limits how much packet data can be inspected, but it does not limit how much data was actually transferred.

TLS encryption would prevent an observer from reading the ETP headers and file contents, but they could still see the IP addresses, connection timing, and approximate amount of data transferred.