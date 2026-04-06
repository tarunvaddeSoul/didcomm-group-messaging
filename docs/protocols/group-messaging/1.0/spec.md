---
title: Group Messaging
publisher: tarunvadde
license: MIT
piuri: https://didcomm.org/group-messaging/1.0
status: Proposed
summary: >
  A protocol for secure, decentralized group communication over DIDComm v2.
  Defines group lifecycle management, efficient group content encryption using
  a shared symmetric key distributed via pairwise DIDComm channels, epoch-based
  key rotation for forward secrecy on membership changes, and optional mediator
  fan-out for scalable delivery.
tags:
  - group
  - messaging
  - encryption
  - mediation
authors:
  - name: Tarun Vadde
    email: vaddeofficial@gmail.com
---

# Group Messaging Protocol 1.0

## Summary

This protocol defines a mechanism for secure group communication over DIDComm v2.
It introduces a **Group Content Key (GCK)** — a shared symmetric key that enables
O(1) message encryption regardless of group size — distributed and rotated through
standard DIDComm pairwise authenticated encryption. Group state progresses through
a linear sequence of **epochs**, where each epoch represents a distinct membership
set and key generation, providing forward secrecy when members are removed.

The protocol is designed to compose with existing DIDComm v2 infrastructure:
[Coordinate Mediation 2.0](https://didcomm.org/coordinate-mediation/2.0/),
[Message Pickup 3.0](https://didcomm.org/messagepickup/3.0/),
[Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing),
and [Discover Features 2.0](https://didcomm.org/discover-features/2.0/).

## Motivation

DIDComm v2 provides robust pairwise encrypted messaging between two parties. However,
many real-world use cases require communication among three or more participants:

- **Organizational coordination**: A credential issuer, holder, and multiple verifiers
  coordinating a presentation workflow.
- **Multi-party negotiations**: Supply chain participants coordinating shipment status
  updates where all parties must see the same information.
- **Community channels**: An SSI community coordinating standards work across
  multiple agents and organizations.

The DIDComm v2 base specification acknowledges that "interactions can involve more
than 2 parties" but provides no protocol for managing group state, distributing
group keys, or efficiently encrypting messages for multiple recipients.

Naively encrypting a message N times (once per member) using standard DIDComm
authcrypt requires O(N) ECDH key agreement operations per message. For a group of
50 members, this means 50 independent ECDH + AES operations on the sender's device
for every message — a cost that is prohibitive on mobile and constrained devices.

This protocol addresses these gaps by defining:

1. A **group lifecycle** (creation, membership changes, dissolution).
2. An **epoch-based key management** scheme inspired by
   [MLS (RFC 9420)](https://www.rfc-editor.org/rfc/rfc9420) concepts, adapted to
   DIDComm's asynchronous, transport-agnostic model.
3. A **group content encryption** layer using a shared symmetric key, achieving O(1)
   encryption cost per message regardless of group size.
4. An optional **group mediator** role for server-assisted fan-out delivery, modeled
   after the existing DIDComm
   [Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing)
   forward message pattern.

### Design Principles

This protocol adheres to the following principles, consistent with the
[DIDComm v2 specification](https://identity.foundation/didcomm-messaging/spec/v2.1/):

- **Transport agnostic**: The protocol makes no assumptions about the underlying
  transport. Group messages can traverse HTTP, WebSocket, Bluetooth, NFC, or any
  transport that DIDComm v2 supports.
- **Asynchronous**: No assumption of simultaneous online presence. Members may be
  offline when group state changes occur and must be able to catch up.
- **Decentralized**: No single point of control. Any authorized member can propose
  membership changes. The optional group mediator is an untrusted relay — it never
  sees plaintext.
- **Composable**: Built entirely on DIDComm v2 primitives. Does not require
  modifications to the base DIDComm specification.
- **Privacy preserving**: Group membership is known only to group members. The group
  mediator (if used) knows member DIDs for routing purposes but cannot read message
  content.

## Roles

This protocol defines the following roles:

| Role | Description |
|------|-------------|
| `admin` | A group member with elevated privileges. Can create the group, add or remove members, rotate keys, and dissolve the group. At least one admin MUST exist at all times. The creator of the group is the initial admin. Admin privileges can be granted to other members. |
| `member` | A participant in the group. Can send and receive group messages. May propose membership changes subject to the group's policy. |
| `group-mediator` | OPTIONAL. An agent that receives group delivery envelopes and fans out to individual member endpoints. Does not hold the GCK and cannot decrypt group content. Operates using the existing DIDComm [Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing) forward message semantics. |

A single agent MAY fulfill multiple roles. For example, the admin is always also a
member. An admin's agent MAY also serve as the group mediator.

## Requirements

- All agents MUST support DIDComm v2 authenticated encryption (authcrypt) for
  pairwise channels used for key distribution.
- All agents MUST support AES-256-GCM ([RFC 5116](https://www.rfc-editor.org/rfc/rfc5116))
  for group content encryption.
- All agents MUST support the `return_route` extension as defined in the DIDComm v2
  specification when communicating with mediators.
- Agents acting as group mediators MUST support
  [Coordinate Mediation 2.0](https://didcomm.org/coordinate-mediation/2.0/) and the
  DIDComm [Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing)
  forward message type.
- All agents SHOULD support
  [Discover Features 2.0](https://didcomm.org/discover-features/2.0/) and SHOULD
  advertise support for `https://didcomm.org/group-messaging/1.0` when queried.

## Connectivity

The following request-response pairs define the protocol's message flow:

| Initiator Sends | Responder Sends | Notes |
|-----------------|-----------------|-------|
| `create` | `create-ack` | Group creation; sent pairwise to each initial member |
| `message` | *(none)* | Group content message; no protocol-level response required |
| `add-member` | `add-member-ack` | Membership addition; pairwise to existing + new members |
| `remove-member` | `remove-member-ack` | Membership removal; pairwise to remaining members |
| `key-rotate` | `key-rotate-ack` | Epoch advancement without membership change |
| `leave` | `leave-ack` | Voluntary departure |
| `info-request` | `info` | Query current group state |
| `dissolve` | `dissolve-ack` | Group dissolution |

## States

### Admin State Machine

```
                          ┌──────────┐
              create ──►  │ CREATED  │
                          └────┬─────┘
                               │ all members ack
                               ▼
                   ┌──────────────────────┐
         ┌───────►│       ACTIVE          │◄────────┐
         │        └───────────┬───────────┘         │
         │                    │                     │
    ack received         membership change     key-rotate
         │            or key rotation               │
         │                    │                     │
         │         ┌──────────▼──────────┐          │
         └─────────│   EPOCH_ADVANCING   ├──────────┘
                   └──────────┬──────────┘
                              │ dissolve
                              ▼
                        ┌───────────┐
                        │ DISSOLVED │
                        └───────────┘
```

### Member State Machine

```
                        ┌──────────┐
           receive ──►  │ INVITED  │
           create       └────┬─────┘
                             │ send create-ack
                             ▼
                  ┌──────────────────────┐
        ┌────────│       ACTIVE          │◄───────┐
        │        └───────────┬───────────┘        │
        │                    │                    │
   new epoch            receive epoch         leave /
   applied              change msg            removed
        │                    │                    │
        │        ┌───────────▼───────────┐        │
        └────────│    UPDATING_KEYS      │        │
                 └───────────┬───────────┘        │
                             │ receive dissolve   │
                             ▼                    │
                       ┌───────────┐              │
                       │   LEFT    │◄─────────────┘
                       └───────────┘
```

### Group Mediator State Machine

The group mediator is stateless with respect to the group protocol. It processes
`forward` messages according to
[Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing)
and maintains a routing table mapping member DIDs to their service endpoints, as
configured through
[Coordinate Mediation 2.0](https://didcomm.org/coordinate-mediation/2.0/).

## Basic Walkthrough

### 1. Group Creation

Alice wants to create a group with Bob and Carol.

1. Alice generates a random `group_id` (UUID v4), sets `epoch` to `0`, and generates
   a cryptographically random 256-bit AES key as the initial Group Content Key (GCK).
2. Alice constructs a `create` message containing the group metadata, member list, and
   the GCK.
3. Alice sends the `create` message to Bob via standard DIDComm v2 **authcrypt** over
   their existing pairwise channel. She sends a separate `create` message to Carol
   over their pairwise channel. The GCK is included in the `body` of these pairwise
   messages — it is protected by DIDComm's authenticated encryption, not transmitted
   in plaintext.
4. Bob and Carol each verify the message, store the group state (group_id, epoch,
   member list, GCK), and respond with `create-ack`.
5. The group is now ACTIVE at epoch 0.

### 2. Sending a Group Message

Alice wants to send "Hello everyone" to the group.

1. Alice constructs a plaintext group message with `type` set to
   `https://didcomm.org/group-messaging/1.0/message`.
2. Alice serializes the plaintext to bytes, then encrypts it using AES-256-GCM with
   the current GCK. This produces a ciphertext, an IV, and an authentication tag.
   This is a **single symmetric encryption operation** regardless of group size.
3. Alice constructs a `delivery` envelope containing the ciphertext, IV, tag, the
   `group_id`, and the current `epoch` number.
4. Alice delivers the envelope to each member:
   - **Without group mediator**: Alice wraps the delivery envelope in a DIDComm v2
     **anoncrypt** message for each member's DID and transmits through each member's
     service endpoint (possibly via their individual mediators using standard
     `forward` messages).
   - **With group mediator**: Alice wraps the delivery envelope in a single DIDComm
     v2 **anoncrypt** message addressed to the group mediator. The group mediator
     then fans out by wrapping the delivery envelope in individual `forward` messages
     to each member's endpoint.
5. Each recipient receives the delivery envelope, looks up the GCK for the indicated
   `group_id` and `epoch`, decrypts the ciphertext using AES-256-GCM, and processes
   the resulting plaintext group message.

### 3. Adding a Member

Alice (admin) wants to add Dave to the group.

1. Alice generates a new GCK for epoch `N+1`.
2. Alice sends an `add-member` message to all **existing** members (Bob, Carol) via
   pairwise authcrypt. This message contains the new member list, the new epoch
   number, and the new GCK.
3. Alice sends a `create` message to Dave via pairwise authcrypt, containing the full
   group state at the new epoch (group_id, epoch N+1, member list, new GCK, group
   metadata). Dave does not receive the previous epoch's GCK — he cannot decrypt
   historical messages.
4. Existing members verify the message, update their stored group state to the new
   epoch, and respond with `add-member-ack`.
5. Dave processes the `create` message as a new group invitation and responds with
   `create-ack`.

### 4. Removing a Member

Alice (admin) wants to remove Carol from the group.

1. Alice generates a new GCK for epoch `N+1`.
2. Alice sends a `remove-member` message to all **remaining** members (Bob, Dave) via
   pairwise authcrypt. This message contains the updated member list, the new epoch
   number, and the new GCK. Carol does NOT receive this message.
3. Alice sends a `remove-member` notification to Carol indicating she has been removed.
   This notification does NOT contain the new GCK.
4. Remaining members verify the message, update their stored group state, discard the
   old GCK, and respond with `remove-member-ack`.
5. Carol discards her stored group state for this group.

**Forward secrecy guarantee**: Because the new GCK is never transmitted to Carol, she
cannot decrypt any messages sent after epoch N+1. Because the new GCK is generated
from fresh randomness (not derived from the old GCK), possession of the old GCK
provides no information about the new one.

### 5. Voluntary Departure

Bob wants to leave the group.

1. Bob sends a `leave` message to all other members via pairwise authcrypt.
2. The admin (Alice) processes the leave by generating a new GCK for epoch `N+1` and
   distributing it to remaining members via a `key-rotate` message.
3. Bob discards his stored group state.

### 6. Periodic Key Rotation

To limit the window of exposure if a GCK is compromised, admins SHOULD periodically
rotate the GCK even when membership has not changed.

1. Alice generates a new GCK for epoch `N+1`.
2. Alice sends a `key-rotate` message to all members via pairwise authcrypt with the
   new GCK and epoch number.
3. Members update their stored group state and respond with `key-rotate-ack`.

Agents SHOULD implement a configurable key rotation policy. Recommended defaults:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rotation_period_seconds` | 604800 (7 days) | Maximum time before rotation |
| `rotation_period_messages` | 100 | Maximum messages before rotation |

These defaults are drawn from production experience in the
[Matrix Megolm specification](https://spec.matrix.org/latest/client-server-api/#end-to-end-encryption),
which uses identical values.

## Design By Contract

### Preconditions

- A pairwise DIDComm v2 connection MUST exist between the admin and every member
  before group creation or membership changes.
- The admin MUST verify that the target DID is resolvable and has a valid
  `keyAgreement` section in its DID document before adding a member.
- The agent MUST have a stored GCK for the indicated epoch before attempting to
  decrypt a group message.

### Postconditions

- After a successful `create` exchange, all initial members MUST hold the same
  `group_id`, `epoch`, member list, and GCK.
- After a successful `add-member` or `remove-member` exchange, all continuing members
  MUST hold the same updated state at the new epoch.
- After a `remove-member`, the removed member MUST NOT hold the new epoch's GCK.

### Invariants

- At least one admin MUST exist in the group at all times.
- The `epoch` counter MUST be strictly monotonically increasing.
- The GCK MUST be generated from a cryptographically secure random number generator.
  It MUST NOT be derived from a previous epoch's GCK.
- The `group_id` MUST NOT change across epochs.
- All group management messages (`create`, `add-member`, `remove-member`,
  `key-rotate`, `dissolve`) MUST be sent via pairwise DIDComm v2 **authcrypt**. They
  MUST NOT be sent as group-encrypted messages.
- All `delivery` envelopes MUST be sent via DIDComm v2 **anoncrypt** to preserve
  sender privacy at the transport layer.

### Error Conditions

| Condition | Behavior |
|-----------|----------|
| Received `delivery` with unknown `group_id` | Discard silently. MUST NOT send problem report (prevents group discovery by probing). |
| Received `delivery` with unknown `epoch` higher than stored | Queue the message. Request current group state via `info-request` to the sender or known admin. |
| Received `delivery` with `epoch` lower than stored | Attempt decryption if the old GCK is still cached (within a grace window). Otherwise discard. |
| Received `add-member` or `remove-member` from non-admin | Discard. Send problem report with code `e.p.group-messaging.unauthorized`. |
| GCK decryption fails (invalid tag) | Discard the message. Send problem report with code `e.m.group-messaging.decrypt-failed`. |
| Member receives `remove-member` listing themselves | Discard the group state. Send `remove-member-ack`. |

## Delivery Models

This protocol supports three delivery models for `delivery` messages. Implementations
MUST support at least the admin-relay model.

### 1. Direct Fan-Out

Each sender encrypts the delivery envelope once (O(1) with the GCK), then sends the
same envelope to every other group member via their pairwise DIDComm v2 connections.
This requires that all members maintain pairwise connections with all other members.

**When to use**: Small groups where all members have established pairwise connections.

### 2. Admin-Relay (Hub-and-Spoke)

Non-admin members send `delivery` messages only to the admin via their pairwise
connection. The admin, upon receiving a delivery, stores and processes the message,
then forwards the same delivery envelope to all other members via the admin's
pairwise connections.

This model is required when using `did:peer:4` (or any pairwise DID method) because
each pairwise connection uses a unique DID. Members may not be able to correlate
group member DIDs with their own pairwise connections to other members, but they
always have a pairwise connection to the admin who created the group.

**Trade-offs**:
- Admin is a single point of failure for delivery — if admin is offline,
  non-admin members cannot send messages.
- Latency doubles for non-admin-originated messages (member -> admin -> members).
- Admin can observe all message timing (but cannot forge content due to
  AAD-bound GCK encryption with the original sender's DID).

**When to use**: Groups using pairwise DID methods, or when members lack full-mesh
pairwise connections.

### 3. Group Mediator Fan-Out

A dedicated mediator service holds the group's fan-out routing table. The sender
sends a single delivery envelope to the mediator, which distributes it to all
members. See the `group_mediator_did` and `group_mediator_endpoint` fields in the
`create` message for configuration.

**When to use**: Large groups (>20 members) or when a persistent delivery
infrastructure is available.

## Security

### Threat Model

This protocol considers the following adversaries:

1. **Network observer**: Can observe encrypted traffic between agents but cannot
   decrypt DIDComm messages. The use of anoncrypt for delivery envelopes prevents the
   observer from identifying the sender. Group message content is additionally
   encrypted under the GCK.

2. **Compromised mediator**: The group mediator (if used) can observe delivery
   envelopes but cannot decrypt group content because it does not hold the GCK. It
   can observe message timing, frequency, and approximate size. It knows the member
   list (required for routing).

3. **Removed member**: After removal, a member no longer holds the current GCK.
   Because each epoch's GCK is generated from fresh randomness, a removed member
   cannot derive future GCKs from past ones. However, a removed member retains all
   GCKs from epochs during which they were a member and can decrypt messages from
   those epochs if they retained the ciphertexts.

4. **Compromised member**: A current member who has their device or keys compromised.
   The attacker gains access to the current GCK and can read all group messages until
   the next key rotation. Periodic key rotation (see [Periodic Key Rotation](#6-periodic-key-rotation))
   limits this window.

### Forward Secrecy

This protocol provides **forward secrecy at epoch boundaries**: compromise of an
epoch's GCK does not reveal messages from other epochs. Within an epoch, all messages
are encrypted under the same GCK, so compromise of the GCK exposes all messages in
that epoch. This is the same trade-off made by
[Matrix Megolm](https://spec.matrix.org/latest/client-server-api/#end-to-end-encryption)
and [Signal Sender Keys](https://signal.org/docs/specifications/group-v2/).

Full per-message forward secrecy (as in MLS TreeKEM) requires O(log N) asymmetric
operations per message and significant state synchronization — a cost deemed
disproportionate for DIDComm's asynchronous, transport-agnostic environment where
members may be offline for extended periods.

Implementations SHOULD enforce the key rotation policy defaults to bound the
exposure window.

### Post-Compromise Security

This protocol provides **post-compromise security through key rotation**: once a
`key-rotate` or membership change message is processed, the new GCK is generated
from fresh randomness unknown to the attacker. Future messages are secure even if
the attacker held the previous GCK.

Admins SHOULD trigger an immediate key rotation if a member reports a suspected
compromise.

### Authentication

- **Group management messages** are authenticated via DIDComm v2 authcrypt. The
  `from` field of the plaintext is cryptographically bound to the sender's DID through
  the ECDH-1PU key agreement. Recipients MUST verify that the sender is an admin
  for operations that require admin privileges.
- **Group content messages** are authenticated at two layers:
  1. The outer DIDComm anoncrypt envelope provides integrity (AEAD).
  2. The inner GCK encryption (AES-256-GCM) provides authenticated encryption. The
     `sender` field in the delivery envelope identifies the sender's DID. Recipients
     MUST verify that the sender is a current member of the group at the indicated
     epoch.

### Replay Protection

Each group message MUST include a unique `id` (as required by the DIDComm v2 base
specification). Recipients MUST track received message IDs per group and discard
duplicates. Implementations SHOULD use a bounded cache (e.g., sliding window or
time-based expiry) for replay detection.

### Confidentiality of Group Existence

The existence of a group and its membership are confidential to group members.
Non-members cannot discover a group by probing — delivery envelopes with unknown
`group_id` values are silently discarded without any response.

The group mediator (if used) is an exception: it necessarily knows which DIDs are
group members for routing purposes. This is consistent with the trust model of
DIDComm mediators generally — mediators know the parties they mediate for.

### Comparison with Alternative Approaches

| Property | This Protocol | MLS (RFC 9420) | Signal Sender Keys | Naive DIDComm N×Authcrypt |
|----------|--------------|----------------|--------------------|----|
| Encryption cost per message | O(1) symmetric | O(1) amortized | O(1) symmetric | O(N) ECDH |
| Key rotation cost | O(N) pairwise | O(log N) TreeKEM | O(N) pairwise | N/A |
| Forward secrecy | Per-epoch | Per-message | Per-chain-step | Per-message |
| Post-compromise security | Yes (via rotation) | Yes (via Update) | No (within session) | Yes (per-message) |
| Async/offline tolerance | High | Moderate | High | High |
| Implementation complexity | Low | High | Low | Minimal |
| DIDComm compatibility | Native | Requires adaptation | Requires adaptation | Native |

This protocol deliberately occupies the **low-complexity, high-compatibility** region
of the design space. It trades per-message forward secrecy for simplicity and full
compatibility with existing DIDComm infrastructure. For use cases requiring stronger
per-message forward secrecy, a future version of this protocol MAY introduce an
optional ratchet mechanism.

## Composition

| Supported Goal Code | Notes |
|---------------------|-------|
| `group.messaging` | General-purpose group communication |
| `group.coordination` | Multi-party workflow coordination |
| `group.notification` | One-to-many broadcast notifications |

## Message Reference

All messages in this protocol use the base URI:
`https://didcomm.org/group-messaging/1.0/`

### `create`

Sent by the admin to each initial member via **pairwise authcrypt** to establish
a new group.

**Type**: `https://didcomm.org/group-messaging/1.0/create`

```json
{
    "id": "create-001",
    "type": "https://didcomm.org/group-messaging/1.0/create",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "created_time": 1743523200,
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "name": "Project Aries Working Group",
        "epoch": 0,
        "members": [
            {
                "did": "did:peer:2.alice",
                "role": "admin"
            },
            {
                "did": "did:peer:2.bob",
                "role": "member"
            },
            {
                "did": "did:peer:2.carol",
                "role": "member"
            }
        ],
        "gck": {
            "k": "dGhpcyBpcyBhIGJhc2U2NHVybCBlbmNvZGVkIGFlcyBrZXk",
            "alg": "A256GCM"
        },
        "policy": {
            "member_can_invite": false,
            "member_can_leave": true,
            "rotation_period_seconds": 604800,
            "rotation_period_messages": 100,
            "max_members": 256
        },
        "group_mediator": {
            "did": "did:web:mediator.example.com",
            "endpoint": "https://mediator.example.com/didcomm"
        }
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.group_id` | string | REQUIRED | Unique identifier for the group. MUST be a URN UUID ([RFC 4122](https://www.rfc-editor.org/rfc/rfc4122)). |
| `body.name` | string | OPTIONAL | Human-readable group name. |
| `body.epoch` | integer | REQUIRED | Epoch counter. MUST be `0` for `create`. |
| `body.members` | array | REQUIRED | Array of member objects. Each object MUST contain `did` (string) and `role` (string, one of `admin` or `member`). |
| `body.gck` | object | REQUIRED | The Group Content Key. `k` is the base64url-encoded ([RFC 4648 §5](https://www.rfc-editor.org/rfc/rfc4648#section-5)) raw key bytes. `alg` identifies the content encryption algorithm. MUST be `A256GCM`. |
| `body.policy` | object | OPTIONAL | Group policy configuration. See [Group Policy](#group-policy). |
| `body.group_mediator` | object | OPTIONAL | Group mediator configuration. `did` is the mediator's DID. `endpoint` is the mediator's service endpoint URI. |

### `create-ack`

Sent by a member to the admin to confirm receipt and acceptance of the group
invitation.

**Type**: `https://didcomm.org/group-messaging/1.0/create-ack`

```json
{
    "id": "create-ack-001",
    "type": "https://didcomm.org/group-messaging/1.0/create-ack",
    "thid": "create-001",
    "from": "did:peer:2.bob",
    "to": ["did:peer:2.alice"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "status": "accepted"
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.group_id` | string | REQUIRED | The group identifier from the `create` message. |
| `body.status` | string | REQUIRED | One of `accepted` or `rejected`. If `rejected`, the member declines to join. |

### `message`

A group content message. This is the application-level payload that members
exchange within the group. This message type is **never sent directly over the
wire** — it is first encrypted with the GCK, then wrapped in a `delivery`
envelope (see [Delivery Envelope](#delivery-envelope)).

**Type**: `https://didcomm.org/group-messaging/1.0/message`

```json
{
    "id": "msg-456",
    "type": "https://didcomm.org/group-messaging/1.0/message",
    "from": "did:peer:2.alice",
    "created_time": 1743523500,
    "body": {
        "content": "The credential schema draft is ready for review."
    },
    "thid": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
    "lang": "en"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.content` | string | REQUIRED | The message content. |
| `thid` | string | REQUIRED | MUST be the `group_id`. This threads all group messages together. |
| `lang` | string | OPTIONAL | Language tag per [BCP 47](https://www.rfc-editor.org/info/bcp47). |
| `attachments` | array | OPTIONAL | DIDComm attachments as defined in the [DIDComm v2 specification](https://identity.foundation/didcomm-messaging/spec/v2.1/#attachments). |

### `delivery`

The delivery envelope wraps a GCK-encrypted group message for transport. This
is the message that actually traverses the network.

**Type**: `https://didcomm.org/group-messaging/1.0/delivery`

```json
{
    "id": "delivery-789",
    "type": "https://didcomm.org/group-messaging/1.0/delivery",
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "epoch": 3,
        "sender": "did:peer:2.alice",
        "ciphertext": "xWE9HUMhQ7...",
        "iv": "QGizMDBW...",
        "tag": "R29vZCBt..."
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.group_id` | string | REQUIRED | The group identifier. |
| `body.epoch` | integer | REQUIRED | The epoch of the GCK used for encryption. |
| `body.sender` | string | REQUIRED | The DID of the message sender. Recipients MUST verify this DID is a member of the group at the indicated epoch. |
| `body.ciphertext` | string | REQUIRED | Base64url-encoded ciphertext of the AES-256-GCM encrypted group message. |
| `body.iv` | string | REQUIRED | Base64url-encoded 96-bit initialization vector. MUST be generated from a cryptographically secure random number generator. MUST be unique per message. |
| `body.tag` | string | REQUIRED | Base64url-encoded 128-bit GCM authentication tag. |

**Additional Authenticated Data (AAD)**: The AAD input to AES-256-GCM MUST be the
UTF-8 encoding of the concatenation: `group_id || "." || epoch || "." || sender`.
This binds the ciphertext to the group context and sender identity, preventing
cross-group and cross-sender ciphertext transplant attacks.

**Transport**: The `delivery` message is wrapped in DIDComm v2 **anoncrypt** for
transmission to each recipient (or to the group mediator). Anoncrypt is used instead
of authcrypt because the sender is already identified in the `sender` field of the
delivery body, and anoncrypt avoids the additional ECDH-1PU cost.

### `add-member`

Sent by an admin to all existing members when a new member is added to the group.
The new member receives a `create` message instead (see
[Adding a Member](#3-adding-a-member)).

**Type**: `https://didcomm.org/group-messaging/1.0/add-member`

```json
{
    "id": "add-001",
    "type": "https://didcomm.org/group-messaging/1.0/add-member",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "created_time": 1743610000,
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "epoch": 4,
        "added": [
            {
                "did": "did:peer:2.dave",
                "role": "member"
            }
        ],
        "members": [
            { "did": "did:peer:2.alice", "role": "admin" },
            { "did": "did:peer:2.bob", "role": "member" },
            { "did": "did:peer:2.dave", "role": "member" }
        ],
        "gck": {
            "k": "bmV3IGdyb3VwIGNvbnRlbnQga2V5IGZvciBlcG9jaCA0",
            "alg": "A256GCM"
        },
        "epoch_hash": "sha256:abc123..."
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.group_id` | string | REQUIRED | The group identifier. |
| `body.epoch` | integer | REQUIRED | The new epoch number. MUST be exactly `previous_epoch + 1`. |
| `body.added` | array | REQUIRED | Array of member objects being added. |
| `body.members` | array | REQUIRED | Complete member list at the new epoch. |
| `body.gck` | object | REQUIRED | The new Group Content Key for this epoch. |
| `body.epoch_hash` | string | REQUIRED | See [Epoch Hash Chain](#epoch-hash-chain). |

### `add-member-ack`

**Type**: `https://didcomm.org/group-messaging/1.0/add-member-ack`

```json
{
    "id": "add-ack-001",
    "type": "https://didcomm.org/group-messaging/1.0/add-member-ack",
    "thid": "add-001",
    "from": "did:peer:2.bob",
    "to": ["did:peer:2.alice"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "epoch": 4,
        "status": "accepted"
    }
}
```

### `remove-member`

Sent by an admin to remaining members when one or more members are removed.

**Type**: `https://didcomm.org/group-messaging/1.0/remove-member`

```json
{
    "id": "remove-001",
    "type": "https://didcomm.org/group-messaging/1.0/remove-member",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "created_time": 1743696000,
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "epoch": 5,
        "removed": ["did:peer:2.carol"],
        "members": [
            { "did": "did:peer:2.alice", "role": "admin" },
            { "did": "did:peer:2.bob", "role": "member" },
            { "did": "did:peer:2.dave", "role": "member" }
        ],
        "gck": {
            "k": "eWV0IGFub3RoZXIgZ3JvdXAgY29udGVudCBrZXk",
            "alg": "A256GCM"
        },
        "epoch_hash": "sha256:def456..."
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.removed` | array | REQUIRED | Array of DID strings of members being removed. |

All other fields follow the same structure as `add-member`.

The removed member(s) receive a separate notification:

```json
{
    "id": "remove-notify-001",
    "type": "https://didcomm.org/group-messaging/1.0/remove-member",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.carol"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "removed": ["did:peer:2.carol"]
    }
}
```

This notification deliberately omits the new epoch, member list, and GCK.

### `remove-member-ack`

**Type**: `https://didcomm.org/group-messaging/1.0/remove-member-ack`

Follows the same structure as `add-member-ack` with the corresponding `thid`.

### `key-rotate`

Sent by an admin to all members to advance the epoch and distribute a new GCK
without a membership change.

**Type**: `https://didcomm.org/group-messaging/1.0/key-rotate`

```json
{
    "id": "rotate-001",
    "type": "https://didcomm.org/group-messaging/1.0/key-rotate",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "created_time": 1744128000,
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "epoch": 6,
        "reason": "scheduled",
        "gck": {
            "k": "cm90YXRlZCBncm91cCBjb250ZW50IGtleQ",
            "alg": "A256GCM"
        },
        "epoch_hash": "sha256:789abc..."
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.reason` | string | OPTIONAL | Reason for rotation. Defined values: `scheduled` (periodic rotation), `compromise` (suspected key compromise), `policy` (policy-triggered). |

### `key-rotate-ack`

**Type**: `https://didcomm.org/group-messaging/1.0/key-rotate-ack`

Follows the same structure as other ack messages with the corresponding `thid`.

### `leave`

Sent by a member to all other members to announce voluntary departure.

**Type**: `https://didcomm.org/group-messaging/1.0/leave`

```json
{
    "id": "leave-001",
    "type": "https://didcomm.org/group-messaging/1.0/leave",
    "from": "did:peer:2.bob",
    "to": ["did:peer:2.alice"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000"
    }
}
```

Upon receiving a `leave` message, the admin MUST initiate a key rotation (equivalent
to a `remove-member` flow for the departing member) to ensure forward secrecy.

### `leave-ack`

**Type**: `https://didcomm.org/group-messaging/1.0/leave-ack`

```json
{
    "id": "leave-ack-001",
    "type": "https://didcomm.org/group-messaging/1.0/leave-ack",
    "thid": "leave-001",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "status": "acknowledged"
    }
}
```

### `info-request`

Sent by a member to any other member (typically the admin) to request current group
state. Useful when a member has been offline and may have missed epoch transitions.

**Type**: `https://didcomm.org/group-messaging/1.0/info-request`

```json
{
    "id": "info-req-001",
    "type": "https://didcomm.org/group-messaging/1.0/info-request",
    "from": "did:peer:2.bob",
    "to": ["did:peer:2.alice"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "known_epoch": 3
    },
    "return_route": "all"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body.known_epoch` | integer | OPTIONAL | The highest epoch the requester knows about. Allows the responder to send only the delta. |

### `info`

Response to `info-request` containing the current group state.

**Type**: `https://didcomm.org/group-messaging/1.0/info`

```json
{
    "id": "info-001",
    "type": "https://didcomm.org/group-messaging/1.0/info",
    "thid": "info-req-001",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "name": "Project Aries Working Group",
        "epoch": 6,
        "members": [
            { "did": "did:peer:2.alice", "role": "admin" },
            { "did": "did:peer:2.bob", "role": "member" },
            { "did": "did:peer:2.dave", "role": "member" }
        ],
        "gck": {
            "k": "cm90YXRlZCBncm91cCBjb250ZW50IGtleQ",
            "alg": "A256GCM"
        },
        "policy": {
            "member_can_invite": false,
            "member_can_leave": true,
            "rotation_period_seconds": 604800,
            "rotation_period_messages": 100,
            "max_members": 256
        },
        "epoch_hash": "sha256:789abc...",
        "group_mediator": {
            "did": "did:web:mediator.example.com",
            "endpoint": "https://mediator.example.com/didcomm"
        }
    }
}
```

The `info` response MUST only be sent to a DID that is a current member of the group.
The responder MUST verify the requester's membership before sending the GCK.

### `dissolve`

Sent by an admin to all members to permanently dissolve the group.

**Type**: `https://didcomm.org/group-messaging/1.0/dissolve`

```json
{
    "id": "dissolve-001",
    "type": "https://didcomm.org/group-messaging/1.0/dissolve",
    "from": "did:peer:2.alice",
    "to": ["did:peer:2.bob"],
    "body": {
        "group_id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "reason": "Project completed."
    }
}
```

Upon receiving a `dissolve` message, members MUST discard all stored GCKs for the
group and transition to the LEFT state. No further messages should be sent or
accepted for this group.

### `dissolve-ack`

**Type**: `https://didcomm.org/group-messaging/1.0/dissolve-ack`

Follows the same structure as other ack messages.

## Epoch Hash Chain

To provide tamper evidence and allow members to verify the integrity of group state
transitions, each epoch (after epoch 0) includes an `epoch_hash` computed as:

```
epoch_hash = SHA-256(
    previous_epoch_hash ||
    epoch_number ||
    sorted_member_dids ||
    gck_fingerprint
)
```

Where:
- `previous_epoch_hash` is the `epoch_hash` from the prior epoch (empty string for
  epoch 0).
- `epoch_number` is the ASCII decimal representation of the epoch.
- `sorted_member_dids` is the concatenation of all member DIDs sorted lexicographically,
  separated by `|`.
- `gck_fingerprint` is the SHA-256 hash of the raw GCK bytes.

The `epoch_hash` is represented as `sha256:<hex-encoded-hash>`.

Members MUST verify the `epoch_hash` on each epoch transition. If verification fails,
the member MUST reject the epoch change and send a problem report with code
`e.p.group-messaging.epoch-hash-mismatch`.

This chain provides:
- **Tamper evidence**: Any modification to past group state is detectable.
- **Consistency verification**: Members can compare `epoch_hash` values to confirm
  they share the same view of group history.
- **Auditability**: The full chain of `epoch_hash` values constitutes a verifiable
  log of group state transitions.

## Group Policy

The `policy` object in the `create` message defines the group's governance rules.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `member_can_invite` | boolean | `false` | Whether non-admin members can propose adding new members. If `true`, members send `add-member` proposals that the admin must approve. |
| `member_can_leave` | boolean | `true` | Whether members can voluntarily leave. If `false`, only admins can remove members. |
| `rotation_period_seconds` | integer | `604800` | Maximum seconds between key rotations. |
| `rotation_period_messages` | integer | `100` | Maximum group messages between key rotations. |
| `max_members` | integer | `256` | Maximum number of members. Implementations MAY enforce lower limits. |

Policy changes are communicated via a `key-rotate` message with updated policy in
the `body`. Policy changes MUST trigger an epoch advancement.

## Group Mediator Integration

### Registration

When a group uses a group mediator, the admin registers the group's member DIDs with
the mediator using the existing
[Coordinate Mediation 2.0](https://didcomm.org/coordinate-mediation/2.0/) protocol:

1. Admin establishes a mediation relationship with the mediator via `mediate-request`
   / `mediate-grant`.
2. Admin registers each member's DID via `keylist-update` with `action: "add"`.
3. When membership changes, admin sends `keylist-update` to add or remove the
   affected DIDs.

### Delivery via Group Mediator

When a member sends a group message through the group mediator:

1. The sender constructs the `delivery` envelope as described in
   [Sending a Group Message](#2-sending-a-group-message).
2. The sender wraps the `delivery` envelope in a DIDComm v2 `forward` message
   ([Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing)):

```json
{
    "type": "https://didcomm.org/routing/2.0/forward",
    "id": "fwd-001",
    "to": ["did:web:mediator.example.com"],
    "body": {
        "next": "urn:uuid:550e8400-e29b-41d4-a716-446655440000"
    },
    "attachments": [
        {
            "id": "group-delivery",
            "media_type": "application/didcomm-plain+json",
            "data": {
                "json": { "...delivery envelope..." }
            }
        }
    ]
}
```

3. The group mediator receives the `forward`, resolves the `next` field (the
   `group_id`) to its member routing table, and fans out the attached delivery
   envelope to each member's service endpoint using individual `forward` messages.
4. Each member receives the delivery envelope through their own mediator (if any)
   using [Message Pickup 3.0](https://didcomm.org/messagepickup/3.0/).

### Mediator Trust Boundary

The group mediator:
- **CAN** observe: `group_id`, `epoch`, `sender` DID, message timing, approximate
  size, and the member list.
- **CANNOT** observe: message content (encrypted under the GCK), specific message
  type, attachments.
- **CANNOT** forge: messages (lacking the GCK, it cannot produce valid ciphertexts;
  the GCM authentication tag will fail verification).

This trust boundary is identical to the trust model for individual DIDComm mediators
as described in the base specification.

## L10n

Group message content localization SHOULD use the `lang` header as defined in the
DIDComm v2 specification. The `lang` header is set on the inner group message
(before GCK encryption), not on the delivery envelope.

Protocol-level messages (`create`, `add-member`, etc.) do not require localization
as they contain structured data rather than human-readable content.

## Implementations

*(No implementations yet. This is a Proposed protocol.)*

## Endnotes

### Future Considerations

- **Read receipts**: A future extension could define a `read-receipt` message type
  for per-message delivery confirmation within the group.
- **Reactions and threading**: Message-level threading within the group (replies to
  specific messages) could use the `pthid` header referencing the original message's
  `id`.
- **Typing indicators**: Real-time presence features could be defined as a separate
  protocol composing with this one.
- **Ratcheting GCK**: A future version could introduce an optional per-message
  symmetric ratchet (similar to Megolm) for improved within-epoch forward secrecy,
  at the cost of increased complexity for offline members.
- **Multi-admin consensus**: For groups with multiple admins, a future extension
  could define a consensus mechanism for membership changes (e.g., requiring approval
  from K of N admins).
- **Message ordering**: This protocol does not guarantee global message ordering
  across members. A future extension could introduce vector clocks or a similar
  mechanism for causal ordering.
- **Large groups (>256)**: For groups exceeding the default `max_members`, a future
  extension could introduce a TreeKEM-inspired key distribution mechanism to reduce
  key rotation cost from O(N) to O(log N).
- **Interoperability with DIDComm v1**: This protocol is specified for DIDComm v2
  only. Bridging to DIDComm v1 agents would require a translation layer.

### References

- [DIDComm Messaging v2.1](https://identity.foundation/didcomm-messaging/spec/v2.1/)
- [Coordinate Mediation 2.0](https://didcomm.org/coordinate-mediation/2.0/)
- [Message Pickup 3.0](https://didcomm.org/messagepickup/3.0/)
- [Routing Protocol 2.0](https://identity.foundation/didcomm-messaging/spec/v2.1/#routing)
- [Discover Features 2.0](https://didcomm.org/discover-features/2.0/)
- [Out of Band 2.0](https://didcomm.org/out-of-band/2.0/)
- [Basic Message 2.0](https://didcomm.org/basicmessage/2.0/)
- [Trust Ping 2.0](https://didcomm.org/trust-ping/2.0/)
- [MLS - Messaging Layer Security, RFC 9420](https://www.rfc-editor.org/rfc/rfc9420)
- [Signal Sender Keys](https://signal.org/docs/specifications/group-v2/)
- [Matrix Megolm](https://spec.matrix.org/latest/client-server-api/#end-to-end-encryption)
- [AES-GCM, RFC 5116](https://www.rfc-editor.org/rfc/rfc5116)
- [UUID, RFC 4122](https://www.rfc-editor.org/rfc/rfc4122)
- [Base64url, RFC 4648 §5](https://www.rfc-editor.org/rfc/rfc4648#section-5)
- [BCP 47 Language Tags](https://www.rfc-editor.org/info/bcp47)
- [DIDComm Problem Reports](https://identity.foundation/didcomm-messaging/spec/v2.1/#problem-reports)

### Problem Report Codes

This protocol defines the following problem report codes for use with
[DIDComm Problem Reports](https://identity.foundation/didcomm-messaging/spec/v2.1/#problem-reports):

| Code | Scope | Description |
|------|-------|-------------|
| `e.p.group-messaging.unauthorized` | Protocol | Sender lacks required privileges for the requested operation. |
| `e.p.group-messaging.epoch-hash-mismatch` | Protocol | The `epoch_hash` does not match the locally computed value. |
| `e.p.group-messaging.unknown-group` | Protocol | The `group_id` is not recognized. Sent only in response to management messages from known members, never for `delivery` envelopes. |
| `e.p.group-messaging.epoch-too-old` | Protocol | The epoch is too far behind the current epoch for the message to be processed. |
| `e.p.group-messaging.max-members-exceeded` | Protocol | Adding the proposed member(s) would exceed `max_members`. |
| `e.m.group-messaging.decrypt-failed` | Message | GCK decryption failed (invalid authentication tag). |
| `e.m.group-messaging.invalid-sender` | Message | The `sender` in the delivery envelope is not a member of the group at the indicated epoch. |
| `w.m.group-messaging.epoch-skipped` | Message | One or more epoch transitions were missed. The member should send `info-request` to sync. |
