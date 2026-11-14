# PA1 Capture Analysis

_Your capture, `evidence/pa1-capture.pcap`, holds two ETP sessions from your own
programs: a PUT, then a GET of the same file. This analysis is about the **GET
session** unless a question says otherwise. Use Wireshark (or tshark) to read the
capture. The numbers must come from your capture: they are checked against it._

## How I captured it

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

## Q2. Your protocol on the wire

_Find the segment that carries your client's GET request. Give its frame number,
paste the first 8 payload bytes in hex, and decode each header field (VERSION,
TYPE, STATUS, RESERVED, LENGTH). Do the same for the header of the server's
GET_OK. Does GET_OK's LENGTH equal the size of your file? Show the arithmetic that
connects `server_to_client_payload_bytes` to the three responses the server sent in
this session._

## Q3. A byte stream, not messages

_How many segments carried the **GET_OK response** (not counting the segments
that carried HELLO_OK and BYE_OK), and what were their payload sizes? Were they
all the same size? Were they all the MSS? Did any one segment carry the
end of one ETP message and the start of the next, or did one ETP message span
several segments? Explain what this means for code that reads an ETP message, and
point to the place in your own code that deals with it._

## Q4. Teardown and TIME-WAIT

_Which side sent the first FIN in the GET session, and why — cite the rule in the
specification. Paste the relevant line(s) from `evidence/time-wait.txt`. Which end
of the connection (address and port) is in TIME-WAIT, why that end, and roughly how
long Linux keeps it there? Connect this to the SO_REUSEADDR requirement._

## Q5. What an observer learns

_Suppose someone else obtained this capture file. List specifically what they
learn about your session (names, sizes, timing, anything else). You captured with a
200-byte snap length: what does that hide, and what does it not hide? Which kind of
protection would stop a real on-path observer from learning these things, and what
would it still not hide?_
