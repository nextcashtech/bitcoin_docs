# Messages Service

The messages service provides HTTP URLs for a client to receive messages. Since most users won't have domain names or want to host web services this is required for peer to peer and two-way client to service communication. The messages should be signed and encrypted whenever possible so little to no trust is required for the messaging service. The service must simply stay up and deliver messages to the appropriate recipients.

A client authenticates with the service using a Bitcoin (secp256k1) key. Then the client can create channels to receive messages. Each channel is associated with a specific key and contains that in the URL to post to that channel. This enables a client to simply provide the URL to someone else and they can parse the public key from the URL to know the public key to use to communicate with them.

To encrypt a message to a channel the sender parses the recipient's public key from the channel URL then calculates the ECDH secret between the sender's private key and the channel's public key and uses that to encrypt the message. The message should also include the sender's unencrypted public key so the recipient knows how to decrypt the encrypted contents of the message.

## Channels

A channel is a specific URL that can be posted to via an HTTP post request. The owner of the channel will receive the HTTP request body contents and 'Content-Type' header value.

This is an example channel URL. It can have any host and prefix path at the beginning but must end with "/channels/" followed by the hex encoding of the compressed public key. A compressed public key is 66 hex characters, representing 33 bytes, starting with '02' or '03'. This ensures the public key can always be easily parsed from the URL.

`https://messages.nextcash.tech/channels/<public_key>`

## Authentication

No authentiation is required to post messages to a channel URL. Authentication is required to connect to the service as a client that can create channels and receive messages.

Documentation for authentication is provided [here](authentication.md).

## Administration

When an account is created a private channel will be established with the account key. Posting to this channel will result in an HTTP status 404 (Not found). The channel will be used for messages from the service to the client, like payment requests. The client can respond to these messages on the channel corresponding to the service's key.

## Fees

Fees will be charged for certain actions within an account. The client is responsible for maintaining a balance on the account and if they do not then their services will be disabled.

When an account's balance is getting low the service will post a payment request on the account's administration channel so the client can keep the account in good standing. The client should respond by completing the payment request and posting it to the service's channel. If the client wants to pay a different amount they can modify the payment request with the endpoint for creating a specific payment request.

## Media Type

Unless otherwise specified the media types for API requests depends on the `Content-Type` header of the request and the media type of the response depends on the `Accept` header of the request. The `Content-Type` header of the response will confirm its media type. The fields for each media type are specified in the definition of each endpoint.

* `application/json` specifies JSON.
* `application/octet-stream` specifies [BSOR](https://github.com/tokenized/pkg/tree/master/bsor).

## API (Application Programming Interface)

### Service Information

HTTP GET Request to URL - `https://<hostname><path_prefix>/service`

No HTTP request header `Authorization` value needed.

The success response will be HTTP status 200 (OK) with the following data structure in the body:
```
	PublicKey bitcoin.PublicKey `json:"public_key" bsor:"1"`
	Fees      Fees              `json:"fees" bsor:"2"`
```

`PublicKey` is the public key of the service. Some messages, like payment requests, will be encrypted and signed with this key. Also, a channel will be created for this key that will be used for client administration messages.

`Fees` is the following data structure:
```
	MinimumBalance int64  `json:"minimum_balance" bsor:"1"`
	Message        uint64 `json:"message" bsor:"2"`
	CreateChannel  uint64 `json:"create_channel" bsor:"3"`
```

`MinimumBalance` is the balance below which only your admin channel will work to provide the client with payment requests. Other channels will return errors when they are posted to. Some services may choose to allow clients to have a negative balance as this can be useful for getting a new wallet funded.

`Message` is the fee per message posted to one of the account's channels.

`CreateChannel` is the fee for creating a new channel under an account.

### Post Message

HTTP POST Request to URL - `https://<hostname><path_prefix>/channels/<channel_key>`

The HTTP request `Content-Type` header value and the contents of the body will be delivered to the owner of the channel.

A success response will be an HTTP status 201 (Created) and an empty body.

### Acknowledge Message

HTTP POST Request to URL - `https://<hostname><path_prefix>/channels/<channel_key>/<sequence>`

The account is determined via the token in the HTTP request header `Authorization` value.

Messages at or below the specified sequence value will be discarded.

A success response will be an HTTP status 200 (OK) and no body.

### Listen for Messages

HTTP POST Request to URL - `wss://<hostname><path_prefix>/listen`

The account to listen for is determined via the token in the HTTP request header `Authorization` value.

A websocket will be opened between the client and service to deliver any available messages. At first all unacknowledged messages will be fed through the websocket and then as new messages are posted to the account's channels they will be fed through.

The data structure of the messages will be in JSON or BSOR depending on the `Accept` header value of the request. JSON messages will be "websocket text" and BSOR will be "websocket binary".

```
	ChannelKey  bitcoin.PublicKey `json:"channel_key" bsor:"1"`
	Sequence    uint64            `json:"sequence" bsor:"2"`
	DateTime    uint64            `json:"datetime" bsor:"3"`
	ContentType string            `json:"content_type" bsor:"4"`
	Body        []byte            `json:"body" bsor:"5"`
```

`ChannelKey` is the public key of the channel the message was posted to.

`Sequence` increments with each consecutive message posted to a channel. It is independent for each channel.

`DateTime` is the nanoseconds since the UNIX epoch.

`ContentType` is the `Content-Type` header of the HTTP request that posted the message.

`Body` is the raw HTTP request body of the HTTP request that posted the message.

### Account Information

HTTP GET Request to URL - `https://<hostname><path_prefix>/accounts`

The account is determined via the token in the HTTP request header `Authorization` value.

The success response will be an HTTP status 200 (OK) with the following data structure in the body:
```
	Balance     int64               `json:"balance" bsor:"1"`
	ChannelKeys []bitcoin.PublicKey `json:"channels" bsor:"2"`
```

`Balance` is the current satoshi balance of the account. Any fee based actions will be deducted from this balance as they happen.

`ChannelKeys` is a list of public keys that are associated with current channels.

### Create Channel

HTTP POST Request to URL - `https://<hostname><path_prefix>/channels`

The account is determined via the token in the HTTP request header `Authorization` value.

The data structure of the request body must be the following:
```
	ChannelKey bitcoin.PublicKey `json:"channel_key" bsor:"1"`
	TimeStamp  uint64            `json:"timestamp" bsor:"2"`
	Signature  bitcoin.Signature `json:"signature" bsor:"3"` // to prove the private key is known
```

`ChannelKey` is the public key to associate with the channel.

`TimeStamp` must be the current time (within 10 seconds of server time).

`Signature` is a signature generated using the channel key and a SHA256 hash of the following data:
```
	Account Key - 33 byte compressed value
	Channel Key - 33 byte compressed value
	TimeStamp - 8 byte little endian integer
```

The signature is required so that the service will only create channels for keys that it knows are owned by the account and a client can't "camp" on public keys it knows belong to other people.

The success response is an HTTP status 201 (Created) with an empty body.

### Remove Channel

HTTP DELETE Request to URL - `https://<hostname><path_prefix>/channels/<channel_key>`

The account is determined via the token in the HTTP request header `Authorization` value.

The success response is an HTTP status 200 (OK) with an empty body.

### Create Specific Payment Request

HTTP POST Request to URL - `https://<hostname><path_prefix>/accounts/payment`

The account is determined via the token in the HTTP request header `Authorization` value.

The data structure of the request body must be the following:
```
	Quantity uint64 `json:"quantity" bsor:"1"`
```

The success response will be an HTTP status 201 (Created) and the body will be and [Envelope containing a payment request](payments.md).
