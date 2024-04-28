# Stratum

This document describes a protocol, that allows a group of miners to connect
to a server, which coordinates the distribution of work packages among
miners.

The initial protocol was written for Bitcoin and contains several pieces that
need adjustment in order to be usable with xelis.

This is currently just a proposal and is meant to be as frictionless to implement as possible.


## Specification


### Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to
be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The stratum protocol mostly adheres to the [JSON RPC 2.0](https://www.jsonrpc.org/specification) specification.

### Overview

Communication happens over a bidirectional channel with messages encoded as
JSON with `LF` delimiter—in this document written as `\n`.

Since requests can be handled out of order, each request in a session SHOULD
have an unique `id`. In order to be able to match request and response,
responses MUST have the same `id` as their matching request. Notifications
sent by the server or calls that do not trigger any response MAY have an `id` of
`null`.

For further details on the members of request and response objects consult the
[JSON RPC 2.0 specification](https://www.jsonrpc.org/specification).


### Protocol flow example

The following shows what a session might look like from subscription to
submitting a solution.

```
Client                                Server
  |                                     |
  | --------- mining.subscribe -------> |
  | --------- mining.authorize -------> |
  | <-------- mining.set_difficulty --- |
  |                                     |----
  | <---------- mining.notify --------- |<--/
  |                                     |
  | ---------- mining.submit ---------> |
```

### New difficulty protocol flow example

The following shows what communication may look like when a new difficulty is set for a miner.

```
Client                                Server
  |                                     |
  | <-------- mining.set_difficulty --- |
  |                                     |----
  | <---------- mining.notify --------- |<--/
  |                                     |
  | ---------- mining.submit ---------> |
```

### Methods
- [mining.subscribe](#mining-subscribe)
- [mining.authorize](#mining-authorize)
- [mining.notify](#mining-notify)
- [mining.set_difficulty](#mining-set_difficulty)
- [mining.submit](#mining-submit)
- [mining.ping](#mining-ping)
- [mining.pong](#mining-pong)
- [mining.print](#mining-print)


### Errors

Whenever an RPC call triggers an error, the response MUST include an `error`
field which maps to a **list** of the following values:

- [ `code` : `int` ]
- [ `message` : `string` ]
- [ `data` : `object` ]

```
{"id": 10, "result": null, "error": [21, "Job not found", null]}
```

Errors SHOULD be identified by their `code` and programs SHOULD do error
handling based on the `code` and not the `message`.
Available error codes, in addition to the codes defined in the [JSON RPC 2.0]() specification, are:

- `20` - Other/Unknown
- `21` - Job not found (=stale)
- `22` - Duplicate share
- `23` - Low difficulty share
- `24` - Unauthorized worker

The `message` field SHOULD be a concise description of the error for human
consumption.

Implementors MAY choose to include an optional `data` object with additional
information relevant to the error.

Including `null` value in `error` object is against the JSON RPC spec. Error should only be included in the response when there is an actual error.

### mining.subscribe

In order to initiate or resume a session with the server, a client needs to
call the subscribe method.

This method call will only be executed by clients.


#### Request:

```
{"id": 1, "method": "mining.subscribe", "params": ["MyMiner/1.0.0"]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`string`) ]: list of method
  parameters
  1. MUST be name and version of mining software in the given format or empty
     string

#### Response

```
{ "id": 1, "result": ["ABC123"] }\n
```

- [ `id` : `int` ]: request id
- [ `result` : (`string`, `int`) ]:
    1. This SHOULD be a unique session id

### mining.authorize

Before a client can submit solutions to a server it MUST authorize at least one worker.

This method call will only be executed by clients.

#### Request

```
{"id": 2, "method": "mining.authorize", "params": ["xel:WALLET_ADDRESS", WORKER_NAME", "WORKER_PASSWORD"]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`string`, `string`, `string`) ]: list of method parameters
    1. The miner wallet address
    2. The worker name
    3. The worker password


#### Response

```
{"id": 2, "result": ["EXTRANONCE", 32]}\n
```

- [ `id` : `int` ]: request id
- [ `result` : (`string`, `int`) ]:
      1. Extra nonce for the miner to use
      2. The length of the extra nonce
- [ `error` : (`int`, `string`, `object`) ]
    - If authorization failed then it MUST contain error object with the
      appropriate error id and description


### mining.set_difficulty

The target difficulty for a share can change and a server needs to be able to notify clients of that.

This method call will only be executed by the server.


#### Request

```
{"id": null, "method": "mining.set_difficulty", "params": [1]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`int`) ]:
    1. The target difficulty

Any subsequent jobs started by a client after receiving this update MUST
honor the new target and servers will reject any shares below this difficulty.

This SHOULD be followed by a `mining.notify` call.


#### Response

There is no explicit response for this call.


### mining.notify

The notify call is used to supply a worker with new work packages.

This method call will only be executed by the server.


#### Request

```
{"id": null, "method": "mining.notify", "params": ["d70fd222", "abc123", "def456", true ]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`string`, `int`, `string`, `bool`) ]: list of method parameters
    1. Job ID
    2. Work hash
    3. Public key
    4. A boolean indicating whether the miners job queue should be emptied or not ("clean jobs")


#### Response

There is no explicit response for this call.


### mining.submit

With this method a worker can submit solutions for the mining puzzle.
This method call will only be executed by clients.


#### Request

```
{"id": 4, "method": "mining.submit", "params": ["WORKER_NAME", "d70fd222", "98b6ac44d2", "000000123"]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`string`, `string`, `string`, `string`) ]: list of method
  parameters
    1. Worker name
    2. Job ID
    3. Timestamp
    4. Miner nonce

#### Response

```
{"id": 4, "result": true}\n
```

- [ `id` : `int` ]: request id
- [ `result`: `bool` ]: submission accepted
    - MUST be `true` if accepted
- [ `error` : (`int`, `string`, `object`) ]
    - If submission failed then it MUST contain error object with the
      appropriate error id and description

### mining.ping

With this method a pool can check if a miner connection is still alive.
This method call will only be executed by servers.


#### Request

```
{"id": 4, "method": "mining.ping"}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name

#### Response
No response is required for this call.  The client should respond with a `mining.pong` call.

### mining.pong

With this method a client/miner can respond to signify that the connection is still alive.
This method call will only be executed by clients.


#### Request

```
{"id": 4, "method": "mining.pong"}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name

#### Response
No response is required for this call.

### mining.print

With this method a server can send a message to the miner to print on screen.

#### Request

```
{"id": 4, "method": "mining.print", "params": [0, "Your wallet address is invalid, please check before attempting to reconnect."]}\n
```

- [ `id` : `int` ]: request id
- [ `method` : `string` ]: RPC method name
- [ `params` : (`int`, `string`) ]: list of method
  parameters
  1. Print level
  2. Message to print

#### Print Levels
- `0` - Information
- `1` - Warning
- `2` - Error
- `3` - Debug

#### Response
No response is required for this call.