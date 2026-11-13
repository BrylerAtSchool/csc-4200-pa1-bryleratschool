# PA1 Design — ETP/1 Client and Server

_One to two pages. Short, specific answers beat long general ones. Every section
should be true of the code you submit. Delete these italic prompts when you write._

## 1. Architecture

_Your modules and what calls what. A small diagram (ASCII is fine) or a list:
which file holds the protocol (encode/decode), which holds the transport, which
holds the server's session logic, which holds the client._

 ```
          client                                  server
+---------------------------+    +----------------------------------------+
| client.c                  |    | server.c                               |
|  run(): HELLO -> command  |    |  main(): args, ignore SIGPIPE, listen  |
|    -> BYE, exit code 0-3  |    |    accept loop                         |
|                           |    |      | one detached thread per client  |
|                           |    |      v                                 |
|                           |    |  handle_session(): read header,        |
|                           |    |    ETP 8.1 checks, HELLO / BYE,        |
|                           |    |    handle_list / get / put <-> --root  |
|                           |    |    log_response -> stdout (locked)     |
+-------------+-------------+    +-------------------+--------------------+
              |                                      |
              v        both programs                 v
     +------------------------------------------------------------+
     | etp.c  (framing)                                           |
     |   etp_send, etp_read_header, etp_valid_name, etp_sha256    |
     +------------------------------------------------------------+
     | transport_tcp.c  (the seam)                                |
     |   tp_listen, tp_accept, tp_connect,                        |
     |   tp_send_all, tp_recv_exact, tp_close                     |
     +------------------------------+-----------------------------+
                                    |
                             TCP byte stream
```
ASCII Diagram

## 2. The transport seam

_Which file is your transport? List the six operations as you implemented them.
Confirm that no other file calls the socket API. In two or three sentences: what
would you have to change, and what would stay the same, when you swap TCP for your
PA2 transport in the final project?_


transport_tcp.c is the transport file, containing six tp functions covering
operations listen, accept, connect, send all, recv exact, and close. No other
file calls socket API which was confirmed with a grep over src/ and include/
for three socket headers and socket calls in section 11, excluding transport.
Command ran:
```
grep -nE '#include *<(sys/socket|netinet/in|netdb)\.h>|\b(socket|bind|listen|accept|connect|send|sendto|sendmsg|recv|recvfrom|recvmsg|setsockopt|shutdown|getaddrinfo)\s*\(' src/*.c include/*.h | grep -v '/transport'
```


When swapping TCP over for the PA2 transport, transport.c would be replaced with a new transport module, using the same six tp functions with the PA2 ETP-T protocol, and an updated Makefile. The files server.c, client.c, and etp.c would stay the same since they interact with the transport through the same interface. The ETP-T would need to handle reliability and distinguish individual client connections to preserve the server's concurrency design.


## 3. Framing

_How does your code read one ETP message from a byte stream? Where and how do you
handle a header that arrives in pieces, or two messages that arrive in one read?
Where do you check VERSION and LENGTH, and why before reading the payload?_


The code reads each ETP message by using an exact-read function first to collect the 8 bytes of the header and handles cases where the header may arrive in several pieces. It checks the VERSION and LENGTH fields before allocating memory or reading the payload, preventing invalid versions or excessively large messages from causing issues. Once the header is validated it reads LENGTH bytes exactly for the payload, with any additional messages arriving in the same read, being available for the next message. The header is built using htonl and mecpy to ensure consistent byte order.


## 4. Concurrency

_Threads, processes or an event loop? What happens to everyone else when one
client stalls half-way through a header? Is any state shared between sessions
(for example, two clients PUT and GET the same file at once), and how is it
protected?_


The server uses a detached pthread per client connection, allowing each session to use simple blocking code without preventing other clients from being server if one stalls halway through a header. The client's peer information is copied into a session_arg structure because the original buffer in main is reused for subsequesnt connections. The only state that is shared is stdout, which is protected by g_log_lock (PUT operations also use unique temporary files with mkstemp and rename them into place so concurrent GET requests never receive partially written files). However, this means that every connected client (including idle ones) consumes a thread. 


## 5. Errors and disconnects

_How do fatal and non-fatal errors differ in your code? What happens when a client
disconnects in the middle of a GET of a large file? (C: how did you deal with
SIGPIPE?) How does your client choose its exit code?_

## 6. Threat model

_One paragraph. Who could attack an ETP/1 server, and how? Name one attack your
server resists (and how), and one it does not. Chapter 5 §5.9 and the
specification's Security Considerations are good starting points._

## 7. Testing

_What did you test beyond `etp_selfcheck.py`? How? Name one bug your testing
found, how you found it, and the fix._
