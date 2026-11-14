# AI Use Disclosure — PA1

_Generative AI is allowed on PA1 as a tutor, reviewer, debugging partner and
helper with tooling. It is not allowed to be the author of your work. See the PA1
brief, "AI, Authorship and You". Fill in every section. "None" is an acceptable
answer only if it is true._

## 1. Did you use generative AI on this assignment?

- [ ] No. I did not use any generative AI tool for any part of PA1.
- [ X ] Yes, as described below.

## 2. Statement (the syllabus form)

> The author acknowledges the utilization of ChatGPT, a language model
> developed by OpenAI, in the preparation of this assignment. ChatGPT
> was employed in the following manner(s) within this assignment: 
> Adam: for providing summaries of provided document and explaning errors/why testing didn't work
> Bryler: for explaining code and assisting with coding logic

## 3. Log

_One row per substantive interaction. Summaries are fine; you do not need to paste
whole conversations, but keep them in case you are asked._

| # | What I asked (summary) | What I used from the answer | What I changed or rejected | How I checked it was right |
|---|---|---|---|---|
| 1 | wait are there two different types of sockets? one for receiving and one for sending? | the information | n/a | intuition |
| 2 | what does tv_sec stand for | clarification | nothing | intuition |
| 3 | so when do you have to use "struct" when defining a variable of a struct type | clarification | nothing | didn't |
| 4 | how does recv() work | code logic | code logic | didn't |
| 5 | what is MSG_NOSIGNAL | code logic | code logic | didn't |
| 6 | so if we fail to send the bytes and we received a SIGPIPE signal, the process would automatically try to terminate? | clarification | nothing | didn't |
| 7 | why is extern void freeaddrinfo (struct addrinfo *__ai) __THROW; a thing? instead of just using typical free methods | clarification | nothing | didn't |
| 8 | so is addrinfo a linkedlist or what? | clarification | nothing | didn't |
| 9 | how does getaddrinfo work | clarification | nothing | didn't |
| 10 | what is restrict in parameter placeholders for functions | clarification | nothing | didn't |
| 11 | what is AF_UNSPEC | clarification | nothing | didn't |
| 12 | what does ai in ai_family stand for? | clarification | nothing | didn't |
| 13 | for memset(&address_query_hints, 0, sizeof(address_query_hints)); couldn't we just initialize it with = {0}? | the information | the initialization of the variable | didn't |
| 14 | how does accept() work? | the information | coding accept() | didn't |
| 15 | explain ntohs() | the information | coding | didn't |
| 16 | tp_accept vs tp_listen when would I call each of them and on the server or the client or what | the information | understanding | didn't |
| 17 | accept blocks until what? until it receives confirmation from the other client that the connection is accepted? | the information | coding accept() | didn't |
| 18 | if the socket is marked as listen() then it is blocking, otherwise if it is marked as bind() it is blocking. is that right? | the information | coding | didn't |
| 19 | why shouldn't I malloc here struct sockaddr_in bind_address = {0}; but I do here: tp_listener *listener = malloc(sizeof(tp_listener)); | the information | coding | didn't |

## 4. Where AI-produced material remains

_List each file and function (or document section) that still contains code or
text an AI tool generated, even after you edited it. If none, say none._

etp.c, client.c, and transport_tcp.c

## 5. One thing the AI got wrong — or how you checked it

_Describe one answer, suggestion or piece of code that was wrong, misleading or
did not fit the specification, and how you caught it. If you found nothing wrong,
say so and describe how you checked (a test you ran, the part of the
specification you compared it with). If you did not use AI, describe one thing
you got wrong yourself and how you caught it._

I asked about the receive timeout in tp_close and it said that the one-second timeout prevents the drain loop from waiting indefinitely, but it really limits each individual recv() wait, not the total time spent in the loop. The loop can still continue indefinitely if the peer kept sending data often enough.
