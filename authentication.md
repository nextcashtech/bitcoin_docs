# Authentication

## Peer to Peer

Peer to peer authentication simply involves signing messages with a key the peer knows.

TODO - Describe Envelope Signing protocol.

## Client to Service

Client to service authentication is simply proving the client is the owner of a specific key and then building a relationship around that key. A client can authenticate with any key and then use that service via that key.

To initiate authentication the client sends an HTTP GET request to the path specified in the service definition that is provided by the service.

The success response will be HTTP 201 (Created) and the response body will either be JSON or BSOR depending on the request header's `Accept` value. `application/json` for JSON or `application/octet-stream` for BSOR.

This is the response structure:

```
	PublicKey       bitcoin.PublicKey `json:"public_key" bsor:"1"`
	Nonce           bitcoin.Hash32    `json:"nonce" bsor:"2"`
	Issued          uint64            `json:"issued" bsor:"3"`
	ChallengeExpiry uint64            `json:"challenge_expiry" bsor:"4"`
	Expiry          uint64            `json:"expiry" bsor:"5"`
```

The client then must calculate the SHA256 hash of this message using the following data:

```
	Identifier - string "AUTH" (this ensures the signature is specific to authentication)
	PublicKey - 33 byte compressed value
	Nonce - 32 bytes
	Issued - 8 byte little endian
	Expiry - 8 byte little endian
```

The client then must sign this hash with the private key of the account key provided in the initial request and post that with an HTTP POST request to the same path specified in the service definition that is provided by the service. The body of the request can be JSON or BSOR with the corresponding HTTP header `Content-Type` value. `application/json` for JSON or `application/octet-stream` for BSOR.

This is the body structure of the HTTP POST request:

```
	Hash      bitcoin.Hash32    `json:"hash" bsor:"1"`
	Signature bitcoin.Signature `json:"signature" bsor:"2"`
```

Then the service will verify the signature and return an HTTP status 200 (OK) with no body.

Until the expiry date the `Hash` value can be used as the HTTP header value `Authorization` to authenticate future requests.