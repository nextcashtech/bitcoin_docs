# Services

Most services will have common functions like authentication, initiating a client/service relationship, and making payments to the service.

## Media Type

Unless otherwise specified the media types for API requests depend on the `Content-Type` header of the request and the media type of the response depends on the `Accept` header of the request. The `Content-Type` header of the response will confirm its media type. The fields for each media type are specified in the definition of each endpoint.

* `application/json` specifies JSON.
* `application/octet-stream` specifies [BSOR](https://github.com/tokenized/pkg/tree/master/bsor).

## Authentication

Services require authentication so they know which client they are communicating with.

Documentation for authentication is provided [here](authentication.md).

## Initiation

Services require two way communication with a client to send payment requests and receive completed payments.

Documentation for initiation is provided [here](initiation.md).

## Payments

Services require payments to cover costs and provide profit to the provider.

Documentation for payments is provided [here](payments.md).

## Definition

Services need to provide information about how to interact with the service, what services are provided, and how much they cost.

HTTP GET Request to URL - `https://<base_url>/service`

No HTTP request header `Authorization` value needed.

The success response will be HTTP status 200 (OK) with the following data structure in the body:
```
	PublicKey   bitcoin.PublicKey `json:"public_key" bsor:"1"`
	ServiceURLs []ServiceURL      `json:"service_urls" bsor:"2"`

	// MinimumBalance is the minimum balance the service requires for an account to be active. It
	// can be set to a negative number to allow accounts to go into a negative balance before
	// becoming inactive. This allows the client to establish a channel and receive funds to pay the
	// service.
	MinimumBalanceToken wallet.TokenID `json:"minimum_balance_token" bsor:"3"`
	MinimumBalance      int64          `json:"minimum_balance" bsor:"4"`

	Fees    []Fee  `json:"fees" bsor:"5"`
	Payload []byte `json:"payload" bsor:"6"` // Custom service information
```

`PublicKey` is the public key of the service. Some messages, like payment requests, will be encrypted and signed with this key. Messages sent to the service using envelopes should be encrypted to this key.

`ServiceURLs` specify the full url (base URL and path) to use to access specific common features of the service.
```
	Name string `json:"name" bsor:"1"`
	URL  string `json:"url" bsor:"1"`
```

Valid service URL names are:
* AUTH - HTTP GET to request an auth challenge, HTTP POST to complete the challenge.
* BALANCE - HTTP GET with an 'Authorization' header obtained from the AUTH process will return the account balance.
* PAYMENTS - HTTP GET with an 'Authorization' header obtained from the AUTH process will return a payment request.
* MESSAGES - HTTP POST posts messages to the service.

Fees specify the fees that will be charged for the usage of the service.
```
	Name        string `json:"name" bsor:"1"`
	Description string `json:"description" bsor:"2"`

	// Token is the token that the fee is charged in. The default is "BSV" meaning the value is in
	// satoshis.
	Token wallet.TokenID `json:"token" bsor:"3"`

	// Value is the number of tokens for the fee.
	Value uint64 `json:"value" bsor:"4"`

	// FrequencyMultiplier specifies how many of the frequency to use. For example a multiplier of 5
	// with a frequency of days means the fee is charged every 5 days.
	Frequency           Frequency `json:"frequency,omitempty" bsor:"5"`
	FrequencyMultiplier uint64    `json:"frequency_multiplier,omitempty" bsor:"6"`

	// Count is the number of items per fee. For example you can be charged 100 satoshis per 10
	// messages.
	Count uint64 `json:"count,omitempty" bsor:"7"`

	// Size is the size per fee. For example you can be charged 100 satoshis per 1000 bytes.
	Size uint64 `json:"size,omitempty" bsor:"8"`
```

`Token` is text to identify the token being used. Empty or BSV means the tokens are satoshis.

`Frequency` is an integer representing a duration:
* 1 - second
* 2 - minute
* 3 - hour
* 4 - day
* 5 - week
* 6 - month
* 7 - year

`FrequencyMultiplier` is the how many of the duration to add together.