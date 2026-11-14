# PA1 Design — ETP/1 Client and Server

_One to two pages. Short, specific answers beat long general ones. Every section
should be true of the code you submit. Delete these italic prompts when you write._

## 1. Architecture

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

## 2. The transport seam

transport_tcp.c is the transport file, containing six tp functions covering
operations listen, accept, connect, send all, recv exact, and close. No other
file calls socket API which was confirmed with a grep over src/ and include/
for three socket headers and socket calls in section 11, excluding transport.
Command ran:
```
grep -nE '#include *<(sys/socket|netinet/in|netdb)\.h>|\b(socket|bind|listen|accept|connect|send|sendto|sendmsg|recv|recvfrom|recvmsg|setsockopt|shutdown|getaddrinfo)\s*\(' src/*.c include/*.h | grep -v '/transport'
```


When swapping TCP over for the PA2 transport, transport_tcp.c would be replaced with a new transport module, using the same six tp functions with the PA2 ETP-T protocol, and an updated Makefile. The files server.c, client.c, and etp.c would stay the same since they interact with the transport through the same interface. The ETP-T would need to handle reliability and distinguish individual client connections to preserve the server's concurrency design.


## 3. Framing

The code reads each ETP message by using an exact-read function first to collect the 8 bytes of the header and handles cases where the header may arrive in several pieces. It checks the VERSION and LENGTH fields before allocating memory or reading the payload, preventing invalid versions or excessively large messages from causing issues. Once the header is validated it reads LENGTH bytes exactly for the payload, with any additional messages arriving in the same read, being available for the next message. The header is built using htonl and memcpy to ensure consistent byte order.


## 4. Concurrency

The server uses a detached pthread per client connection, allowing each session to use simple blocking code without preventing other clients from being served if one stalls halfway through a header. The client's peer information is copied into a session_arg structure because the original buffer in main is reused for subsequent connections. The only state that is shared is stdout, which is protected by g_log_lock (PUT operations also use unique temporary files with mkstemp and rename them into place so concurrent GET requests never receive partially written files). However, this means that every connected client (including idle ones) consumes a thread. 


## 5. Errors and disconnects

The code distinguishes fatal and non-fatal errors: fatal errors, such as BAD_VERSION, TOO_LARGE, and NOT_READY, send an ERROR response before closing the connection with break, while non-fatal errors send an ERROR response and allow the session to continue. If a client disconnects halfway through a large GET, the server stops sending, frees any allocated resources, and closes the connection without attempting another response. To prevent a broken connection from terminating the server, I ignore SIGPIPE and use MSG_NOSIGNAL; the client determines its exit code based on whether the operation completed successfully or encountered an error.


## 6. Threat model

Anyone who can reach the server over the network could attempt to exploit it, including by sending malicious requests, exhausting memory, or overwriting files. The server resists path traversal attacks because etp_valid_name rejects filenames containing / or beginning with ., and it checks files with lstat and S_ISREG to prevent symlink escapes. It also validates LENGTH before allocating memory to prevent oversized payloads from exhausting memory. However, it does not protect against eavesdropping or tampering because there is no encryption or authentication, and anyone with access to the server can potentially overwrite existing files using PUT.


## 7. Testing

.bin files were built and sent with nc to test and verify how the server handled different messages. I checked the responses by each byte using xxd against the specification in section 12 of the ETP doc, verified file integrity using SHA-256 hashes, and tested invalid message types with a bad.bin file. One bug that came up was etp_read_header returned -1 even when it succeeded, causing the server to quit during the first header read without responding or printing a debug message. I diagnosed this by confirming the server was still running with pgrep, traced the issue to the incorrect return value, and fixed it by changing the success return to 1.