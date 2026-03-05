<div align="center">

# @decentralchain/browser-bus

**Secure cross-window browser communication for the DecentralChain ecosystem**

[![CI](https://github.com/Decentral-America/browser-bus/actions/workflows/ci.yml/badge.svg)](https://github.com/Decentral-America/browser-bus/actions/workflows/ci.yml)
[![npm version](https://img.shields.io/npm/v/@decentralchain/browser-bus)](https://www.npmjs.com/package/@decentralchain/browser-bus)
[![license](https://img.shields.io/npm/l/@decentralchain/browser-bus)](./LICENSE)
[![Node.js](https://img.shields.io/node/v/@decentralchain/browser-bus)](./package.json)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.9-blue.svg)](https://www.typescriptlang.org/)
[![Bundle Size](https://img.shields.io/bundlephobia/minzip/@decentralchain/browser-bus)](https://bundlephobia.com/package/@decentralchain/browser-bus)
[![ESM Only](https://img.shields.io/badge/Module-ESM%20Only-F7DF1E.svg)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)

[Installation](#installation) · [Quick Start](#quick-start) · [API Reference](#api-reference) · [Architecture](#architecture) · [Security](#security-model) · [Contributing](#contributing)

</div>

---

## Overview

**`@decentralchain/browser-bus`** is a production-grade, type-safe messaging library that enables secure communication between browser windows, tabs, and iframes using the native [`postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) API. It is purpose-built for the **DecentralChain** blockchain ecosystem, powering interactions between decentralized applications (DApps), wallet extensions, and signing interfaces.

In the DecentralChain ecosystem, DApps frequently need to communicate with wallet applications that run in separate browser contexts — whether as popup windows, embedded iframes, or browser extensions. **browser-bus** provides the structured messaging layer that makes this seamless, handling origin validation, request/response patterns, and channel isolation out of the box.

### Key Features

| Feature                          | Description                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------- |
| 🔒 **Origin-Based Security**     | Whitelist trusted origins to prevent unauthorized cross-origin message injection |
| 📡 **Event & Request/Response**  | Both fire-and-forget events and Promise-based request/response patterns          |
| 🏷️ **Channel Isolation**         | Multiplex independent message channels over a single window connection           |
| 🛡️ **Runtime Schema Validation** | Validates every incoming message against the protocol schema before processing   |
| 📦 **Tree-Shakeable ESM**        | Pure ES Module output with full tree-shaking support and minimal bundle impact   |
| 🔧 **TypeScript-First**          | Complete generic type safety for events, requests, and responses                 |
| ⚡ **Zero Dependencies**         | No runtime dependencies — only the browser `postMessage` API                     |
| 🔌 **Pluggable Adapters**        | Swap transport layers via the abstract `Adapter` interface                       |

### How It Works with DecentralChain

DecentralChain is a blockchain platform designed for building decentralized applications. When users interact with DecentralChain DApps in their browser, several cross-window communication scenarios arise:

1. **DApp ↔ Wallet Communication** — A DApp running in the main browser tab opens a wallet popup (or communicates with a wallet iframe) to request transaction signing. The bus carries the transaction payload to the wallet and returns the signed transaction back to the DApp.

2. **Transaction Signing Popups** — When a DApp needs to sign a transaction, it spawns a popup window containing the wallet UI. The popup receives the unsigned transaction via `browser-bus`, presents it for user approval, signs it with the user's private key, and sends the signed result back through the bus.

3. **Multi-Tab Synchronization** — Multiple browser tabs running the same DApp can coordinate state, share authentication tokens, or broadcast real-time blockchain events through a shared iframe relay.

4. **Embedded DApp Interfaces** — Third-party DApps can be embedded as iframes within a host platform, with `browser-bus` providing the secure messaging bridge between the host and embedded application.

```
┌──────────────────────────┐     postMessage      ┌──────────────────────────┐
│     DecentralChain       │◄────────────────────►│     Wallet Extension     │
│         DApp             │    (browser-bus)      │     / Popup Window       │
│                          │                       │                          │
│  • Build transaction     │  ── request ──────►   │  • Validate transaction  │
│  • Send for signing      │                       │  • Prompt user approval  │
│  • Receive signed tx     │  ◄── response ─────   │  • Sign with private key │
│  • Broadcast to chain    │                       │  • Return signed tx      │
└──────────────────────────┘                       └──────────────────────────┘
```

## Requirements

- **Node.js** >= 24
- **npm** >= 10

## Installation

```bash
npm install @decentralchain/browser-bus
```

## Quick Start

### Parent window with iframe

```typescript
import { Bus, WindowAdapter } from '@decentralchain/browser-bus';

const url = 'https://some-iframe-content-url.com';
const iframe = document.createElement('iframe');

WindowAdapter.createSimpleWindowAdapter(iframe).then((adapter) => {
  const bus = new Bus(adapter);

  bus.once('ready', () => {
    // Received message from iframe
  });
});
iframe.src = url;
document.body.appendChild(iframe);
```

### Iframe side

```typescript
import { Bus, WindowAdapter } from '@decentralchain/browser-bus';

WindowAdapter.createSimpleWindowAdapter().then((adapter) => {
  const bus = new Bus(adapter);

  bus.dispatchEvent('ready', null);
});
```

### DApp-to-Wallet Transaction Signing

A common DecentralChain pattern: the DApp requests the wallet to sign a transaction.

**DApp side (main window):**

```typescript
import { Bus, WindowAdapter } from '@decentralchain/browser-bus';

// Open wallet popup
const walletPopup = window.open('https://wallet.decentralchain.io/sign', 'wallet');

WindowAdapter.createSimpleWindowAdapter(walletPopup, {
  origins: ['https://wallet.decentralchain.io'],
}).then((adapter) => {
  const bus = new Bus(adapter);

  // Request the wallet to sign a transaction
  bus
    .request('sign-transaction', {
      type: 'transfer',
      recipient: '3N...',
      amount: 100000000,
      assetId: 'DCC',
    })
    .then((signedTx) => {
      console.log('Signed transaction:', signedTx);
      // Broadcast signedTx to the DecentralChain network
    });
});
```

**Wallet side (popup):**

```typescript
import { Bus, WindowAdapter } from '@decentralchain/browser-bus';

WindowAdapter.createSimpleWindowAdapter().then((adapter) => {
  const bus = new Bus(adapter);

  bus.registerRequestHandler('sign-transaction', async (txData) => {
    // Present transaction to user for approval
    const approved = await showApprovalDialog(txData);
    if (!approved) throw new Error('User rejected transaction');

    // Sign with user's private key
    return signTransaction(txData, userPrivateKey);
  });
});
```

### Multi-Tab State Synchronization

Coordinate state across multiple tabs through a shared iframe relay:

```typescript
import { Bus, WindowAdapter } from '@decentralchain/browser-bus';

// Each tab connects to a shared hidden iframe
const relay = document.getElementById('relay-iframe') as HTMLIFrameElement;

WindowAdapter.createSimpleWindowAdapter(relay).then((adapter) => {
  const bus = new Bus(adapter);

  // Listen for balance updates from other tabs
  bus.on('balance-updated', (data) => {
    updateBalanceDisplay(data.balance);
  });

  // Broadcast state changes to all tabs
  bus.dispatchEvent('balance-updated', { balance: newBalance });
});
```

---

## Architecture

`browser-bus` is built around a layered architecture that separates concerns and enables extensibility:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Application Layer                         │
│  Your DApp / Wallet code using Bus.dispatchEvent() & .request() │
├──────────────────────────────────────────────────────────────────┤
│                       Bus (Message Bus)                          │
│  • Event dispatch (fire-and-forget)                              │
│  • Request/response (Promise-based)                              │
│  • Handler registration & lifecycle                              │
├──────────────────────────────────────────────────────────────────┤
│                     Adapter (Transport)                          │
│  • WindowAdapter (postMessage)                                   │
│  • Origin validation & channel filtering                         │
│  • Runtime message schema validation                             │
├──────────────────────────────────────────────────────────────────┤
│                  WindowProtocol (Protocol)                       │
│  • LISTEN mode (addEventListener)                                │
│  • DISPATCH mode (postMessage)                                   │
│  • Low-level browser API wrapper                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Message Flow

1. **Sending:** Your code calls `bus.dispatchEvent()` or `bus.request()` → the `Bus` constructs a typed message → the `Adapter` attaches a channel ID and passes it to the `WindowProtocol` → `postMessage()` delivers it to the target window.

2. **Receiving:** The browser fires a `message` event → `WindowProtocol` captures it → `WindowAdapter` validates the origin, channel, and schema → the `Bus` routes it to the appropriate event handler or resolves the pending request Promise.

### Message Types

| Type         | Enum                     | Purpose                                |
| ------------ | ------------------------ | -------------------------------------- |
| **Event**    | `EventType.Event (0)`    | One-way notification — fire and forget |
| **Action**   | `EventType.Action (1)`   | Request that expects a response        |
| **Response** | `EventType.Response (2)` | Reply to a previous Action             |

---

## API Reference

### `Bus`

Creates a bus instance for sending and receiving events and requests. The `Bus` class is generic, supporting typed event maps and handler maps for full type safety.

**Constructor:**

```typescript
const bus = new Bus(adapter, 5000); // optional timeout in ms (default: 5000)
```

- `adapter` — an `Adapter` instance for the messaging transport
- `timeout` (optional) — default response timeout in milliseconds (default: 5000)

#### `dispatchEvent(name, data)`

Send an event to all connected Bus instances.

```typescript
bus.dispatchEvent('some-event-name', jsonLikeData);
```

#### `request(name, data?, timeout?)`

Send a request and receive a response. Returns a `Promise`.

```typescript
const result = await bus.request('some-method', jsonLikeData, 100);
```

#### `on(name, handler)`

Subscribe to an event.

```typescript
bus.on('some-event', (data) => {
  // handle event
});
```

#### `once(name, handler)`

Subscribe to an event once.

```typescript
bus.once('some-event', (data) => {
  // fires only once
});
```

#### `off(eventName?, handler?)`

Unsubscribe from events.

```typescript
bus.off('some-event', handler); // Unsubscribe specific handler
bus.off('some-event'); // Unsubscribe all from 'some-event'
bus.off(); // Unsubscribe from everything
```

#### `registerRequestHandler(name, handler)`

Register a handler for incoming requests. Handlers can be synchronous or asynchronous.

```typescript
// Synchronous handler
bus.registerRequestHandler('get-random', () => Math.random());

// Asynchronous handler
bus.registerRequestHandler('get-data', async () => {
  const data = await fetchData();
  return data;
});
```

Handlers may return Promises:

```typescript
bus.registerRequestHandler('get-data', () => Promise.resolve(someData));
```

#### `unregisterHandler(name)`

Remove a previously registered request handler.

```typescript
bus.unregisterHandler('get-random');
```

#### `changeAdapter(adapter)`

Create a new Bus instance with the same event and request handlers but a different adapter transport. Useful for reconnecting to a different window without re-registering handlers.

```typescript
const newBus = bus.changeAdapter(newAdapter);
```

#### `destroy()`

Destroy the bus instance and its adapter. All pending requests are rejected, all event handlers are removed, and the adapter is cleaned up.

```typescript
bus.destroy();
```

### `WindowAdapter`

Adapter implementation for cross-window communication via the browser `postMessage` API. Handles origin validation, channel filtering, and runtime message schema validation.

#### `WindowAdapter.createSimpleWindowAdapter(iframe?, options?)`

Factory method that creates a `WindowAdapter` for simple parent/iframe or parent/popup communication. Automatically detects the target origin from the iframe `src` or `document.referrer`.

```typescript
// Parent window — connect to an iframe
const adapter = await WindowAdapter.createSimpleWindowAdapter(iframeElement, {
  origins: ['https://trusted-domain.com'],
  availableChannelId: ['my-channel'],
  channelId: 'my-channel',
});

// Child window (iframe or popup) — connect back to parent
const adapter = await WindowAdapter.createSimpleWindowAdapter();
```

**Options:**

| Option               | Type                        | Description                                                            |
| -------------------- | --------------------------- | ---------------------------------------------------------------------- |
| `origins`            | `string \| string[]`        | Whitelist of trusted origins (in addition to `window.location.origin`) |
| `availableChannelId` | `string \| number \| Array` | Channel IDs to accept (empty = accept all)                             |
| `channelId`          | `string \| number`          | Channel ID to attach to outgoing messages                              |

#### Constructor (Advanced)

For complex multi-window topologies, you can construct a `WindowAdapter` directly with multiple listen/dispatch protocols:

```typescript
const adapter = new WindowAdapter(
  [listenProtocol1, listenProtocol2], // listen from multiple sources
  [dispatchProtocol1], // dispatch to one or more targets
  options,
);
```

### `WindowProtocol`

Low-level wrapper around the browser `postMessage` and `addEventListener` APIs. Operates in two modes:

- **LISTEN** — Attaches a `message` event listener to a window
- **DISPATCH** — Sends messages via `postMessage` to a target window

```typescript
import { WindowProtocol } from '@decentralchain/browser-bus';

// Listen for incoming messages
const listener = new WindowProtocol(window, WindowProtocol.PROTOCOL_TYPES.LISTEN);

// Dispatch messages to a target window
const dispatcher = new WindowProtocol(targetWindow, WindowProtocol.PROTOCOL_TYPES.DISPATCH, origin);
```

### `Adapter`

Abstract base class for custom transport implementations. Extend this class to create adapters for non-`postMessage` transports (e.g., WebSocket, BroadcastChannel, or custom protocols).

```typescript
import { Adapter } from '@decentralchain/browser-bus';

class MyCustomAdapter extends Adapter {
  send(data) {
    /* send message via your transport */
  }
  addListener(cb) {
    /* register incoming message callback */
  }
  destroy() {
    /* cleanup resources */
  }
}
```

### Exported Types

```typescript
import type {
  IOneArgFunction, // Single-argument function type
  TMessageContent, // Union of all message types
  TChannelId, // string | number
  IEventData, // Event message shape
  IRequestData, // Request message shape
  IResponseData, // Response message shape
} from '@decentralchain/browser-bus';
```

### Configuration

Control logging verbosity via the `config` namespace:

```typescript
import { config } from '@decentralchain/browser-bus';

config.console.logLevel = 'production'; // 'production' | 'errors' | 'verbose'
```

---

## Security Model

`browser-bus` implements defense-in-depth security for cross-window communication:

### Origin Validation

Every incoming message is checked against a whitelist of trusted origins. Messages from untrusted origins are silently dropped, preventing cross-origin injection attacks.

```typescript
// Only accept messages from these origins
const adapter = await WindowAdapter.createSimpleWindowAdapter(iframe, {
  origins: ['https://wallet.decentralchain.io', 'https://app.decentralchain.io'],
});
```

### Channel Isolation

Multiple independent communication channels can be multiplexed over a single window connection. Messages are filtered by channel ID, ensuring that unrelated message flows don't interfere with each other.

### Runtime Schema Validation

Every incoming `postMessage` is validated against the expected protocol schema before being processed. Messages with invalid types, missing required fields, or malformed structures are rejected.

### Error Isolation

Error messages sent across windows intentionally strip stack traces to prevent leaking internal code paths to potentially untrusted contexts. Only the error message and name are transmitted.

### Best Practices

- Always specify explicit `origins` rather than relying on the wildcard `*`
- Use unique `channelId` values when running multiple bus instances
- Set appropriate `timeout` values for request/response patterns
- Call `bus.destroy()` when communication is no longer needed

---

## Browser Compatibility

`browser-bus` uses standard Web APIs supported by all modern browsers:

| API                                                                                                 | Chrome | Firefox | Safari | Edge   |
| --------------------------------------------------------------------------------------------------- | ------ | ------- | ------ | ------ |
| [`postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)                | ✅     | ✅      | ✅     | ✅     |
| [`addEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener) | ✅     | ✅      | ✅     | ✅     |
| [ES Modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)                 | ✅ 61+ | ✅ 60+  | ✅ 11+ | ✅ 16+ |

---

## Development

### Prerequisites

- **Node.js** >= 24 (see `.node-version`)
- **npm** >= 10

### Setup

```bash
git clone https://github.com/Decentral-America/browser-bus.git
cd browser-bus
npm install
```

### Scripts

| Command                     | Description                              |
| --------------------------- | ---------------------------------------- |
| `npm run build`             | Build distribution files                 |
| `npm test`                  | Run tests with Vitest                    |
| `npm run test:watch`        | Tests in watch mode                      |
| `npm run test:coverage`     | Tests with V8 coverage                   |
| `npm run typecheck`         | TypeScript type checking                 |
| `npm run lint`              | ESLint                                   |
| `npm run lint:fix`          | ESLint with auto-fix                     |
| `npm run format`            | Format with Prettier                     |
| `npm run validate`          | Full CI validation pipeline              |
| `npm run bulletproof`       | Format + lint fix + typecheck + test     |
| `npm run bulletproof:check` | CI-safe: check format + lint + tc + test |

### Quality Gates

- ESLint with strict TypeScript rules
- Prettier formatting
- 90%+ code coverage thresholds
- Bundle size budget enforcement
- Package export validation (publint + attw)

## Contributing

We welcome contributions from the community! See [CONTRIBUTING.md](./CONTRIBUTING.md) for development guidelines, coding standards, and the pull request process.

## Security

Found a vulnerability? Please report it responsibly. See [SECURITY.md](./SECURITY.md) for our security policy and vulnerability reporting process.

## Code of Conduct

This project follows the Contributor Covenant. See [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md).

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for a detailed list of changes in each release.

## Related Packages

This library is part of the **DecentralChain** JavaScript/TypeScript SDK ecosystem:

| Package                         | Description                                       |
| ------------------------------- | ------------------------------------------------- |
| **@decentralchain/browser-bus** | Cross-window browser communication (this package) |

## License

[MIT](./LICENSE) — Copyright (c) 2026-present DecentralChain

---

<div align="center">

Built with ❤️ by [Decentral America](https://github.com/Decentral-America) for the DecentralChain community

</div>
