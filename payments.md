# Payments

Payment requests and responses should be wrapped in an [Envelope](https://tsc.bsvblockchain.org/standards/envelope-specification/) using a few protocols to specify the data needed.

There are more complex scenarios possible for wallets that support it, but a standard payment request would just be a partial transaction with only outputs. The outputs specify what the sender of the payment request wants to receive. The response to that payment request is a completed transaction with matching outputs and added inputs (and likely change outputs) that fund the transaction and make it valid according to all relevant rules.

# Envelope

The Envelope protocol is encoded with Bitcoin script, so is easily embedded in transactions if needed, and should be familiar to Bitcoin developers.

The format is as follows:
* The header to signify it is unspendable data in Envelope format version 1. (OP_FALSE, OP_RETURN, 0xbd01)
* A number representing the number of protocols used.
* A script item for each protocol identifier.
* A number representing the number of data script items that compose the payload of the Envelope.
* The specified number of push datas for the payload of the Envelope.

## Bitcoin Script

Envelopes are encoded in Bitcoin script. Bitcoin script is composed of "op codes" and "push datas". An op code is a single byte that has a specific meaning. A push data is an encoded size of the data followed by the data.

A Bitcoin script push data starts with one of the following:
* A byte value equal to or less than 0x4b. The integer value of this byte is the size of the push data.
* The byte 0x4c. The next byte's integer value is the size of the push data.
* The byte 0x4d. The next two byte's integer value (little endian) is the size of the push data.
* The byte 0x4e. The next four byte's integer value (little endian) is the size of the push data.

# Protocols

These protocols are combined into an envelope to make a payment request or response. Protocol identifiers don't have to be ASCII, but I find it easiest to work with. We just have to make sure everyone is using the same identifier to specify the same format or we won't understand each other's messages.

## Text Identification (TID)

The protocol identifier is ASCII "TID" or binary 0x544944.

A single push data is interpreted as an ASCII identifier.

## Message URL (M_URL)

The protocol identifier is ASCII "M_URL" or binary 0x4d5f55524c.

A single push data is interpreted as an ASCII URL for posting messages.

## Public Key (PK)

The protocol identifier is ASCII "PK" or binary 0x504b.

The push data must be a fixed size of 33 bytes that represent a public key.

## Fee Rate (FEE)

The protocol identifier is ASCII "FEE" or binary 0x464545.

The next push data is interpreted as a fee rate. The fee rate is the following [BSOR](https://github.com/tokenized/pkg/tree/master/bsor) encoded structere:
```
	Satoshis  uint64 `json:"satoshis" bsor:"1"`   // satoshis to charge for the specified quantity of bytes
	ByteCount uint64 `json:"byte_count" bsor:"2"` // quantity of bytes to charge satoshis
```

## Note (NOTE)

The protocol identifier is ASCII "NOTE" or binary 0x4e4f5445.

A single push data is interpreted as ASCII text.

## Encrypt (E)

The protocol identifier is ASCII "E" or binary 0x45.

The next push data is a 16 byte initialization vector (IV) and the push data after that is the encrypted data.

## Signature (S)

The protocol identifier is ASCII "S" or binary 0x53.

The next push data contains a signature that is over all the following push datas of the Envelope.

## Transaction (BEEF)

The protocol identifier is ASCII "BEEF" or binary 0x42454546.

The next push data contains [BRC-0062 BEEF](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md) encoded transaction data.

# Example

## Hex Decompile of Payment Request

This is an example payment request represented in hex and broken down by piece. The actual data in the Bitcoin script is binary, so sizes are in binary. It is just represented in hex here to help explain it. The hex size is twice as big as binary since it takes two hex characters to represent one binary byte.

Here is the full hex payload:
```
006a02bd015603544944054d5f55524c02504b014501530442454546552465643561313266352d663866392d343536322d623138332d3732373639383234303965370b746573743a2f2f7465737421026233c68852e48c6efcc0e679fed53ec10d014e3e5cdeb2a1720eb22ff49f3671105d5df72924f38ef1b25708b63790a9cb4c8041ad1d1c1c4c910c2e21f3fc16de2b049b70ed2a1f19e6c12aafd4ef1a47363906e2af5271d5ed17e507683d17c7aa75135cefb68c22113028cf65daf1793f5271461cb2a04fbcf1d83c5525f44a9c33f955506f017c3058d38b939baf7f8998839e52ad078d53867cf07134cee127ff00c3bed7c743104d8b2e1cfda7f558cf
```

`006a02bd01` - OP_FALSE, OP_RETURN, 0xbd01 (02 is the size of the push data containing 0xbd01)

Protocol Identifiers:
`56` - OP_6 specifying that 6 protocol identifiers follow

* `03544944` - "TID" (prepended with push data size of 3 bytes)
* `054d5f55524c` - "M_URL" (prepended with push data size of 5 bytes)
* `02504b` - "PK" (prepended with push data size of 2 bytes)
* `0145` - "E" (prepended with push data size of 1 byte)
* `0153` - "S" (prepended with push data size of 1 byte)
* `0442454546` - "BEEF" (prepended with push data size of 4 bytes)

`55` - OP_5 specifying that 5 push datas follow that are included in the Envelope.

Text Identifier:
`2465643561313266352d663866392d343536322d623138332d373237363938323430396537`

ASCII "ed5a12f5-f8f9-4562-b183-7276982409e7" (prepended with push data size of 0x24 or 36 bytes)

Message URL:
`0b746573743a2f2f74657374`

"test://test" (prepended with push data size of 0x0b or 11 bytes)

Public Key:
`21026233c68852e48c6efcc0e679fed53ec10d014e3e5cdeb2a1720eb22ff49f3671`

The public key `0x026233c68852e48c6efcc0e679fed53ec10d014e3e5cdeb2a1720eb22ff49f3671` (prepended with push data size of 0x21 or 33 bytes)

Encrypt:

The initialization vector (IV) for the encrypted data:
`105d5df72924f38ef1b25708b63790a9cb`

The initialization vector `0x5d5df72924f38ef1b25708b63790a9cb` (prepended with push data size of 0x10 or 16 bytes)

`4c80` - This is a special push data size. The `0x4c` means the next byte represents the size. The prior push data sizes don't require the `0x4c` byte because they are less than the threshold `0x4b` for encoding the size in a single byte.

This is the encrypted data contained in the Envelope:
```
41ad1d1c1c4c910c2e21f3fc16de2b049b70ed2a1f19e6c12aafd4ef1a47363906e2af5271d5ed17e507683d17c7aa75135cefb68c22113028cf65daf1793f5271461cb2a04fbcf1d83c5525f44a9c33f955506f017c3058d38b939baf7f8998839e52ad078d53867cf07134cee127ff00c3bed7c743104d8b2e1cfda7f558cf
```

The encryption secret is a Diffie-Hellman secret generated between the sender's and recipient's keys. One of the private keys of those two and the other public key is required to derive the secret. This is obviously not encoded in the Envelope as it would defeat the purpose of encrypting in the first place. Multiplying the private key by the public key results in an x and y value. The secret is the x value left padded to 32 bytes with zero.

Secret: `0xba58c188319aa0ec25babc50a79d47c3f5f829254c290003d2c91089da158215`

Decrypted data using the secret and IV:
```
46304402205fef5ccac796d4f32b429a0c846a5e1b2dfd3a2e9e41dedd2cc1c8beb62ecfd60220522dcc63515553a7b8884562f1860755129b0a50d4d3e52f95c2c1502b040c2d330100beef000101000000000110270000000000001976a914384adcbfc86280b28c1f43f3912aab8df14a4dd288ac0000000000
```

When we parse the decrypted data as a Bitcoin script we get two push datas. Since there are two remaining protocols remaining in the Envelope's list of protocol identifiers we use them to process the decrypted data.

Signature:
`46304402205fef5ccac796d4f32b429a0c846a5e1b2dfd3a2e9e41dedd2cc1c8beb62ecfd60220522dcc63515553a7b8884562f1860755129b0a50d4d3e52f95c2c1502b040c2d`

The signature `0x304402205fef5ccac796d4f32b429a0c846a5e1b2dfd3a2e9e41dedd2cc1c8beb62ecfd60220522dcc63515553a7b8884562f1860755129b0a50d4d3e52f95c2c1502b040c2d` (prepended with push data size of 0x46 or 70 bytes). The signature is over the remaining push datas in the Envelope encoded in Bitcoin script and can be verified with the public key included in the payload or one must be referenced by context.

BEEF:
`330100beef000101000000000110270000000000001976a914384adcbfc86280b28c1f43f3912aab8df14a4dd288ac0000000000`

BEEF data `0x0100beef000101000000000110270000000000001976a914384adcbfc86280b28c1f43f3912aab8df14a4dd288ac0000000000` (prepended with push data size of 0x33 or 51 bytes)

This decodes to a transaction with one output (following) and no ancestors or merkle paths.
```
{
  "locking_script": "76a914384adcbfc86280b28c1f43f3912aab8df14a4dd288ac",
  "value": 10000
}
```

Full JSON representation (for documentation/display purposes only):
```
{
  "ID": "ed5a12f5-f8f9-4562-b183-7276982409e7",
  "CallBackURL": "test://test",
  "Tx": {
    "transaction": {
      "txid": "4daad71c697a9b533791b3f35c022aa54e8d616382d30bd2022e3011719b8b03",
      "version": 1,
      "inputs": [],
      "outputs": [
        {
          "locking_script": "76a914384adcbfc86280b28c1f43f3912aab8df14a4dd288ac",
          "value": 10000
        }
      ],
      "locktime": 0
    },
    "ancestor_txs": [],
    "merkle_paths": []
  },
  "FeeRate": null,
  "Sender": "026233c68852e48c6efcc0e679fed53ec10d014e3e5cdeb2a1720eb22ff49f3671",
  "WasSigned": true,
  "WasEncrypted": true
}
```