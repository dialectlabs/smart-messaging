# Smart Messaging

Smart Messages are a URL-encoded specification for interactive messaging. Built on top of Solana Pay, smart messages allow senders to encode arbitrary actions into URLs that will then be rendered and actionable for the recipient.

These actions can be both on-chain transactions as well as off-chain, wallet-authenticated actions. They can be simple token transfers, NFT buyout offers, DAO or multisig votes, social media likes and follows, e-commerce purchasing and referrals, and more.

![5 0](https://user-images.githubusercontent.com/3632945/200105159-0af75807-980c-425f-a72f-5ebe1d714b08.png)

In its minimal form, the Smart Message specification is simply the Solana Pay transaction request spec. The full spec includes support for additional metadata in the GET and POST requests, state that is managed by an intermediate service, usage in broader environments like OpenGraph Link Previews, as well as multi-action and multi-party functionality.

This documentation covers the design of the Smart Message spec, as well as implementation details and examples.

## Why Solana Pay?

Solana Pay translates URL-encoded text into executable transactions. This lends itself naturally to messaging.

# Minimal Specification (Solana Pay)

The Smart Message specification in its simplest form is the [Solana Pay](https://github.com/solana-labs/solana-pay) spec, with support for both Transfer Requests, Transaction Requests, and Sign Message Requests.

When sending smart messages, messaging clients will simply append Solana Pay URLs to the message text. Then, when rendering the sent message, clients will extract Solana Pay URLs to render them in usable ways.

This extract-and-render approach is analogous to how Link Previews are handled in traditional messaging clients.

## Solana Pay: Transfer Request

Read the Solana Pay Transfer Request spec [here](https://github.com/solana-labs/solana-pay/blob/master/SPEC.md#specification-transfer-request).

In the Solana Pay Transfer request, full transaction information is encoded in the url. It is a pure P2P protocol.

The encoding is as follows:

```
solana:<recipient>
  ?amount=<amount>
  &spl-token=<spl-token>
  &reference=<reference>
  &label=<label>
  &message=<message>
  &memo=<memo>
```

### Smart Message: Transfer Request

Smart messages support Transfer Requests, but it is recommended that Transaction Requests be used, even for transfers.

When a messaging client detects a string of the above form in a message, it is extracted from the message content and rendered in the following form.

*rendering

In the above rendering, standard libraries are used to translate `amount` and `spl-token` to `displayAmount` and `displaySplToken`, which are the amount divided by the token's decimals, and the human-readable token name, respectively.

These values are then rendered in the button text as "Send `<displayAmount>` `<displaySplToken>`".

When the button is tapped, a standard client library is used to construct the transfer transaction based on the metadata encoded in the URL. This transaction is then passed to the user's wallet's `signTransaction` method.

The wallet's `signTransaction` then prompts the user to approve or reject the transaction — whether via an RPC call to a local wallet, or embedded in the client app.

Once signed, the transaction is then submitted to the blockchain.

## Solana Pay: Transaction Request

![solana-pay-transaction-request](https://user-images.githubusercontent.com/3632945/200104128-f1d98665-f676-41a4-a0c6-83c6b65650b9.png)

Read the Solana Pay Transaction Request spec here.

The Solana Pay transaction request allows for executing arbitrary transactions. This is done by making a pair of GET and POST requests to a URL that returns human-readable information and a transaction ready for signing and execution, respectively.

```
solana:<url>
```

The full flow for a transaction request is:

1. The client executes a GET to fetch a human-readable text label and image icon for the user to consume, for educational purposes.
2. The client executes a POST, either initiated by the user or in parallel with the GET, to fetch a base64-encoded transaction.
3. The user then takes some action to sign the returned transaction, and submits it to the blockchain for execution.

The schemas for the steps above are described here.

### GET request

The initial GET request returns

`// GET <url> response`
```json
{ "label": "<label>", "icon": "<icon-url>" }
```

where `label` is some descriptive text, and `icon` is some image url for display purposes.

### POST request

The user may then execute a POST to the same url, with the payload and response

POST <url> payload
```json
{ "account": "<account>" }
```

POST <url> response
```json
{ "transaction": "<transaction>" }
```

### Smart Message: Transaction Request

When a messaging client detects a string of the form `solana:<url>` in a message, it is extracted and rendered according to the label and icon provided by the `GET` request.

*rendering

The `label` should be used as the button text. The user may then tap the button to begin execution. On tap, the `POST` request is made, a transaction is returned and passed to the user's wallet's `signTransaction` method.

The wallet's `signTransaction` then prompts the user to approve or reject the transaction — whether via an RPC call to a local wallet, or embedded in the client experience.

Once signed, the transaction is then submitted to the blockchain.

## Solana Pay Sign Message Request

### Smart Message: Sign Message Request

## Limitations of the Minimal Specification

# Full Specification

The full Smart Message Specification extends Solana Pay to include support for

- Additional metadata, including state
- State persistence
- Use in other specifications like OpenGraph Link Previews
- Multi-action support (e.g. "vote yes", "vote no")
- Multi-party support (e.g. group chats)

Additionally, the full specification includes 3 main system components:

1. The Client — the Dialect app or any other messaging app.
2. The Transaction Request Service — This is a Solana Pay Transaction Request Service, or any number of them.
3. [NEW] The State Service — Manages interactions between the Client and Transaction Request Services, while managing state.

The Client and Transaction Request services are mandatory for any implementation of Smart Messaging or Solana Pay. The State Service is only needed to persiste state.

Let's start by extending the metadata returned by Transaction Request services, without preserving any state.

## Metadata

To provide a better user experience, Smart Message Transaction Request services may optionally extend the metadata provided by simple Solana Pay Transaction request services.

### Metadata for state management

Recall that the minimal Solana Pay Transaction Request service GET routes returns a `label` and `icon`, which in a Smart Message context are the button text and image preview, respectively:

GET <url> response
```json
{ "label": "<label>", "icon": "<icon-url>" }
```

However, to provide a best user experience, Transaction Request services may return metadata for every combination of `<component>.<state>.<user_role>` for the components `label` or `icon`, for smart message state, and for users `sender` & `recipient`.

Smart Messages support the following states:

- `ready`
- `invalidated`
- `executing`
- `succeeded`
- `failed`

For example, a token transfer Transaction Request service may provide the following additional metadata:

```json
{
  "icon": "<icon-url>",
  "label": {
    "ready": {
      "sender": "Requested 20 SOL",
      "recipient": "Send 20 SOL"
    },
    "executing": {
      "sender": "Receiving 20 SOL",
      "recipient": "Sender 20 SOL"
    },
    "succeeded": {
      "sender": "Received 20 SOL",
      "receipient": "Sent 20 SOL"
    },
    "failed": "Failed",
    "invalidated": "No longer valid"
  }
}
```

Note that at the `component` or `state` level, the value can be either a string or nested JSON, depending on whether differentiation is needed.

The following defaults are used in the absence of values provided in the above:

```json
{
  "icon": "<icon-url>",
  "label": {
    "ready": {
      "sender": "Requested",
      "recipient": "Execute"
    },
    "executing": "Executing",
    "succeeded": "Succeeded",
    "failed": "Failed",
    "invalidated": "No longer valid"
  }
}
```

### Metadata for enhanced rendering

Beyond a Solana Pay `label` and `icon`, Smart Messages may also provide the following metadata from the GET route:

```json
{
  "icon": "<icon-url>",
  "label": "<label>",
  "title": "<title>",
  "description": "<description>"
}
```

The values `title` and `description` allow for Link Preview-like smart messages, which provide more content for the user. In this form, the `label` is still used as the text for the button, which is now placed beside or below the title and description. The `icon` serves as the preview image.

### Transaction Request Service state

Transaction Request services may optionally return a `state`, of value `ready` or `invalidated`, since both of these are possible states before any user action.

For example, if Alice receives a buyout offer smart message from Bob on an NFT, but circumstances change such that she either no longer owns the NFT, or Bob cancels the offer, the smart message metadata should reflect that the "accept offer" action is no longer valid.

```json
{
  "icon": "<icon-url>",
  "label": "<label>",
  "state": "invalidated",
}
```

Additionally, the same validation may be performed in the Transaction Request POST route, which can also return `ready` or `invalidated`.

No `state` provided at all is equivalent to `ready`.

Developers building Transaction Request services should take care to implement validation only if it is needed for the end user experience, since this involves additional computational overhead on the GET and POST routes.

Transaction Request services will never return `executing`, `succeeded`, or `failed`, as these states are specific to the smart message user interaction flow, and are handled by the state service.

## State persistence

Smart Messages are not ephemeral. They persist in the chat long after their use. Solana Pay is deliberately stateless, but persisting smart message state can help with usability and user experience.

In this section, we introduce the State Service, and describe how it handles requests between the Client and Transaction Request services.

The State Service supports the following routes:

1. A GET route that forwards the request to the underlying Transaction Request API GET route.
2. A POST route that forwards the request to the underlying Transaction Request API POST route.
3. A PUT route for submitting transactions to the blockchain (to monitor for completion and update to `succeeded` or `failed`).
4. [FUTURE] A PUT route for updating smart message state by its UUID. This route is not supported in the current Smart Message implementation.

![smart-messaging-diagrams-1](https://user-images.githubusercontent.com/3632945/200104076-dd4d27db-b168-48fb-abb2-79db6dd832b8.png)

### UUID Parameter

To persist data, we first need some kind of reference id. Clients who wish to persist smart message state must generate a UUID and append it as a query parameter in the url in solana:<url>.

```
solana:<url>&uuid=<generated-uuid>
```

By using a UUID encoded in the url itself, smart message state is decoupled from any messaging-related state.

The URL still specifies the URL for the Transaction Request API. If a UUID is present in this URL, the client will interpret this as an intent to pass calls through the State Service to manage state.

This UUID parameter is then used across the smart messaging system for tracking smart message state.

### Routing Transaction Request GETs & POSTs

**GET**

Clients that detect a UUID in the URL will pass GET calls through the State Service.

```
// Client
GET <state-service-url>?url=<url>
```

The request is then forwarded to the Transaction Request at the provided solana `url`.

```
// State service
GET <url>
```

This Transaction Request service may then return metadata as described above.

If no record exists with id `uuid`, a new record is created with all of the returned metadata.

The state service then returns the metadata to the client.

**POST**

Post requests are similarly passed through the state service using the UUID in the url, and as described for GET, may create new records based on the UUID, and may update state from `ready` to `invalidated` during validation.

```
// Client
POST <state-service-url>?url=<url>
```

```
// State service
POST <url>
```

This route then returns the result of the POST request back to the client.

### Tracking transaction state

Transaction Request service POST requests return a transaction ready for signing by the client.

Once signed by the user, the transaction can be submitted back to the State Service for submission to the blockchain via a PUT request

Client PUT <state-service-url>
```json
{ "transaction": "<signed-transaction>" }
```

The State service then manages submitting the signed transaction to the blockchain, updating smart message state to `executing`, monitoring transaction finality, and then settling state to either `succeeded` or `failed`.

**Terminal states**

Message states `ready` and `executing` are non-terminal states, in that they can further transition to other states. States `succeeded`, `failed`, and `invalidated` are terminal. This will be important later.

## Smart Message usage in OpenGraph Link Previews

![6 0](https://user-images.githubusercontent.com/3632945/200104889-b17dc897-0384-4263-9617-b9619cf8f89f.png)

Develoeprs may add an additional `dialect` meta tag in the `<head>` section of their sites. Any Dialect comptabible clients will then render a button on the link preview according to the smart message.

```html
<html>
  <head>
    <title>...</title>
    <meta property="og:title" content="..." />
    <meta property="og:description" content="..." />
    <meta property="dialect" content="solana:<url>" />
    ...
  </head>
  ...
</html>
```

## Multi-Action Smart Messages

Future versions of the Smart Message specification will support multiple actions, such as vote yes or vote no on a DAO proposal or multisig.

![5 0](https://user-images.githubusercontent.com/3632945/200104298-1b8a7562-dba1-4a1f-ba1b-8c240a254712.png)

## Multi-Party Smart Messages

Smart Message support for multi-party environments, such as group chat, will be coming in a future version of the specification.
