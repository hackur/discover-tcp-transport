# discover-tcp-transport

_Stability: 1 - [Experimental](https://github.com/tristanls/stability-index#stability-1---experimental)_

TCP transport for [Discover](https://github.com/tristanls/node-discover), a distributed master-less node discovery mechanism that enables locating any entity (server, worker, drone, actor) based on node id.

## Installation

    npm install discover-tcp-transport

## Tests

    npm test

## Overview

Discover TCP transport implements the Discover transport protocol consisting of a stripped down version of the Kademlia DHT protocol, PING and FIND-NODE. The TCP transport reports a node as unreachable if a connection cannot be established with it. 

_NOTE: Unreachability of nodes depends on the transport. For example, other transports (like TLS transport) could use other criteria (like invalid certificate) for reporting unreachable nodes._

_**WARNING**: Using TCP transport is meant primarily for development in a development environment. TCP transport exists because it is a low hanging fruit. It is most likely that it should be replaced with DTLS transport in production (maybe TLS if DTLS is not viable). There may also be a use-case for using UDP transport if communicating nodes are on a VPN/VPC. Only if UDP on a VPN/VPC seems not viable, should TCP transport be considered._

## Documentation

### TcpTransport

**Public API**
  * [TcpTransport.listen(options, callback)](#tcptransportlistenoptions-callback)
  * [new TcpTransport(options)](#new-tcptransportoptions)
  * [tcpTransport.close(callback)](#tcptransportclosecallback)
  * [tcpTransport.findNode(contact, nodeId, sender)](#tcptransportfindnodecontact-nodeid-sender)
  * [tcpTransport.listen(callback)](#tcptransportlistencallback)
  * [tcpTransport.ping(contact)](#tcptransportpingcontact)
  * [Event 'findNode'](#event-findnode)
  * [Event 'node'](#event-node)
  * [Event 'reached'](#event-reached)
  * [Event 'unreachable'](#event-unreachable)

#### TcpTransport.listen(options, callback)

  * `options`: See `new TcpTransport(options)` `options`.
  * `callback`: See `tcpTransport.listen(callback)` `callback`.
  * Return: _Object_ An instance of TcpTransport with server running.

Creates new TCP transport and starts the server.

#### new TcpTransport(options)

  * `options`:
    * `host`: _String_ _(Default: 'localhost')_
    * `port`: _Integer_ _(Default: 6742)_ A port value of zero will assign a random port.

Creates a new TCP transport.

#### tcpTransport.close(callback)

  * `callback`: _Function_ _(Default: undefined)_ Optional callback to call once the server is stopped.

Stops the server from listening to requests from other nodes.

#### tcpTransport.findNode(contact, nodeId, sender)

  * `contact`: _Object_ The node to contact with request to find `nodeId`.
    * `id`: _String (base64)_ Base64 encoded contact node id.
    * `host`: _String_ IP address to connect to.
    * `port`: _Integer_ Port to connect to.
  * `nodeId`: _String (base64)_ Base64 encoded string representation of the node id to find.
  * `sender`: _Object_ The node making the request
    * `id`: _String (base64)_ Base64 encoded sender node id.
    * `data`: _Any_ Sender node data.
    * `host`: _String_ Host to connect to.
    * `port`: _Integer_ Port to connect to.

Issues a FIND-NODE request to the `contact`. In other words, sends FIND-NODE request to the contact at `contact.host` and `contact.port` using TCP. The transport will emit `node` event when a response is processed (or times out).

#### tcpTransport.listen(callback)

  * `callback`: _Function_ _(Default: undefined)_ Optional callback to call once the server is up.

Starts the server to listen to requests from other nodes.

#### tcpTransport.ping(contact)

  * `contact`: _Object_ Contact to ping.
    * `id`: _String (base64)_ Base64 encoded contact node id.
    * `host`: _String_ Host to connect to.
    * `port`: _Integer_ Port to connect to.

Issues a PING request to the `contact`. In other words, pings the contact at the `contact.host` and `contact.port` using TCP. The transport will emit `unreachable` event if the contact is deemed to be unreachable, or `reached` event otherwise.

#### Event: `findNode`

  * `nodeId`: _String (base64)_ Base64 encoded string representation of the node id to find.
  * `sender`: _Object_ The contact making the request.
    * `id`: _String (base64)_ Base64 encoded sender node id.
    * `data`: _Any_ Sender node data.
    * `host`: _String_ Host of the sender.
    * `port`: _Integer_ Port of the sender.
  * `callback`: _Function_ The callback to call with the result of processing the FIND-NODE request.
    * `error`: _Error_ An error, if any.
    * `response`: _Object_ or _Array_ The response to FIND-NODE request.

Emitted when another node issues a FIND-NODE request to this node.

```javascript
var tcpTransport = require('discover-tcp-transport');
tcpTransport.on('findNode', function (nodeId, sender, callback) {
    // ... find closestNodes to the desired nodeId
    return callback(null, closestNodes);
});
```

In the above example `closestNodes` is an Array of contacts that are closest known to the desired `nodeId`.

If the node handling the request itself contains the `nodeId`, then it sends only itself back.

```javascript
var tcpTransport = require('discover-tcp-transport');
tcpTransport.on('findNode', function (nodeId, sender, callback) {
    // ... this node knows the node with nodeId or is itself node with nodeId
    return callback(null, nodeWithNodeId); 
});
```

In the above example, `nodeWithNodeId` is not an array, but an individual `contact` representing the answer to `findNode` query.

#### Event: `node`

  * `error`: _Error_ An error, if one occurred.
  * `contact`: _Object_ The node that FIND-NODE request was sent to.
  * `nodeId`: _String (base64)_ The original base64 encoded node id requested to be found.
  * `response`: _Object_ or _Array_ The response from the queried `contact`.

If `error` occurs, the transport encountered an error when issuing the `findNode` request to the `contact`. `contact` and `nodeId` will also be provided in case of an error. `response` is undefined if an `error` occurs.

`response` will be an Array if the `contact` does not contain the `nodeId` requested. In this case `response` will be a `contact` list of nodes closer to the `nodeId` that the queried node is aware of. The usual step is to next query the returned contacts with the FIND-NODE request.

`response` will be an Object if the `contact` contains the `nodeId`. In other words, the node has been found.

#### Event: `reached`

  * `contact`: _Object_ The contact that was reached when pinged.
    * `id`: _String (base64)_ Base64 encoded contact node id.
    * `data`: _Any_ Data included with the contact.
    * `host`: _String_ Host of reached contact.
    * `port`: _Integer_ port of reached contact.

Emitted when a previously pinged `contact` is deemed reachable by the transport.

#### Event: `unreachable`

  * `contact`: _Object_ The contact that was unreachable when pinged.
    * `id`: _String (base64)_ Base64 encoded contact node id.
    * `host`: _String_ Host of unreachable contact.
    * `port`: _Integer_ port of unreachable contact.

Emitted when a previously pinged `contact` is deemed unreachable by the transport.

## Wire Protocol

Wire protocol for TCP transport is simple one-line \r\n terminated ASCII.

### FIND-NODE

    {"request":{"findNode":"Zm9v"},"sender":{"id":"YmF6","data":"some data","host":"127.0.0.1","port":6742}}\r\n

FIND-NODE request consists of base64 encoded node id followed by \r\n.

#### Object Response

    {"id":"Zm9v","data":"some data","host":"127.0.0.1","port":6742}\r\n

An Object response is JSON representation of the contact followed by \r\n.

#### Array Response

    [{"id":"YmFy","data":"some data","host":"192.168.0.1","port":6742},{"id":"YmF6","data":"some data","host":"192.168.0.2","port":6742}]\r\n

An Array response is JSON representation of an array of closest contacts followed by \r\n.

### PING

Ping tests connectivity only and transfers no data.