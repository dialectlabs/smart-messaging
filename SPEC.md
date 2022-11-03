# Smart Messages

Smart Messages are a URL-encoded specification for interactive messaging. Built on top of Solana Pay, smart messages allow senders to encode arbitrary actions into URLs that will then be rendered and actionable for the recipient.

These actions can be both on-chain transactions as well as off-chain, wallet-authenticated actions. They can be simple token transfers, NFT buyout offers, DAO or multisig votes, social media likes and follows, e-commerce purchasing and referrals, and more.

In its minimal form, the Smart Message specification is simply the Solana Pay transaction request spec. The full spec includes support for additional metadata in the GET and POST requests, state that is managed by an intermediate service, usage in broader environments like OpenGraph Link Previews, as well as multi-action and multi-party functionality.

This documentation covers the design of the Smart Message spec, as well as implementation details and examples.

## Why Solana Pay?

Solana Pay translates URL-encoded text into executable transactions. This lends itself naturally to messaging.

# Minimal Specification (Solana Pay)

The Smart Message specification in its simplest form is the Solana Pay spec, with support for both Transfer Requests, Transaction Requests, and Sign Message Requests.

When sending smart messages, messaging clients will simply append Solana Pay URLs to the message text. Then, when rendering the sent message, clients will extract Solana Pay URLs to render them in usable ways.

This extract-and-render approach is analogous to how Link Previews are handled in traditional messaging clients.

## Solana Pay: Transfer Request

Read the Solana Pay Transfer Request spec here.

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

```json
// GET <url> response
{ "label": "<label>", "icon": "<icon-url>" }
```

where `label` is some descriptive text, and `icon` is some image url for display purposes.

### POST request

The user may then execute a POST to the same url, with the payload and response

```json
// POST <url> payload
{ "account": "<account>" }

// POST <url> response
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

- Persistent state
- Additional metadata
- Use in broader environments like OpenGraph Link Previews
- Multi-action support (e.g. "vote yes", "vote no")
- Multi-party support (e.g. group chats)

## Persistent state

Smart Messages are not ephemeral. They persist in the chat long after their use. Persisting smart message state can help with usability and user experience.

In this section we describe the optional states that Smart Messages support, as well as how that state can be tracked and persisted.

A fully state-persisted system involves the following services:

1. The Client — the Dialect app or any other messaging app.
2. The Transaction Request Service — This is a Solana Pay Transaction Request Service, or any number of them.
3. The State Service — Manages interactions between the Client and Transaction Request Services to manage state.

*diagram

### UUID Parameter

Clients who wish to persist smart message state must generate a UUID and append it as a query parameter in the url in solana:<url>.

```
solana:<url>&uuid=<generated-uuid>
```

By using a UUID encoded in the url itself, smart message state is decoupled with any messaging-related state.

The URL still specifies the URL for the Transaction Request API. If a UUID is present in this URL, the client will interpret this as an intent to proxy calls through the State Service to manage state.

This UUID parameter is then used across the smart messaging system for tracking smart message state.

### States

Smart Messages may be in one of the following states:

- `ready`
- `executing`
- `succeeded`
- `failed`
- `invalidated`

Smart messages in state `ready` are ready for execution. If they are `executing` then they are in progress and will either resolve to `succeeded` or `failed`. Messages in state `succeeded` have completed successfully and may no longer be executed. Those in state `failed` may be retried. Smart messages in state `invalidated` are deemed no longer valid.

Smart Messages with no state are considered `ready`.

**Terminal states**

Message states `ready` and `executing` are non-terminal states, in that they can further transition to other states. States `succeeded`, `failed`, and `invalidated` are terminal. This will be important later.

**Pre- and post-action states**

It is worth noting that the states `executing`, `succeeded`, and `failed` are only reachable by some action taken by the user in a smart messaging context, but `invalidated` is reachable without any user taking any action.

For example, if Alice receives a buyout offer smart message from Bob on an NFT, but circumstances change such that she either no longer owns the NFT, or Bob cancels the offer, the smart message metadata should reflect that the "accept offer" action is no longer valid.

Because of this, `invalidated` is treated slightly differently than `executing`, `succeeded`, and `failed`, in that it can be the result of some pre-validation performed by the Transaction Request Service. This will be explained in the Transaction Request Service section below.

### Transaction Request Service

The Transaction Request Service is equivalent to the Solana Pay Transaction Request API that developers may build to support Solana Pay Transaction Requests, but with some optional additional metadata and responsibilities.

*diagram

**Request validation (`ready` or `invalidated`)**

Smart Messages about actions that are no longer valid should be rendered as such — e.g. a mint that has run out, or an NFT that is no longer for sale.

To handle this, the GET and POST requests to the Transaction Request service may optionally do some validation to return a `state` attribute that is either in state `ready` or `invalidated`. If no state is returned at all, it is considered `ready`.

Developers building Transaction Request services should take care to implement validation only if it is needed for the end user, since this involves additional computational overhead.

Transaction Request services will never return `executing`, `succeeded`, or `failed`, as these states are specific to the smart message user interaction flow, and are handled by the state service.

### State Service

When persisting state, the State Service can be used between the client and any Transaction Request APIs. This intermediate state service forwards all requests to the Transaction Request URLs in question, while managing state for any given smart message.

This service allows Transaction Request APIs to remain stateless, as minimal extensions of the Solana Pay Transaction Request APIs.

Any State Service should support the following routes.

1. A GET route that forwards the request to the underlying Transaction Request API GET route.
2. A POST route that forwards the request to the underlying Transaction Request API POST route.
3. A PUT route for submitting transactions to the blockchain (to monitor for completion and update to `succeeded` or `failed`).
4. [FUTURE] A PUT route for updating smart message state by its UUID. This route is not supported in the current Smart Message implementation.

The first two routes are passthrough calls from the client to the underlying Transaction Request APIs, managing intermediate state.

The third route manages transaction execution and subsequent management to track & update final state (`executing`, `succeeded`, or `failed`).


## Metadata

To provide a better user experience, Smart Message Transaction Request services may optionally extend the metadata provided by simple Solana Pay Transaction request services.

### Metadata for state management

Recall that the minimal Solana Pay Transaction Request service GET routes returns a  `label` and `icon`, which in a Smart Message context are the button text and image preview, respectively:

```json
// GET <url> response
{ "label": "<label>", "icon": "<icon-url>" }
```

However, to provide a best user experience, Transaction Request services may return metadata for every combination of `<component>.<state>.<user_role>` for the components `label` or `icon`, for smart message states described above, and for users `sender` & `recipient`.

For example, a token transfer Transaction Request service should provide the following additional metadata:

```json
// TODO: Disambiguate sender/recipient nomenclature
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
// TODO: Disambiguate sender/recipient nomenclature
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

Beyond a Solana Pay `label` and `icon`, Smart Messages may also provide the following metadata:

```json
// TODO: Consolidate this <>-y format with exampled formats above
{
  "icon": "<icon-url>",
  "label": "<label>",
  "title": "<title>",
  "description": "<description>"
}
```

The values `title` and `description` allow for Link Preview-like smart messages, which provide more content for the user. In this form, the `label` is still used as the text for the button, which is now placed beside or below the title and description. The `icon` serves as the preview image.

## Smart Message usage in OpenGraph Link Previews



## Multi-Action Smart Messages

## Multi-Party Smart Messages