---
caip: 295
title: Browser Wallet Messaging for Iframes
author: Pedro Gomes (@pedrouid)
discussions-to: https://github.com/ChainAgnostic/CAIPs/issues/295
status: Draft
type: Standard
created: 2024-06-26
requires: 25, 27, 282
---

## Simple Summary

CAIP-295 defines a standardized messaging transport for browser iframe wallets.

## Abstract

To interface with a Decentralized Application (dapp), users install browser wallets to manage their blockchain accounts, which are required to sign messages and transactions. Leveraging existing browser messaging APIs, these are used to initiate a dapp-wallet connection in a browser environment.

## Motivation

Currently, in order for Decentralized Applications to be able to support all users they need support different messaging standards for each namespace such as Ethereum's [EIP-6963][eip-6963], Solana Wallet Protocol, etc., they do not cover all wallets and are not chain-angostic.

Developers must support different SDKs for different blockchain namespaces which work very differently despite following the same patterns of discovery, handshake and signing.

This proposal is motivated by the fragmentation on the events across different ecosystems and aims to bring cohesion to reduce the unnecessary logic to support multiple chains in different namespaces.

Additionally this aligns the messaging for browser wallets to leverage existing standards for handshake (CAIP-25) and signing (CAIP-27). Thus introducing a solution focused on optimizing interoperability for multiple Wallet Providers, fostering fairer competition by reducing the barriers to entry for new Wallet Providers, and enhancing user experience across all blockchain networks.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC-2119].

### Definitions

Wallet Provider: A user agent that manages accounts and facilitates transactions with a blockchain.

Decentralized Application (dapp): A web page that relies upon one or many Web3 platform APIs which are exposed to the web page via the Wallet.

Blockchain Library: A library or piece of software that assists a dapp to interact with a blockchain and interface with the Wallet.

### Messaging APIs

Browser Extensions make use of events with `window.postMessage` and `window.addEventListener` which will be necessary for publishing and listening messages, respectively, between the Blockchain Library used by the Decentralized Application to communicate with the Wallet Provider.

This provides the foundation for any Wallet Provider to interface with a Decentralized Application using a Blockchain Library which implements this standard.

Different loading times can be affected by multiple factors, which makes it non-deterministic to publish and listen to messages from different sources within the browser.

#### Discovery

Both Wallet Providers and blockchain libraries must listen to incoming messages that might be published after their initialization. Additionally both Wallet Providers and blockchain libraries must publish a message to both announce themselves and their intent to connect, respectively.

Here is the expected logic from the Blockchain Library:

```typescript
interface WalletMapEntry {
  data: WalletAnnounceRequestParams;
  targetOrigin: string;
}

const wallets: Record<string, WalletMapEntry> = {}

// Blockchain Library starts listening on init
window.addEventListener("message", (event) => {
  if (event.data.method === 'wallet_announce') {
    // when an announce message was received then the library can index it by uuid
    wallets[event.data.params.uuid] = {
      params: event.data.params,
      targetOrigin: event.targetOrigin
    }
  }
});

// Blockchain Library publishes on init
window.postMessage({
  id: 1,
  jsonrpc: "2.0"
  method: "wallet_prompt",
  params: {
    // optionally the Blockchain Library can prompt wallets to announce matching only the chains
    chains: []  // optional
    //  if the Blockchain Library supports CAIP-275 then it can include a name
    authName: "", // optional
  },
});
```

Here is the expected logic from the Wallet Provider:

```typescript
// Wallet Provider sets data on init
const walletData = { ... }

// Wallet Provider publishes on init
window.postMessage({
  id: 2,
  jsonrpc: "2.0"
  method: "wallet_announce",
  params: walletData
});


// Wallet Providers starts listenning on init
window.addEventListener("message", (event) => {
  if (event.data.method === 'wallet_prompt') {
    // when a prompt message was received then the wallet will announces again
    window.postMessage({
      id: 2,
      jsonrpc: "2.0"
      method: "wallet_announce",
      params: walletData
    });
  }
});
```

#### Handshake

After the wallet has been selected by the user then the Blockchain Library MUST publish a message to share its intent to establish a connection. This can be either done as a [CAIP-25][caip-25] request.

The communication will use the `uuid` shared by the initial Wallet Provider announcement payload, which the Wallet Provider will listen to for any incoming requests, and consequently, the Blockchain Library will also be used for publishing messages. The same will happen again the other way around but vice-versa, where the Wallet Provider will be the Blockchain Library that will be listening to any incoming responses, and consequently, the Wallet Provider will also use it for publishing messages.

#### Signing

