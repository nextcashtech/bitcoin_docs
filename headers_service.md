# Headers Service

The provides HTTP URLs to retrieve and listen for block headers.

This service implements the standard [service API](services.md) for identification, authentication, and payments.

## Bootstrap

The bootstrap endpoint allows fetching historical headers for wallet initialization.

HTTP GET Request to URL - `https://<hostname><path_prefix>/headers/bootstrap/<height>/<count>`

Height is the block height of the first block header to return.
Count is the maximum headers to return.

A success response will be an HTTP status 200 (Okay) and the body will contain a list of consecutive headers from a single branch/chain. If the `Accept` request header is `application/octet-stream` then the response body will be the raw 80 bytes for each header concatenated together. If `Accept` is `application/json` then the response body will be the hex 160 bytes for each header concatenated together.

## Sync

The sync endpoint allows fetching recent headers for when a wallet is coming back online. It will return relevant branches and connect back to the most POW chain if the header hash provided is no longer on the most POW chain.

HTTP GET Request to URL - `https://<hostname><path_prefix>/headers/after/<after_hash>/<max_depth>`

After hash is the hash of the latest header known to the client.
Max depth is the maximum depth in the chain to search for matches. This will be limited by the service so deeper requests should use the bootstrap endpoint.

If the "after hash" is not known by the service then an HTTP status 404 (Not found) will be returned. The bootstrap endpoint should be used to recover from this.

A success response will be an HTTP status 200 (Okay) and the body will contain a list of headers. The headers are not guaranteed to be consecutive or from the same branch/chain as they will include multiple branches to bring you back up to sync. If the `Accept` request header is `application/octet-stream` then the response body will be the raw 80 bytes for each header concatenated together. If the `Accept` request header is `application/json` then the response body will be the hex 160 bytes for each header concatenated together.

## Listen

The listen endpoint creates a websocket to listen for new headers as they are announced on the Bitcoin network.

Websocket Request to URL - `wss://<hostname><path_prefix>/listen`

New headers will be sent over the websocket as the service receives them from the Bitcoin network.

If the `Accept` request header is `application/octet-stream` then the each message will be the raw 80 bytes for each header. If the `Accept` request header is `application/json` then the response body will be the hex 160 bytes for each header.