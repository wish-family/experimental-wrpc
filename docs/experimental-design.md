# Experimental WRPC

## What is WRPC
WRPC (Wish RPC) is a modern remote procedure call framework based on C++20. The primary goal of WRPC is modernization, which means that a series of infrastructures will be built to enable users to not only easily build their own rpc services, but also to flexibly organize their code logic by virtue of the abstractions constructed in WRPC.

## Protocol
WRPC uses WebSocket to communicate due to :
- WebSocket provides message flow on top of TCP, whereas using a bare TCP connection usually requires repeating similar concepts of construction.
- WebSocket supports full-duplex communication, whereas the HTTP/1.x server can only passively answer client requests.
- WebSocket is a very lightweight protocol, with packet headers of only a few bytes after the handshake, and provides extensions related to data compression (permessage-deflate). In contrast, HTTP/2 (or HTTP/3), while a full-duplex and multiplexing-enabled protocol, supports many features that are of little significance for RPC, such as URIs and message headers that are carried in almost every message.
- WebSocket is the de facto standard for full-duplex communication on the Web, and HTTP/2 (or HTTP/3), while supporting server-sent event (SSE), does not have a peer-to-peer status between server and client.
- Boost.Beast provides very good support for WebSocket, and there is no HTTP/2 C++ library that seems modern to me.

In order to handle different remote calls asynchronously on the same connection, we need to add a multiplexing layer on top of WebSocket.

- There was a draft RFC (draft-ietf-hybi-websocket-multiplexing-11) that added multiplexing to WebSocket, but it was built underneath the WebSocket layer. This means that if a new channel needs to be built, a new handshake has to be performed, and this causes the entire existing WebSocket framework to fail, which is obviously not what we want. We would like to do multiplexing after the WebSocket handshake, and we would use an additional number (possibly using varint encoding) to indicate the channel where the current data frame is located (a new channel is created for each remote call).

- On the other hand, we want to support streaming RPC, which means we need to identify on the multiplexing layer whether this frame is the last frame of this channel, which also facilitates recycling resources. We will also add some additional marker bits to store some redundant information, such as frame type (request or response). We also consider adding flow control and data flow priority to control data transmission more finely.

## Coding

We use the encoding specified by protocol buffers, but not their interface description language, i.e., the protocol buffers (version 2 or 3) language. Instead, we use template types to describe data structures, which is how we can better combine the different abstractions.

## Framework

We will build the entire framework on top of Boost.Asio, which provides some very good abstractions that allow asynchronous IO logic to be very refined. Asio provides different ways of doing asynchronous IO, we will use only one of them, based on Coroutines TS (which has been merged into C++20).

Coroutines TS provides very efficient stackless coroutines that compile suspend and resume operations in coroutines into a state machine, which allows switching between function stacks in a matter of nanoseconds. Coroutines TS also provides very flexible customization points, making the use of coroutines not only limited to IO operations.

However, the infrastructure provided by Asio is not sufficient to allow us to build the RPC framework directly, and we need to write some new infrastructure ourselves.
- Asio provides `awaitable` for use with Coroutines TS, which allows us to define continuations using `co_await`, but this is not enough. For streaming RPC, we need `async_generator` to combine `co_await` and `co_yield` in the same coroutine to switch control flow flexibly.
- Asio does not provide a mechanism for communication between coroutines, we need `channel` and `select` to synchronize the state of different coroutines. Richard Hodges designed an experimental library after hearing about this problem, we may need to investigate it.
- Although Asio uses strands to ensure thread safety, we may still need mutexes to synchronize execution state. Under coroutines, we need `async_mutex` to do this with `co_await`, and similarly, we may need `async_conditional_variable`.