This same channel `uuid` can then be used for a connected session using [CAIP-27][caip-27] which then would use the `sessionId` from the established connection to identify incoming payloads that need to be respond to, and also which `chainId` is being targetted.

#### UUIDs

The generation of UUIDs is crucial for this messaging interface to work seamlessly for the users.

A Wallet Provider MUST always generate UUIDs distinctly for each web page loaded, and they must not be re-used without a session being established between the application and the wallet with the user's consent.

A UUID can be re-used as a `sessionId` if and only if the [CAIP-25][caip-25] procedure has been prompted to the user and the user has approved its permissions to allow the application to make future signing requests.

Once established, the UUID is used as `sessionId` for the [CAIP-27][caip-27] payloads, which can verify that incoming messages are being routed through the appropriate channels.

## Rationale

Browser wallets differentiate themselves because they can be installed by users without the application developer requiring any further integration. Therefore, we optimize for a messaging interface that leverages the two-way communication available to browser wallets to make themselves discoverable, and negotiate a set of parameters that enable not only easy human readability with a clear name and icon but also machine-readability using strong identifiers with uuids and rdns.

The choice for using `window.postMessage` is motivated by expanding the range of Wallet Providers it can support, including browser extensions that can alternatively use `window.dispatchEvent` but instead it would also cover Inline Frames, Service Workers, Shared Workers, and more.

The use of UUID for message routing is important because while RDNS is useful for identifying the Wallet Provider, it causes issues when it comes to the session management of different webpages connected to the same Wallet Provider or even managing stale sessions, which can be out-of-sync. Since UUID generation is derived dynamically on page load, Wallet Providers can track these sessions more granularly rather than making assumptions around the webpage URL and RDNS relationship.

The existing standards around wallet session creation (CAIP-25) are fundamental to this experience because they create clear intents for a wallet to "connect" with a webpage url after it's been discovered. This standard does not enforce either one but strongly recommends these standards as the preferred interface for connecting or authenticating a wallet.

Finally the use of CAIP-27 leverages the work above to properly target signing requests that are intended to be prompt to wallet users which will include a `sessionId` and `chainId` in parallel with the pre-established sessions using either CAIP-25.

## Test Cases

Here is a test case where we demonstrate a scenario with logic from both a Blockchain Library and a Wallet Provider.

Logic from the Blockchain Library:

```typescript
// 1. Blockchain Library initializes by listening to wallet_announce messages and
// also by posting a prompt message
interface WalletMapEntry {
  data: WalletAnnounceRequestParams;
  targetOrigin: string;
}

const wallets: Record<string, WalletMapEntry> = {}

window.addEventListener("message", (event) => {
  if (event.data.method === 'wallet_announce') {
    // when an announce message was received then the library can index it by uuid
    wallets[event.data.params.uuid] = {
      params: event.data.params,
      targetOrigin: event.targetOrigin
    }
  }
});

window.postMessage({
  id: 1,
  jsonrpc: "2.0"
  method: "wallet_prompt",
  params: {},
});

// 2. User presses "Connect Wallet" and the library display the discovered wallets

const selected_uuid = "350670db-19fa-4704-a166-e52e178b59d2"

// 3. User selects a Wallet with UUID = "350670db-19fa-4704-a166-e52e178b59d2" and
// Blockchain Library will send a CAIP-25 request to establish a wallet connection

const sessionRequest = {
  id: 123,
  jsonrpc: "2.0",
  method: "wallet_createSession",
  params: {
    optionalScopes: {
      eip155: {
        scopes: ["chain:777"],
        methods: ["chain_signMessage", "chain_sendTransaction"],
        notifications: ["accountsChanged"],
      },
    },
    sessionProperties: {
      expiry: "2024-06-06T13:10:48.155Z",
    },
  },
};

let sessionResult = {}

window.addEventListener("message", (event) => {
  if (event.targetOrigin !== wallets[selected_uuid].targetOrigin) return;
  if (event.data.id === sessionRequest.id) {
    // Get JSON-RPC response
    if (event.data.error) {
      console.error(event.data.error.message);
    } else {
      sessionResult = event.data.result
    }
  }
});


window.postMessage(sessionRequest, wallets[selected_uuid].targetOrigin);


// 4. After the response was received by the Blockchain Library from the wallet
// provider then the session is established with a sessionId matchin the UUID
// thus signing requests can be using a CAIP-27 request to the wallet user
const signingRequest = {
  id: 456,
  jsonrpc: "2.0",
  method: "wallet_requestMethod",
  params: {
    // UUID from WalletData is used as SessionId
    sessionId: walletData.uuid,
    scope: "chain:777",
    request: {
      method: "chain_signMessage",
      params: [
        "Verifying my wallet with this message",
        "0xa89Df33a6f26c29ea23A9Ff582E865C03132b140",
      ],
    },
  },
};

let signingResult = {}

window.addEventListener("message", (event) => {
  if (event.targetOrigin !== wallets[selected_uuid].targetOrigin) return;
  if (event.data.id === signingRequest.id) {
    // Get JSON-RPC response
    if (event.data.error) {
      console.error(event.data.error.message);
    } else {
      signingResult = event.data.result
    }
  }
});

window.postMessage(signingRequest, wallets[selected_uuid].targetOrigin);
```

