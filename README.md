Socket.IO protocol
==================

The present document describes the protocol and encoding a client/server pair
must follow to establish a successful Socket.IO connection.

## Protocol versions

The present document describes the latest version of the protocol, **1**.

Versions are a single integer incremented with each revision of the protocol.

## Overview

Socket.IO aims to bring a WebSocket-like API to many browsers and devices, with
some specific features to help with the creation of real-world realtime
applications and games.

- Multiple transport support (old user agents, mobile browsers, etc).
- Multiple sockets under the same connection (namespaces).
- Disconnection detection through heartbeats.
- Optional acknowledgments.
- Reconnection support with buffering (ideal for mobile devices or bad networks)
- Lightweight protocol that sits on top of HTTP.

### Anatomy of a Socket.IO socket

A Socket.IO client first decides on a transport to utilize to connect. 

The state of the Socket.IO socket can be `disconnected`, `disconnecting`,
`connected` and `connecting`.

The transport connection can be `closed`, `closing`, `open`, and `opening`.

A simple HTTP handshake takes place at the beginning of a Socket.IO connection.
The handshake, if successful, results in the client receiving:

- A session id that will be given for the transport to open connections.
- A number of seconds within which a heartbeat is expected (`heartbeat
timeout`)
- A number of seconds after the transport connection is closed when the socket
is considered disconnected if the transport connection is not reopened (`close
timeout`).

At this point the socket is considered connected, and the transport is signaled
to open the connection.

If the transport connection is closed, both ends are to buffer messages and
then frame them appropriately for them to be sent as a batch when the
connection resumes.

If the connection is not resumed within the negotiated timeout the socket is
considered disconnected. At this point the client might decide to reconnect the
socket, which implies a new handshake.

## Socket.IO HTTP requests

Socket.IO HTTP URIs take the form of:

    [scheme] '://' [host] '/' [namespace] '/' [protocol version] '/' [transport id] '/' [session id] '/' ( '?' [query] )

Only the methods `GET` and `POST` are utilized (for the sake of compatibility
with old user agents), and their usage varies according to each transport.

The main transport connection is always a `GET` request.

### URI scheme

The URI scheme is decided based on whether the client requires a secure connection
or not. Defaults to `http`, but `https` is the recommended one.

### URI host

The host where the Socket.IO server is located. In the browser environment, it
defaults to the host that runs the page where the client is loaded (`location.host`)

### Namespace

The connecting client has to provide the namespace where the Socket.IO requests
are intercepted.

This defaults to `socket.io` for all client and server distributions.

### Protocol version

Each client should ship with the revision ID it supports, available as a public
interface to developers.

For example, the browser client supports `io.protocolVersion`.

### Transport ID

The following transports are supported:

- `xhr-polling`
- `xhr-multipart`
- `htmlfile`
- `websocket`
- `flashsocket`
- `jsonp-polling`

The client first figures out what transport to use. Usually this occurs in the
browser, utilizing [feature detection](http://mzl.la/6tm7Aj).

User-defined transports are allowed.

### Query

The query component (eg: `?token=48737481747&`) is not present on all URLs. 
Certain query keys are reserved by Socket.IO:

* `t`: Contains a timestamp, only used to bypass caching on certain old UAs.
* `disconnect`: Triggers a disconnection.

User-defined query components are allowed. For example,
`?t=1238141910&token=mytoken` is a valid query).

## Handshake

The client will perform an initial HTTP POST request like the following

    http://example.com/socket.io/1/

The absence of the `transport id` and `session id` segments will signal the server
this is a new, non-handshaken connection.

The server can respond in three different ways:

  - 401 Unauthorized

  If the server refuses to authorize the client to connect, based on the supplied
  information (eg: `Cookie` header or custom query components).

  No response body is required.

  - 503 Service Unavailable

  If the server refuses the connection for any reason (eg: overload).

  No response body is required.

  - 200 OK

  The handshake was successful.

  The body of the response should contain the session id (`sid`) given to the
  client, followed by the heartbeat timeout, the connection closing timeout,
  and the list of supported transports separated by `:`

  The absence of a heartbeat timeout ('') is interpreted as the server and
  client not expecting heartbeats.

  For example `4d4f185e96a7b:15:10:websocket,xhr-polling`.

## Transport connection

