# PA1: Step-by-Step Work Plan

## Goal

Build a C client and server for ETP/1, a file transfer protocol over TCP. The client connects, sends `HELLO`, runs one operation (`list`, `get`, or `put`), and ends with `BYE`. The server serves files from a configured root directory and must handle multiple client sessions.

TCP is a byte stream: one read is not guaranteed to return one whole ETP message. Your program must assemble headers and payloads using the message length in the header. Follow the course PA1 brief and ETP/1 specification whenever this guide and those documents differ.

## Before coding

- [ ] Read the PA1 brief, Project Handbook, and ETP/1 specification. Mark requirements for message formats, validation order, errors, concurrency, shutdown, and evidence.
- [ ] Review the starter headers: [`transport.h`](project/include/transport.h) defines the transport interface; [`etp.h`](project/include/etp.h) defines protocol constants and helper interfaces.
- [ ] Build the starter once with `make build` and note the compiler output. The starter is incomplete, so build errors or self-check failures are expected at first.
- [ ] Keep a short list of requirements and tests. Complete one layer at a time so failures are easier to trace.

## Step 1: Implement the TCP transport

Work in [`transport_tcp.c`](project/src/transport_tcp.c). Keep socket operations inside this transport file, as required by the interface comments.

- [ ] Implement `tp_listen`: resolve the bind address, create a TCP socket, set `SO_REUSEADDR`, bind, and listen. Clean up resources if a step fails.
- [ ] Implement `tp_accept`: accept a connection and format the peer address and port for logging.
- [ ] Implement `tp_connect`: resolve the host, try suitable addresses, and connect.
- [ ] Implement `tp_send_all`: loop until all bytes are sent or an error occurs. Prevent a disconnected peer from terminating the process with `SIGPIPE`.
- [ ] Implement `tp_recv_exact`: keep receiving until the requested byte count arrives; distinguish clean EOF before any bytes from a partial message and from an error.
- [ ] Implement `tp_close` and `tp_close_listener`: follow the interface's shutdown, drain, timeout, and resource cleanup requirements.
- [ ] Build and test small transport cases: successful loopback connection, peer disconnect, and short/failed send or receive behavior.

**Checkpoint:** both programs can use the transport functions without making socket calls outside `transport_tcp.c`.

## Step 2: Implement ETP framing and helpers

Work in [`etp.c`](project/src/etp.c), checking every field and rule against the specification.

- [ ] Implement `etp_send`: construct the fixed 8-byte header, encode the 32-bit payload length in network byte order, then send the header and payload completely.
- [ ] Implement `etp_read_header`: receive exactly 8 bytes and decode the fields into the provided structure.
- [ ] Check version, reserved fields, message type, and maximum length in the order required by the specification. Reject invalid or oversized lengths before allocating or reading a payload.
- [ ] When handling a message, read exactly its declared payload length after the header has passed validation. Do not assume a single `recv` returns the whole payload.
- [ ] Use the supplied name-validation and SHA-256 helpers where appropriate; verify how their rules fit the protocol.

**Checkpoint:** a test message can be encoded and decoded correctly, including when its header or payload arrives in several TCP reads.

## Step 3: Build server session handling

Work in [`server.c`](project/src/server.c). Treat each accepted connection as an independent ETP session.

- [ ] Add the required concurrency model (threads, processes, or an event loop) so a stalled client does not block other clients. Handle session failures locally so they do not stop the listening server.
- [ ] For each session, read and validate `HELLO`, check its payload/name according to the spec, and send the appropriate response.
- [ ] After a successful greeting, process requests until `BYE`, a fatal protocol error, or a disconnect. Send exactly the required response for each request and log responses in the required format.
- [ ] Implement `LIST` by enumerating the configured root according to the specification.
- [ ] Implement `GET` by validating the requested filename, opening only files inside the configured root, enforcing size limits, and sending the specified payload and digest fields.
- [ ] Implement `PUT` by validating the filename and request size, receiving the complete payload, checking its digest, and writing the file according to the specification.
- [ ] Apply the specified error behavior: distinguish errors that end a session from errors that allow it to continue.
- [ ] Consider simultaneous sessions accessing the same file and protect shared state as needed.
- [ ] Ensure disconnects during a large transfer and failed writes do not crash or terminate the server.

**Checkpoint:** the server can complete a session and continue accepting clients after that session ends or fails.

## Step 4: Build client commands

Work in [`client.c`](project/src/client.c). Keep diagnostics on standard error and follow the required exit statuses.

- [ ] Connect to the requested host and port, send `HELLO` with the client name, and verify the server's response.
- [ ] Implement `list`: send the request, receive the response payload completely, and print it as specified.
- [ ] Implement `get`: send the remote name, receive and validate the response, write the local file, and check any digest required by the protocol.
- [ ] Implement `put`: read the local file, enforce protocol limits, calculate the required digest, send the request and data, and handle the response.
- [ ] Send `BYE` after the operation when the protocol permits, verify the response, and close the connection cleanly.
- [ ] Map server errors, local failures, and digest failures to the required exit codes.

**Checkpoint:** each command works against your own server, and errors produce useful diagnostics without corrupting output intended for the user.

## Step 5: Test incrementally

- [ ] Rebuild after each layer: `make build`.
- [ ] Start the server in one terminal with a temporary test root, then run the client from another terminal. Try `list`, `put`, and `get`; compare transferred files byte-for-byte.
- [ ] Run the course self-check and save its complete output. Run it from the repository as documented by the course materials.
- [ ] Exercise boundary and failure cases: empty file, maximum permitted size, too-large payload, invalid filename, missing file, invalid header fields, truncated header/payload, and disconnect during transfer.
- [ ] Check concurrent clients by running more than one client, including one that stalls or disconnects.
- [ ] Fix failures by tracing them to a requirement or code path; rerun the relevant case and the full self-check after changes.

## Step 6: Capture and analyze your own traffic

The repository's [`evidence/ANALYSIS.md`](project/evidence/ANALYSIS.md) lists the required capture facts and questions.

- [ ] Capture your own successful `PUT` and `GET` sessions using the capture instructions from the course materials.
- [ ] Save the required packet capture and `TIME-WAIT` evidence in the specified evidence directory.
- [ ] Use Wireshark or `tshark` to collect the required values from your capture. Use raw TCP sequence numbers if the worksheet requests them.
- [ ] Complete the analysis using measurements from your own capture. Explain what your code does with stream framing and connect the packet behavior to the implementation.

## Step 7: Complete the documentation and submit

- [ ] Complete [`DESIGN.md`](project/DESIGN.md) based on the implementation you actually wrote: architecture, transport seam, framing, concurrency, errors, threat model, and testing.
- [ ] Complete [`AI_USE.md`](project/AI_USE.md) accurately. Log substantive AI help and identify any AI-produced material that remains, as required by the course policy.
- [ ] Update [`README.md`](project/README.md) with build/run instructions, dependencies, self-check results, known limitations, and the required authorship statement.
- [ ] Review all required source and evidence files, confirm the repository state and commit, and follow the submission steps in the PA1 brief.

## Useful mental model

```text
client/server session logic
          ↓ ETP messages
ETP header and payload framing
          ↓ exact byte operations
TCP transport interface
          ↓
       TCP sockets
```

Keep the layers separate: session logic decides *which* ETP message to exchange, framing turns messages into bytes, and the transport moves exact byte counts.
