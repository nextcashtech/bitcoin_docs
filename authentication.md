# Authentication

To initiate authentication the user sends an HTTP GET request to the path `https://<hostname><prefix>/auth/<account_key>` where the `<hostname><prefix>` is published by the service and `<account_key>` is the hex encoded public key the user wants to use to authenticate with the service.

The response will be HTTP 201 (Created) and the response body will either JSON or BSOR depending on the request header's `Accept` value. `application/json` for JSON or `application/octet-stream` for BSOR.

This is the structure:

```
	PublicKey       bitcoin.PublicKey `json:"public_key" bsor:"1"`
	Nonce           bitcoin.Hash32    `json:"nonce" bsor:"2"`
	Issued          uint64            `json:"issued" bsor:"3"`
	ChallengeExpiry uint64            `json:"challenge_expiry" bsor:"4"`
	Expiry          uint64            `json:"expiry" bsor:"5"`
```

The user then must calculate the SHA256 hash of this message using the following data:

```
	PublicKey - 33 byte compressed value
	Nonce - 32 bytes
	Issued - 8 byte little endian
	Expiry - 8 byte little endian
```

The user then must sign this hash with the private key of the account key provided in the initial request and post that with an HTTP POST request to the same URL `https://<hostname><prefix>/messages/v1/auth/<account_key>`. The body of the request can be JSON or BSOR with the corresponding HTTP header `Content-Type` value. `application/json` for JSON or `application/octet-stream` for BSOR.

This is the body structure:

```
	Hash      bitcoin.Hash32    `json:"hash" bsor:"1"`
	Signature bitcoin.Signature `json:"signature" bsor:"2"`
```

Then the service will verify the signature and return an HTTP status 200 (OK) with no body.

Until the expiry date the `Hash` value can be used as the HTTP header value `Authorization` to authenticate future requests.