Logic from the Wallet Provider:

```typescript
// 1. Wallet Provider sets their WalletData and then listens to prompt_wallet message
// and also immediatelly posts a message with the WalletData as wallet_announce type
const walletData = {
  uuid: generateUUID(); // eg. "350670db-19fa-4704-a166-e52e178b59d2"
  name: "Example Wallet",
  icon: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==",
  rdns: "com.example.wallet",
}

window.postMessage({
  id: 2,
  jsonrpc: "2.0"
  method: "wallet_announce",
  params: walletData
});

window.addEventListener("message", (event) => {
  if (event.data.method === 'wallet_prompt') {
    // when a prompt message was received then the wallet will announces again
    window.postMessage({
      id: 2,
      jsonrpc: "2.0"
      method: "wallet_announce",
      params: walletData
    });
  }
});

// 2. User presses "Connect Wallet" on the application webpage which will select UUID

const selected_uuid = "350670db-19fa-4704-a166-e52e178b59d2"

// 3. Wallet Provider receives a CAIP-25 request to establish a wallet connection
// prompts the user to approve and once its approved it can respond back to app
// Wallet Provider listens for request
let sessionRequest = {}
let sessionOrigin = ""

window.addEventListener("message", (event) => {
  if (event.data.method === "wallet_createSession") {
    sessionRequest = event.data
    // if incoming requests match the WalletData UUID
    if (checkSupportedScopes(event.data.params)) {
        // prompt user to approve session
        // persist the targetOrigin for sessionRequest
        sessionOrigin = event.targetOrigin
    }
  }
});

const sessionResponse = {
  id: sessionRequest.id, // 123
  jsonrpc: "2.0",
  result: {
    sessionId: walletData.uuid, // "350670db-19fa-4704-a166-e52e178b59d2"
    sessionScopes: {
      eip155: {
        scopes: ["eip155:1", "eip155:10"],
        methods: ["eth_sendTransaction", "personal_sign"],
        notifications: ["accountsChanged", "chainChanged"],
        accounts: [
          "eip155:1:0x43e3ca49c7be4f429abce408da6b738f879d02a0",
          "eip155:10:0x43e3ca49c7be4f429abce408da6b738f879d02a0"
        ]
      },
    },
    sessionProperties: {
      expiry: "2024-06-06T13:10:48.155Z",
    }
  }
}

window.postMessage(sessionResponse, sessionOrigin);

// 4. Once the connection is established then the Wallet Provider can receive
// incoming CAIP-27 requests which will be prompted to the user to sign and
// once signed the response is sent back to the dapp with the expected result
let signingRequest = {}

window.addEventListener("message", (event) => {
	if (event.targetOrigin !== sessionOrigin) return;
  if (event.data.method === "wallet_createSession" && event.data.params.sessionId === walletData.uuid) {
    signingRequest = event.data.params
  }
});

const signingResponse = {
  id: signingRequest.id // 456
  jsonrpc: "2.0",
  result: "0xe670ec64341771606e55d6b4ca35a1a6b75ee3d5145a99d05921026d1527331"
}

window.postMessage(signingResponse, sessionOrigin);
```

## Security Considerations

TODO

## Privacy Considerations

TODO

## Backwards Compatibility

TODO

## Links

- [EIP-6963][eip-6963] - Multi Injected Provider Discovery
- [CAIP-27][caip-27] - Blockchain ID Specification
- [CAIP-25][caip-25] - Blockchain ID Specification
- [CAIP-282][caip-282] - Browser Wallet Discovery Interface

[eip-6963]: https://eips.ethereum.org/EIPS/eip-6963
[caip-27]: https://chainagnostic.org/CAIPs/caip-27
[caip-25]: https://chainagnostic.org/CAIPs/caip-25
[caip-282]: https://chainagnostic.org/CAIPs/caip-282

## Copyright

Copyright and related rights waived via [CC0](../LICENSE).