Once the handshake request-response cycle is complete (and it ended with success),
a new connection is opened by the transport that was negotiated, with a `GET`
HTTP request.

The transport *can* modify the URI if the transport requires it, as long as no
information is lost. For example, if `websocket` is accepted as the transport,
and the connection was secure, the URI for the transport connection will become:

    wss://example.com/socket.io/1/websocket/4d4f185e96a7b

The URI still contains all the information required by Socket.IO to continue the
message exchange (protocol security, namespace, protocol version, transport, etc).

Messages can be sent and received by following this convention. **How** the messages
are encoded and framed depends on each transport, but generally boils down to
whether the transport has built-in framing (unidirectionally and/or bidirectionally).

### Unidirectional transports

Transports that initialize unidirectional connections (where the server can
write to the client but not vice-versa), should perform `POST` requests to send
data back to the server to the same endpoint URI.

## Messages

### Framing

Certain transports, like `websocket` or `flashsocket`, have built-in lightweight
framing mechanisms for sending and receiving messages.

For `xhr-multipart`, the built-in MIME framing is used for the sake of consistency.

When no built-in lightweight framing is available, and multiple messages need to be
delivered (i.e: buffered messages), the following is used:

    `\ufffd` [message lenth] `\ufffd`

Transports where the framing overhead is expensive (ie: when the xhr-polling
transport tries to send data to the server).

### Encoding

Messages have to be encoded before they're sent. The structure of a message is
as follows:

    [message type] ':' [message id ('+')] ':' [message endpoint] (':' [message data]) 

The message type is a single digit integer.

The message id is an incremental integer, required for ACKs (can be omitted).
If the message id is followed by a `+`, the ACK is not handled by socket.io,
but by the user instead.

Socket.IO has built-in support for multiple channels of communication (which we
call "multiple sockets"). Each socket is identified by an endpoint (can be
omitted).

### (`0`) Disconnect

Signals disconnection. If no endpoint is specified, disconnects the entire
socket.

Examples:

- Disconnect a socket connected to the `/test` endpoint.

      0::/test

- Disconnect the whole socket

      0

### (`1`) Connect

Only used for multiple sockets. Signals a connection to the endpoint.
Once the server receives it, it's echoed back to the client.

Example, if the client is trying to connect to the endpoint /test, a message
like this will be delivered:

    '1::' [path] [query]

Example:

    1::/test?my=param

To acknowledge the connection, the server echoes back the message. Otherwise,
the server might want to respond with a error packet.

### (`2`) Heartbeat

Sends a heartbeat. Heartbeats must be sent within the interval negotiated with
the server. It's up to the client to decide the padding (for example, if the
heartbeat timeout negotiated with the server is 20s, the client might want to
send a heartbeat evert 15s).

### (`3`) Message

    '3:' [message id ('+')] ':' [message endpoint] ':' [data]

A regular message.

    3:1::blabla

### (`4`) JSON Message

    '4:' [message id ('+')] ':' [message endpoint] ':' [json]

A JSON encoded message.

    4:1::{"a":"b"}

### (`5`) Event

    '5:' [message id ('+')] ':' [message endpoint] ':' [json encoded event]

An event is like a json message, but has mandatory `name` and `args` fields.
`name` is a string and `args` an array.

The event names

    'message'
    'connect'
    'disconnect'
    'open'
    'close'
    'error'
    'retry'
    'reconnect'

are reserved, and cannot be used by clients or servers with this message type.

### (`6`) ACK

    '6:::' [message id] '+' [data]

An acknowledgment contains the message id as the message data. If a `+` sign
follows the message id, it's treated as an event message packet.

Example 1: simple acknowledgement

    6:::4

Example 2: complex acknowledgement

    6:::4+["A","B"]

### (`7`) Error

    '7::' [endpoint] ':' [reason] '+' [advice]

For example, if a connection to a sub-socket is unauthorized.

### (`8`) Noop

No operation. Used for example to close a poll after the polling duration times
out.

## Forced socket disconnection

A Socket.IO server must provide an endpoint to force the disconnection of the
socket.

While closing the transport connection is enough to trigger a disconnection, it
sometimes is desirable to make sure no timeouts are activated and the
disconnection events fire immediately.

    http://example.com/socket.io/1/xhr-polling/812738127387123?disconnect

The server must respond with `200 OK`, or `500` if a problem is detected.
