# Lightning Native Messaging Protocol (LNMP) - Complete Specification

## Executive Summary

LNMP is a decentralized, encrypted messaging protocol that uses Bitcoin's Lightning Network for native payments and spam prevention. It replaces email's flawed architecture with cryptographic identities, economic incentives, and true user sovereignty. Built on Bitcoin principles: no central authority, cryptographic proof over trust, and aligned economic incentives for all participants.

## Table of Contents

1. [Core Philosophy & Principles](#core-philosophy--principles)
2. [Protocol Architecture](#protocol-architecture)
3. [Technical Components](#technical-components)
4. [Network Topology](#network-topology)
5. [Economic Model](#economic-model)
6. [Implementation Specification](#implementation-specification)
7. [User Stories](#user-stories)
8. [Advantages Over Email](#advantages-over-email)
9. [Security Model](#security-model)
10. [Development Roadmap](#development-roadmap)

---

## Core Philosophy & Principles

### Bitcoin Alignment
- **No Trusted Third Parties**: Every component can be self-hosted
- **Cryptographic Proof**: All messages cryptographically signed
- **Economic Incentives**: Spam costs money, quality content earns
- **Permissionless**: Anyone can run infrastructure
- **Censorship Resistant**: No single point of control
- **Open Source**: Entire stack freely available

### Design Principles
1. **User Sovereignty**: Users own their identity (private keys)
2. **Privacy First**: End-to-end encryption by default
3. **Economic Spam Prevention**: Make spam economically unfeasible
4. **Decentralized Architecture**: No single point of failure
5. **Backwards Compatible**: Bridge to legacy email/messaging

---

## Protocol Architecture

### Identity Layer

```typescript
interface LNMPIdentity {
  // Core identity
  publicKey: string;           // secp256k1 public key
  lightningNode?: string;      // Optional: separate LN node pubkey

  // Human-readable addressing
  addresses: LNMPAddress[];    // Multiple addresses per identity

  // Capabilities
  features: Feature[];         // Supported message types
  relays: RelayInfo[];        // Preferred relays

  // Economic preferences
  pricing: PricingPolicy;      // How much to charge/accept

  // Cryptographic proofs
  proofs: IdentityProof[];     // Domain ownership, social attestations
}

interface LNMPAddress {
  username: string;            // Human-chosen name
  checksum: string;           // 6-char pubkey derivative
  relay?: string;             // Optional relay hint
  expiry?: number;            // Optional expiration

  toString(): string {
    // Returns: alice#a3f2b9@relay.com
  }
}
```

### Message Structure

```typescript
interface LNMPMessage {
  // Metadata (unencrypted)
  id: string;                 // Unique message ID
  from: string;               // Sender pubkey
  to: string[];               // Recipient pubkeys
  timestamp: number;          // Unix timestamp

  // Payment proof
  payment?: {
    preimage: string;         // Lightning payment proof
    amount: number;           // Satoshis paid
    route?: string;          // Payment routing hints
  };

  // Encrypted payload
  encrypted: {
    algorithm: 'ChaCha20-Poly1305' | 'AES-256-GCM';
    nonce: string;
    ciphertext: string;       // Encrypted MessageContent
  };

  // Delivery hints
  relays: string[];          // Suggested relay servers
  expiry?: number;           // Message expiration
  priority: Priority;        // Based on payment amount

  // Signature
  signature: string;         // Schnorr signature
}

interface MessageContent {
  // Message data (encrypted)
  type: 'text' | 'voice' | 'file' | 'payment' | 'receipt';
  subject?: string;
  body: string;

  // Threading
  threadId?: string;
  replyTo?: string;

  // Attachments
  attachments?: FileAttachment[];

  // Rich features
  formatting?: 'plain' | 'markdown' | 'html';
  mentions?: string[];       // Pubkeys mentioned
  reactions?: Reaction[];

  // Lifecycle
  editHistory?: Edit[];
  deleteAfter?: number;      // Auto-delete timestamp
  readReceipts?: boolean;
}

interface FileAttachment {
  ipfsHash: string;          // IPFS content ID
  filename: string;
  size: number;
  mimeType: string;

  // Encryption
  encryptionKey: string;     // Symmetric key for file
  algorithm: string;

  // Payment
  price?: number;           // Sats to download

  // Validation
  checksums: {
    sha256: string;
  };
}
```

### Relay Protocol

```typescript
interface RelayServer {
  // Server info
  publicKey: string;         // Relay's identity
  endpoint: string;          // wss://relay.example.com

  // Capabilities
  features: RelayFeature[];
  limits: RelayLimits;

  // Economic model
  pricing: RelayPricing;

  // Statistics
  stats: RelayStats;
}

interface RelayProtocol {
  // Client → Relay
  SUBSCRIBE: {
    publicKey: string;
    signature: string;       // Prove key ownership
    filters?: Filter[];      // What messages to receive
  };

  PUBLISH: {
    message: LNMPMessage;
    targetRelays?: string[]; // Specific relay routing
  };

  QUERY: {
    filter: QueryFilter;
    limit?: number;
    payment?: string;        // Payment for premium queries
  };

  // Relay → Client
  MESSAGE: {
    subscription: string;
    message: LNMPMessage;
  };

  NOTICE: {
    message: string;
    severity: 'info' | 'warning' | 'error';
  };

  INVOICE: {
    amount: number;
    description: string;
    bolt11: string;
  };
}
```

---

## Technical Components

### 1. Identity Service

```typescript
// src/identity/IdentityManager.ts
export class IdentityManager {
  private privateKey: PrivateKey;
  private publicKey: PublicKey;
  private lightning: LightningService;

  constructor(seed?: string) {
    // HD wallet derivation for identity
    const hdNode = HDNode.fromSeed(seed || randomBytes(32));
    this.privateKey = hdNode.derivePath("m/138'/0'/0'"); // BIP138 for messaging
    this.publicKey = this.privateKey.publicKey;
  }

  async generateAddress(username: string, relay?: string): Promise<LNMPAddress> {
    // Generate deterministic checksum
    const checksum = this.calculateChecksum(username);

    return {
      username,
      checksum,
      relay,
      toString: () => `${username}#${checksum}${relay ? '@' + relay : ''}`
    };
  }

  private calculateChecksum(username: string): string {
    // Deterministic checksum from pubkey + username
    const hash = sha256(this.publicKey.toString() + username);
    return base58.encode(hash).substring(0, 6);
  }

  async sign(message: LNMPMessage): Promise<string> {
    const serialized = this.serializeForSigning(message);
    return schnorr.sign(serialized, this.privateKey);
  }

  async createIdentityProof(domain: string): Promise<IdentityProof> {
    // Prove domain ownership
    const challenge = `lnmp-verify:${this.publicKey}:${Date.now()}`;
    const signature = await this.sign(challenge);

    return {
      type: 'domain',
      domain,
      publicKey: this.publicKey.toString(),
      challenge,
      signature,
      timestamp: Date.now()
    };
  }
}
```

### 2. Encryption Service

```typescript
// src/crypto/EncryptionService.ts
export class EncryptionService {
  // Use Signal Protocol's Double Ratchet for forward secrecy
  private sessions: Map<string, DoubleRatchetSession> = new Map();

  async encryptMessage(
    content: MessageContent,
    recipientPubKeys: string[],
    senderPrivKey: PrivateKey
  ): Promise<EncryptedPayload> {
    // Generate ephemeral key for this message
    const ephemeralKey = PrivateKey.random();

    // For each recipient, create encrypted payload
    const encryptedCopies = await Promise.all(
      recipientPubKeys.map(async (pubKey) => {
        const sharedSecret = await this.deriveSharedSecret(
          ephemeralKey,
          PublicKey.fromString(pubKey)
        );

        return this.encryptWithSharedSecret(content, sharedSecret);
      })
    );

    return {
      ephemeralPubKey: ephemeralKey.publicKey.toString(),
      encryptedCopies,
      algorithm: 'ChaCha20-Poly1305'
    };
  }

  async decryptMessage(
    encrypted: EncryptedPayload,
    recipientPrivKey: PrivateKey
  ): Promise<MessageContent> {
    const ephemeralPubKey = PublicKey.fromString(encrypted.ephemeralPubKey);
    const sharedSecret = await this.deriveSharedSecret(
      recipientPrivKey,
      ephemeralPubKey
    );

    // Find our encrypted copy and decrypt
    const ourCopy = encrypted.encryptedCopies.find(copy =>
      this.canDecrypt(copy, sharedSecret)
    );

    if (!ourCopy) {
      throw new Error('Message not encrypted for us');
    }

    return this.decryptWithSharedSecret(ourCopy, sharedSecret);
  }

  private async deriveSharedSecret(
    privateKey: PrivateKey,
    publicKey: PublicKey
  ): Promise<Buffer> {
    // ECDH key agreement
    return ecdh(privateKey, publicKey);
  }
}
```

### 3. Lightning Payment Service

```typescript
// src/lightning/LightningService.ts
export class LightningService {
  private node: LightningNode;

  constructor(config: LightningConfig) {
    this.node = new LightningNode(config);
  }

  async createMessageInvoice(
    amount: number,
    metadata: MessageMetadata
  ): Promise<MessageInvoice> {
    // Create HODL invoice that only settles on message acceptance
    const preimageHash = sha256(randomBytes(32));

    const invoice = await this.node.createHodlInvoice({
      amountSats: amount,
      hash: preimageHash,
      memo: `LNMP message to ${metadata.recipient}`,
      expiry: 3600, // 1 hour

      // Include message ID in invoice metadata
      descriptionHash: sha256(metadata.messageId)
    });

    return {
      bolt11: invoice.bolt11,
      preimageHash,
      messageId: metadata.messageId,

      // Methods for settling/canceling
      async accept() {
        await this.node.settleInvoice(preimageHash);
      },

      async reject() {
        await this.node.cancelInvoice(preimageHash);
      }
    };
  }

  async streamPayments(
    recipient: string,
    amountPerSecond: number
  ): Promise<PaymentStream> {
    // For real-time communication (voice/video)
    const stream = new PaymentStream({
      destination: recipient,
      amountPerSecond,
      interval: 1000 // Pay every second
    });

    stream.on('tick', async () => {
      await this.sendKeysend(recipient, amountPerSecond);
    });

    return stream;
  }

  async verifyPayment(payment: PaymentProof): Promise<boolean> {
    // Verify the payment preimage matches the hash
    const hash = sha256(Buffer.from(payment.preimage, 'hex'));
    return hash.equals(Buffer.from(payment.hash, 'hex'));
  }
}
```

### 4. Relay Client

```typescript
// src/relay/RelayClient.ts
export class RelayClient extends EventEmitter {
  private ws: WebSocket;
  private subscriptions: Map<string, Subscription> = new Map();
  private messageQueue: MessageQueue;

  constructor(private relay: RelayServer, private identity: Identity) {
    super();
    this.messageQueue = new MessageQueue();
  }

  async connect(): Promise<void> {
    this.ws = new WebSocket(this.relay.endpoint);

    this.ws.on('open', async () => {
      // Authenticate with relay
      await this.authenticate();

      // Resume subscriptions
      for (const sub of this.subscriptions.values()) {
        await this.subscribe(sub);
      }
    });

    this.ws.on('message', (data) => {
      this.handleRelayMessage(JSON.parse(data));
    });

    // Automatic reconnection
    this.ws.on('close', () => {
      setTimeout(() => this.connect(), 5000);
    });
  }

  async sendMessage(message: LNMPMessage): Promise<void> {
    // Queue if offline, send immediately if connected
    if (this.ws.readyState === WebSocket.OPEN) {
      await this.publish(message);
    } else {
      await this.messageQueue.add(message);
    }
  }

  private async publish(message: LNMPMessage): Promise<void> {
    // Sign the message
    const signature = await this.identity.sign(message);
    message.signature = signature;

    // Send to relay
    this.ws.send(JSON.stringify({
      type: 'PUBLISH',
      message
    }));

    // Wait for acknowledgment
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error('Publish timeout'));
      }, 10000);

      this.once(`ack:${message.id}`, () => {
        clearTimeout(timeout);
        resolve();
      });
    });
  }

  async queryMessages(filter: QueryFilter): Promise<LNMPMessage[]> {
    // Query historical messages
    const queryId = randomBytes(16).toString('hex');

    this.ws.send(JSON.stringify({
      type: 'QUERY',
      id: queryId,
      filter
    }));

    return new Promise((resolve) => {
      const messages: LNMPMessage[] = [];

      this.on(`query:${queryId}:message`, (msg) => {
        messages.push(msg);
      });

      this.once(`query:${queryId}:end`, () => {
        resolve(messages);
      });
    });
  }
}
```

### 5. IPFS Integration

```typescript
// src/storage/IPFSService.ts
export class IPFSService {
  private ipfs: IPFS;
  private encryption: EncryptionService;

  constructor() {
    this.ipfs = await IPFS.create({
      repo: './ipfs-repo',
      config: {
        Bootstrap: [...defaultBootstrapNodes, ...lnmpBootstrapNodes]
      }
    });
  }

  async uploadFile(
    file: Buffer,
    metadata: FileMetadata,
    recipientPubKeys: string[]
  ): Promise<FileAttachment> {
    // Generate encryption key
    const encryptionKey = randomBytes(32);

    // Encrypt file
    const encrypted = await this.encryptFile(file, encryptionKey);

    // Add to IPFS
    const result = await this.ipfs.add(encrypted, {
      pin: true,
      wrapWithDirectory: false
    });

    // Create shareable encryption keys for recipients
    const sharedKeys = await Promise.all(
      recipientPubKeys.map(pubKey =>
        this.encryption.encryptForRecipient(encryptionKey, pubKey)
      )
    );

    return {
      ipfsHash: result.cid.toString(),
      filename: metadata.filename,
      size: file.length,
      mimeType: metadata.mimeType,
      encryptionKey: encryptionKey.toString('hex'),
      algorithm: 'AES-256-GCM',
      checksums: {
        sha256: sha256(file).toString('hex')
      },
      sharedKeys // Each recipient gets their encrypted key
    };
  }

  async downloadFile(
    attachment: FileAttachment,
    decryptionKey: Buffer
  ): Promise<Buffer> {
    // Fetch from IPFS
    const chunks: Uint8Array[] = [];
    for await (const chunk of this.ipfs.cat(attachment.ipfsHash)) {
      chunks.push(chunk);
    }

    const encrypted = Buffer.concat(chunks);

    // Decrypt
    const decrypted = await this.decryptFile(encrypted, decryptionKey);

    // Verify checksum
    const checksum = sha256(decrypted).toString('hex');
    if (checksum !== attachment.checksums.sha256) {
      throw new Error('File integrity check failed');
    }

    return decrypted;
  }

  async setupFileSharing(): Promise<void> {
    // Join LNMP-specific pubsub channels for file discovery
    await this.ipfs.pubsub.subscribe('lnmp-files', (msg) => {
      this.handleFileAnnouncement(msg);
    });

    // Announce our files periodically
    setInterval(() => {
      this.announceLocalFiles();
    }, 60000); // Every minute
  }
}
```

### 6. Message Router

```typescript
// src/routing/MessageRouter.ts
export class MessageRouter {
  private relays: Map<string, RelayClient> = new Map();
  private directConnections: Map<string, PeerConnection> = new Map();
  private dht: DistributedHashTable;

  constructor(
    private identity: Identity,
    private lightning: LightningService
  ) {
    this.dht = new DistributedHashTable({
      nodeId: identity.publicKey
    });
  }

  async sendMessage(
    message: LNMPMessage,
    options: SendOptions = {}
  ): Promise<SendResult> {
    // Try direct peer-to-peer first
    const directSent = await this.tryDirectSend(message);
    if (directSent.success) {
      return directSent;
    }

    // Fall back to relay routing
    const recipientRelays = await this.discoverRelays(message.to);

    // Send to multiple relays for redundancy
    const results = await Promise.allSettled(
      recipientRelays.map(relay =>
        this.sendViaRelay(message, relay)
      )
    );

    // Return success if at least one relay accepted
    const successCount = results.filter(r => r.status === 'fulfilled').length;

    return {
      success: successCount > 0,
      relaysUsed: successCount,
      messageId: message.id,
      timestamp: Date.now()
    };
  }

  private async tryDirectSend(message: LNMPMessage): Promise<SendResult> {
    const recipientPubKey = message.to[0]; // TODO: Handle multiple recipients

    // Check if we have direct connection
    let connection = this.directConnections.get(recipientPubKey);

    if (!connection) {
      // Try to establish direct connection
      const peerInfo = await this.dht.findPeer(recipientPubKey);

      if (peerInfo && peerInfo.addresses.length > 0) {
        connection = await this.establishDirectConnection(peerInfo);
        this.directConnections.set(recipientPubKey, connection);
      }
    }

    if (connection && connection.isConnected()) {
      await connection.send(message);
      return { success: true, direct: true };
    }

    return { success: false };
  }

  private async discoverRelays(recipientPubKeys: string[]): Promise<RelayInfo[]> {
    const relayLists = await Promise.all(
      recipientPubKeys.map(pubKey =>
        this.dht.get(`relays:${pubKey}`)
      )
    );

    // Merge and deduplicate relay lists
    const allRelays = new Set(relayLists.flat());

    // Sort by reliability/performance metrics
    return Array.from(allRelays).sort((a, b) =>
      this.getRelayScore(b) - this.getRelayScore(a)
    );
  }
}
```

---

## Network Topology

### Distributed Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client A  │────▶│   Relay 1   │────▶│   Client B  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                    │                    │
       │                    ▼                    │
       │            ┌─────────────┐              │
       └───────────▶│   Relay 2   │◀─────────────┘
                    └─────────────┘
                            │
                            ▼
                    ┌─────────────┐
                    │    IPFS     │
                    │   Network   │
                    └─────────────┘
```

### Component Interactions

1. **Clients** maintain connections to multiple relays
2. **Relays** federate messages between each other
3. **IPFS** stores encrypted attachments
4. **Lightning** handles all payments
5. **DHT** enables peer and relay discovery

---

## Economic Model

### Actor Incentives

#### Users
- **Send messages** by paying small amounts (spam prevention)
- **Earn sats** by receiving legitimate messages
- **Control pricing** for their attention
- **No subscription fees** - pay only for what you use

#### Relay Operators
```typescript
interface RelayEconomics {
  revenue: {
    storageFeesPercent: 0.1,      // 10% of message payments
    premiumFeatures: 5000,         // Monthly sats for premium
    apiAccess: 'pay-per-call',     // Commercial API usage
    bandwidthOverage: 100          // Sats per GB over limit
  };

  costs: {
    hosting: 'Self-managed',
    bandwidth: 'Metered',
    storage: 'Pruned after 30 days'
  };

  profitability: {
    breakEven: '1000 active users',
    profitable: '5000 active users',
    margin: '70% after scale'
  };
}
```

#### Developers
- **Build clients** and earn from premium features
- **Create bridges** to other protocols
- **Offer services** (backup, search, analytics)

### Pricing Examples

```typescript
const defaultPricing: PricingPolicy = {
  stranger: 100,        // 100 sats from unknown senders
  acquaintance: 10,     // 10 sats from one-time contacts
  friend: 0,            // Free from whitelisted
  priority: 1000,       // 1000+ sats for immediate attention

  // Dynamic pricing
  surge: {
    enabled: true,
    multiplier: 2,      // 2x during busy hours
    threshold: 100      // When inbox > 100 unread
  }
};
```

### Economic Flows

```
Sender (100 sats) ──▶ Relay 1 (10 sats) ──▶ Relay 2 (10 sats) ──▶ Recipient (80 sats)
                            │                         │
                            ▼                         ▼
                     Relay Operator             Relay Operator
                       Earnings                   Earnings
```

---

## Implementation Specification

### Project Structure

```
lnmp/
├── packages/
│   ├── core/               # Core protocol implementation
│   │   ├── src/
│   │   │   ├── identity/   # Key management
│   │   │   ├── crypto/     # Encryption/signing
│   │   │   ├── messages/   # Message handling
│   │   │   └── network/    # P2P networking
│   │   └── package.json
│   │
│   ├── relay/             # Relay server
│   │   ├── src/
│   │   │   ├── server/    # WebSocket server
│   │   │   ├── storage/   # Message storage
│   │   │   ├── federation/# Relay-to-relay
│   │   │   └── billing/   # Lightning payments
│   │   └── package.json
│   │
│   ├── client/            # Reference client
│   │   ├── src/
│   │   │   ├── ui/        # User interface
│   │   │   ├── sync/      # Message sync
│   │   │   └── wallet/    # Lightning wallet
│   │   └── package.json
│   │
│   └── bridge/            # Email bridge
│       ├── src/
│       │   ├── smtp/      # SMTP server
│       │   ├── imap/      # IMAP server
│       │   └── converter/ # Format conversion
│       └── package.json
│
├── bun.lockb
└── README.md
```

### Core Dependencies

```json
{
  "dependencies": {
    "@noble/secp256k1": "^2.0.0",     // Cryptography
    "@ipld/dag-cbor": "^9.0.0",       // IPFS data structures
    "lightning": "^9.0.0",             // Lightning Network
    "ws": "^8.0.0",                    // WebSockets
    "@chainsafe/libp2p-noise": "^13.0.0", // Encrypted transport
    "level": "^8.0.0",                 // Local storage
    "zod": "^3.0.0"                    // Schema validation
  },
  "devDependencies": {
    "bun-types": "latest",
    "@types/ws": "^8.0.0",
    "vitest": "^1.0.0"
  }
}
```

### Key Implementation Files

```typescript
// packages/core/src/protocol.ts
export const PROTOCOL_VERSION = '1.0.0';
export const DEFAULT_PORT = 7138; // LNMP on phone keypad

export const MessageKinds = {
  TEXT: 1,
  VOICE: 2,
  FILE: 3,
  PAYMENT: 4,
  RECEIPT: 5,
  PRESENCE: 6,
  TYPING: 7,
  REACTION: 8
} as const;

export const RelayCommands = {
  // Client to Relay
  AUTH: 'AUTH',
  SUB: 'SUB',
  UNSUB: 'UNSUB',
  PUBLISH: 'PUBLISH',
  QUERY: 'QUERY',

  // Relay to Client
  MSG: 'MSG',
  NOTICE: 'NOTICE',
  OK: 'OK',
  ERROR: 'ERROR',
  INVOICE: 'INVOICE'
} as const;
```

---

## User Stories

### Story 1: Sarah the Freelancer
**As a** freelance designer
**I want to** receive payment requests via messages
**So that** clients can pay me instantly without wire transfers

**Implementation:**
```typescript
// Sarah receives a message with embedded invoice
const message = {
  type: 'payment_request',
  content: 'Invoice for logo design',
  payment: {
    amount: 500000, // sats
    memo: 'Logo design - Project X',
    invoice: 'lnbc500...'
  }
};

// One-click payment from message
await client.payInvoice(message.payment.invoice);
```

### Story 2: Alex the Privacy Advocate
**As a** privacy-conscious user
**I want to** communicate without revealing my identity
**So that** I can maintain anonymity while still being reachable

**Implementation:**
```typescript
// Alex creates temporary addresses
const tempAddress = await identity.createTemporaryAddress({
  expiry: Date.now() + 86400000, // 24 hours
  maxUses: 1 // Single use
});
// Returns: anon#x7j3k2@relay.onion

// Messages to this address forward to Alex's main identity
// But sender cannot link the two
```

### Story 3: TechCo Customer Support
**As a** support team
**I want to** prioritize urgent requests
**So that** critical issues are handled first

**Implementation:**
```typescript
// Customers pay more for urgent support
const urgentMessage = {
  to: 'support#a3f2b9@techco.com',
  content: 'Server is down!',
  payment: { amount: 5000 } // 50x normal rate
};

// Support dashboard shows paid messages first
const queue = await support.getMessageQueue();
// Sorted by payment amount descending
```

### Story 4: Bitcoin Podcast Network
**As a** podcast network
**I want to** receive listener feedback with micropayments
**So that** quality feedback is incentivized

**Implementation:**
```typescript
// Listeners send boosts with messages
const boost = {
  to: 'podcast#b4c3d2@bitcoinpod.com',
  content: 'Great episode! More content on Lightning please',
  payment: { amount: 1000 }, // 1000 sat boost
  metadata: {
    episode: 'ep-123',
    timestamp: '00:45:30'
  }
};

// Podcasters see real-time feedback
dashboard.on('boost', (boost) => {
  console.log(`${boost.amount} sats: ${boost.content}`);
});
```

### Story 5: Global Marketplace
**As a** marketplace seller
**I want to** negotiate with buyers in real-time
**So that** deals can be closed instantly with payment

**Implementation:**
```typescript
// Real-time negotiation with payments
const negotiation = new MessageThread({
  participants: ['buyer', 'seller'],
  escrow: true // Payments held until deal confirmed
});

// Buyer makes offer
await negotiation.send({
  from: 'buyer',
  content: 'Would you accept 50,000 sats?',
  payment: {
    amount: 50000,
    type: 'held' // Not released until seller accepts
  }
});

// Seller accepts and payment releases automatically
await negotiation.accept();
```

---

## Advantages Over Email

### Technical Superiority

| Feature | Email (SMTP) | LNMP |
|---------|--------------|------|
| **Identity** | Spoofable addresses | Cryptographic keys |
| **Spam Prevention** | Filters (often fail) | Economic cost |
| **Encryption** | Bolt-on (PGP) | Native, always on |
| **Payments** | Impossible | Native Lightning |
| **Attachments** | Base64 bloat | IPFS references |
| **Privacy** | Metadata exposed | Onion routing option |
| **Decentralization** | Tends toward Gmail | Incentivized federation |
| **Message Control** | None after send | Edit/delete/expire |

### User Experience Improvements

1. **No Spam**: Economic cost makes mass spam impossible
2. **Instant Payments**: Send money as easily as text
3. **True Privacy**: Not even relays can read messages
4. **Global Access**: Works without bank account
5. **Censorship Resistant**: No company can ban you

---

## Security Model

### Threat Model

```typescript
interface ThreatModel {
  // What we protect against
  protected: [
    'Message content interception',
    'Sender identity spoofing',
    'Relay censorship',
    'Spam and abuse',
    'Payment theft',
    'Metadata analysis'
  ];

  // Known limitations
  limitations: [
    'Traffic analysis at network level',
    'Compromised endpoint devices',
    'Quantum computer attacks (future)',
    'Social engineering'
  ];
}
```

### Security Features

1. **End-to-End Encryption**: Signal protocol with forward secrecy
2. **Authentication**: Every message cryptographically signed
3. **Payment Security**: Lightning's atomic swaps
4. **Metadata Protection**: Onion routing option
5. **Relay Security**: TLS + Noise protocol

---

## Development Roadmap

### Phase 1: Core Protocol (Months 1-3)
- [ ] Identity management system
- [ ] Basic message encryption
- [ ] Lightning payment integration
- [ ] Reference relay implementation
- [ ] Protocol specification v1.0

### Phase 2: Network Launch (Months 4-6)
- [ ] Public relay network
- [ ] Desktop client (Electron)
- [ ] Mobile client (React Native)
- [ ] Email bridge
- [ ] Developer documentation

### Phase 3: Advanced Features (Months 7-9)
- [ ] Group messaging
- [ ] Voice/video calls
- [ ] File sharing marketplace
- [ ] Advanced privacy features
- [ ] Enterprise features

### Phase 4: Ecosystem Growth (Months 10-12)
- [ ] Third-party client ecosystem
- [ ] Plugin architecture
- [ ] Governance model
- [ ] Mobile SDK
- [ ] Business partnerships

---

## Conclusion

LNMP represents a fundamental reimagining of electronic messaging, built on Bitcoin's principles of decentralization, cryptographic proof, and aligned incentives. By solving email's core problems at the protocol level, we can create a communication network that is spam-free, private, and economically sustainable for all participants.

The protocol is designed to be:
- **Decentralized**: No single point of control
- **Encrypted**: Privacy by default
- **Economic**: Spam becomes unprofitable
- **Extensible**: Rich ecosystem possible
- **Interoperable**: Bridges to legacy systems

With Lightning Network providing instant, global payments and IPFS handling distributed storage, LNMP can offer a superior messaging experience while maintaining the open, permissionless ethos of Bitcoin.
