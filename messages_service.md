# Messages Service

The messages service provides HTTP URLs for a user to receive messages. Since most users won't have domain names or want to host web services this is required for user to user and service to user communication. The messages should be signed and encrypted whenever possible so little to no trust is required for the messaging service. The service must simply stay up and deliver messages to the appropriate recipients.

A user authenticates with the service using a secp256k1 key. Then the user can create channels to receive messages. Each channel is associated with a specific key and contains that in the URL to post to that channel. This enables a user to simply provide the URL to someone else and they can parse the public key from the URL to know the public key to use to communicate with them.

To encrypt a message to a channel the sender parses the recipient's public key from the channel URL then calculates the ECDH secret between the sender's private key and the channel's public key and uses that to encrypt the message. The message should also include the sender's unencrypted public key so the recipient knows how to decrypt the encrypted contents of the message.

## Channels

A channel is a specific URL that can be posted to via an HTTP post request. The owner of the channel will receive the HTTP request body contents and 'Content-Type' header value.

This is an example channel URL. It can have any host and prefix path at the beginning but must end with "/channels/" followed by the hex encoding of the compressed public key. A compressed public key is 66 hex characters, representing 33 bytes, starting with '02' or '03'. This ensures the public key can always be easily parsed from the URL.

`https://messages.nextcash.tech/channels/<public_key>`

## API (Appliation Programming Interface)

### Authentication

No authentiation is required to post messages to a channel URL when the URL is provided by another party. Authentication is required to connect to the service as a user that can create channels and receive messages.

Documentation for authentication is provided [here](authentication.md).