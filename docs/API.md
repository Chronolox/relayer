# Chronolox Relayer API

API for the Chronolox Relayer service. This service is the operational backbone of the
Chronolox protocol: it bridges the Polygon payment layer to the Oasis Sapphire TEE
(Trusted Execution Environment), enables gasless secret access for heirs, and acts as
the gas sponsor for voucher users who hold no cryptocurrency.

## Overview

The relayer performs **four main functions**:

1. **Event Monitoring**: Watches the Gateway contract on Polygon (and optionally BSC/Arbitrum)
   for secret-related events (`SecretCreatedFromGateway`, `SecretUpdatedFromGateway`,
   `SecretCancelledFromGateway`, `FlashSecretCreated`, `VoucherRedeemed`). Each authorized
   relayer instance automatically discovers its shard identity at startup and processes only
   the events assigned to it (`uint256(messageHash) % totalRelayers == myIndex`). If the
   primary owner fails, fallback relayers claim it after a staggered wait time
   (4 + offset minutes), ensuring high availability without thundering-herd collisions.

2. **Cross-chain Delivery**: Submits signed transactions to Oasis Sapphire to create, update,
   or cancel time-locked secrets on behalf of users. The relayer pays ROSE gas upfront and is
   reimbursed at 105% from the on-chain Gas Reserve Pool, providing a small profit margin that
   makes operating a relayer economically sustainable.

3. **Gasless Secret Access**: Allows recipients (heirs) to retrieve unlocked secrets without
   owning ROSE or any other cryptocurrency. The recipient builds an Encrypted Access Envelope
   client-side (Argon2id hash + relayer binding + 8-minute freshness window), submits it to
   this API, and the relayer forwards it to the Sapphire TEE. Gas is paid by the relayer and
   reimbursed from the Gas Reserve Pool.

4. **Voucher Gas Sponsorship**: Non-technical users who purchase USDC credit via fiat vouchers
   (e.g. Stripe checkout) never hold any POL (Polygon gas token). The relayer sponsors 100% of
   Polygon gas for all voucher operations — redemption, secret creation, update, and cancellation.
   This cost is absorbed from the fiat margin on voucher sales. The relayer calls dedicated
   `relayedCreateSecret()`, `relayedUpdateSecret()`, and `relayedCancelSecret()` functions on
   the Gateway contract, deducting payment from the user's persistent `CreditLedger` balance
   rather than their wallet.

## Two Creator Archetypes

- **Crypto-native creator** (has wallet, holds USDC): calls the Gateway contract directly from
  their wallet. The relayer picks up the on-chain event and bridges to Sapphire automatically.
  No interaction with this API is required beyond `/api/secrets/access` at retrieval time.

- **Voucher user** (no wallet, paid with fiat): the Secret Maker app generates a BIP-39 wallet
  for them, loads USDC credit via `VoucherRegistry`, and calls the `/api/voucher/*` endpoints
  on this API. The relayer acts as a complete gas proxy — the user pays zero POL and zero ROSE.

## Staking & Authorization

Before a relayer instance can process events or call `relayedCreate/Update/Cancel`, it must:
1. Complete off-chain KYC review by the team (governance proposal via TimelockMultisig).
2. Stake a minimum of 300 POL on the `RelayerStakingRegistry` contract on Polygon.

The `RelayerStakingRegistry.isOperational(address)` check gates voucher redemption and
all relayed operations. Use `/api/staking/*` endpoints to manage the staking lifecycle.



## Authentication

Most endpoints are public. The following mechanisms are used:
1. **Response Signing (Server Identity)**: Every JSON response (except `/api/health`,
   `/api/metrics`, `/api/metrics/summary`, `/api/bootstrap/snapshot`, `/api/logs/ws`, and
   `/api/update/binary`) carries an ECDSA `signature` field alongside the relayer's wallet
   (`publicKey`) and `version`.
   To verify response authenticity:
   1. Construct the canonical UTF-8 string `{success}|{timestamp}|{data_json}|{error}|{version}`,
      where `{data_json}` is the compact JSON serialization of `data` (or `null` if absent),
      and `{error}` is empty string when absent.
   2. Hash the canonical string with `keccak256` (raw hash, no EIP-191 prefix).
   3. Recover the secp256k1 signer address (`ecrecover`) from the 65-byte `signature` and
      confirm it matches `publicKey` (case-insensitive).
   4. Confirm `publicKey` is an authorized relayer in the on-chain registry and check `timestamp`
      freshness.
2. **Client Identification (Client Identity, reserved)**: The `BearerAuth` JWT
   scheme is reserved for optional client identification (rate-limit tiers,
   auditing). It is not enforced yet - no token-issuance endpoint exists and
   current handlers ignore the `Authorization` header.
3. **Secret Access**: The `/api/secrets/access` endpoint additionally 
   requires a valid Encrypted Access Envelope that binds the recipient's 
   Argon2id password hash to this relayer's address.

## Error Handling

All error responses use a consistent envelope:

```json
{
  "success": false,
  "timestamp": 1710000000,
  "signature": "0x...",
  "publicKey": "0x...",
  "version": "1.0.2",
  "code": "INSUFFICIENT_CREDIT",
  "error": "CreditLedger balance too low for create operation",
  "details": {
    "balance": "5000000",
    "required": "10000000"
  }
}
```

- **`code`** — stable machine-readable value from the `ErrorCode` enum. Clients MUST switch on this field.
- **`error`** — human-readable string for logs and debugging. May change between versions; never parse it.
- **`details`** — optional structured payload specific to the error code.

HTTP status codes remain the primary classification:
- `400` — client errors (validation, bad request, most business-logic failures)
- `401` / `403` — authentication / authorization
- `402` — insufficient credit (always `code: INSUFFICIENT_CREDIT`)
- `404` — resource not found
- `413` — request body too large (global 4 MB cap; 128 KB on voucher and journal write endpoints)
- `429` — rate limited
- `500` / `503` — internal or temporary unavailability

See the `ErrorCode` schema for the complete list of codes and their meanings.

## Rate Limiting

Rate limits may apply per client IP, per wallet, or per secret ID depending on the
endpoint, and per-client limits when a valid `BearerAuth` token is supplied are
reserved for future enforcement (`Authorization` headers are currently accepted and
ignored). Currently enforced server limits:

- **Baseline public API limit** (generated public `/api/*` endpoints plus
  `/api/bootstrap/snapshot-info` and `/api/logs/ws`, excluding `GET /api/health`
  which is unthrottled, and excluding routes with dedicated limiters below):
  **1,200 requests/hour per client IP**. Peer-to-peer routes (`/api/records`,
  `/api/records/sync`, `/api/records/iblt-sync`, `/api/vouchers/sync`,
  `/api/notary/timestamps/sync`, `/api/dead-letter/notify`,
  `/api/bootstrap/snapshot`) share the same bucket with a **10x ceiling
  (12,000 requests/hour)** so a relayer fleet behind one egress IP is never
  mistaken for abuse. Non-OpenAPI documentation/utility routes
  (`/api/sign-auth-request`, `/api/docs*`, `/api/explorer*`, `/api/openapi*.yaml`)
  and `webAuthMiddleware`-protected `/api/database/*` routes do not pass through
  this baseline limiter.
- **Shared rate-limited endpoints** (`/api/voucher/redeem`, `/api/voucher/check`,
  `/api/notary/timestamp`, `/api/wallet/{wallet}/pubkey`, `/api/reprocess/gateway`,
  `/api/gateway/reprocess`, `/api/vouchers/discover`, `/api/update/binary`, and
  `/api/abuse/report`): **50 requests/hour per client IP**.
- **Secret access** (`/api/secrets/access`): **50 requests/hour** on the paid/gas
  path and **1,000 requests/hour** on the cached/already-viewed path, keyed per
  `secretId` and per client IP. The per-`secretId` bucket always enforces; the
  per-IP buckets ship in observe-only mode (see below) and are skipped entirely
  when the client IP resolves to a local reverse proxy.
- **Voucher secret writes** (`/api/voucher/secret/create`, `/update`, `/cancel`):
  **100 requests/hour per `userWallet`** and **300 requests/hour per client IP**;
  **128 KiB** (`131,072` bytes) request body cap (`413`).
- **Journal writes** (`/api/journal/entry`): **50 requests/hour per `secret_id`** and
  **120 requests/hour per client IP**; **128 KiB** (`131,072` bytes) request body cap (`413`).
- **Global request body cap**: **4 MiB** across API routes.

Exceeding a rate limit returns HTTP `429` (`code: RATE_LIMITED`) with a `Retry-After`
response header (integer seconds remaining in the current window, minimum `1`). Clients should honor
`Retry-After` or back off exponentially when retrying `429` responses.

The public baseline limiter ships in **observe-only mode**: over-threshold requests are
counted and logged but still served, so the operator can size the ceiling from real
traffic before enforcing. Set `relayer.baseline_limiter_mode` (env
`RELAYER_BASELINE_LIMITER_MODE`) to `enforce` to activate `429` responses on that
bucket; the current mode and would-block counters are exported by `GET /api/metrics`
(`relayer_limiter_mode`, `relayer_limiter_would_block_total`). Any value other than
`enforce` behaves as `log`. The admin API baseline limiter follows the same pattern
via `relayer.admin_general_limiter_mode` (`RELAYER_ADMIN_GENERAL_LIMITER_MODE`), as
does the secret-access per-IP bucket via `relayer.access_secret_ip_bucket_mode`
(`RELAYER_ACCESS_SECRET_IP_BUCKET_MODE`); the per-`secretId` access bucket is
unaffected by the mode and always enforces.


**Version:** 1.0.2

## Table of Contents

- [/api/blocks](#-api-blocks)
- [/api/contracts/addresses](#-api-contracts-addresses)
- [/api/events](#-api-events)
- [/api/events/secret/{secretId}](#-api-events-secret-secretId)
- [/api/events/{txHash}](#-api-events-txHash)
- [/api/health](#-api-health)
- [/api/income/claim-data](#-api-income-claim-data)
- [/api/income/mark-reimbursed](#-api-income-mark-reimbursed)
- [/api/income/transactions](#-api-income-transactions)
- [/api/journal/entry](#-api-journal-entry)
- [/api/messages](#-api-messages)
- [/api/metrics](#-api-metrics)
- [/api/metrics/summary](#-api-metrics-summary)
- [/api/notary/timestamp](#-api-notary-timestamp)
- [/api/notary/timestamp/{doc_hash}](#-api-notary-timestamp-doc_hash)
- [/api/notary/timestamps](#-api-notary-timestamps)
- [/api/pricing](#-api-pricing)
- [/api/relayers](#-api-relayers)
- [/api/reprocess-pending](#-api-reprocess-pending)
- [/api/reprocess/gateway](#-api-reprocess-gateway)
- [/api/reprocess/{txHash}](#-api-reprocess-txHash)
- [/api/sapphire/public-keys](#-api-sapphire-public-keys)
- [/api/sapphire/secrets/owner](#-api-sapphire-secrets-owner)
- [/api/secrets/access](#-api-secrets-access)
- [/api/secrets/autocomplete](#-api-secrets-autocomplete)
- [/api/secrets/gateway-history/{id}](#-api-secrets-gateway-history-id)
- [/api/secrets/info/{id}](#-api-secrets-info-id)
- [/api/secrets/status](#-api-secrets-status)
- [/api/status](#-api-status)
- [/api/submissions/{submissionId}](#-api-submissions-submissionId)
- [/api/voucher/check](#-api-voucher-check)
- [/api/voucher/credit-balance](#-api-voucher-credit-balance)
- [/api/voucher/redeem](#-api-voucher-redeem)
- [/api/voucher/secret/cancel](#-api-voucher-secret-cancel)
- [/api/voucher/secret/create](#-api-voucher-secret-create)
- [/api/voucher/secret/update](#-api-voucher-secret-update)
- [/api/vouchers](#-api-vouchers)
- [/api/vouchers/batches](#-api-vouchers-batches)
- [/api/vouchers/discover](#-api-vouchers-discover)
- [/api/wallet/{wallet}/messages](#-api-wallet-wallet-messages)
- [/api/wallet/{wallet}/pubkey](#-api-wallet-wallet-pubkey)
- [/api/wallet/{wallet}/transactions](#-api-wallet-wallet-transactions)

## Servers

- https://relayer.athora.me
  TESTNET relayer 1
- https://relayer2.athora.me
  TESTNET relayer 2
- https://relayer3.athora.me
  TESTNET relayer 3

## Authentication

### RelayerAuth
**Type:** `apiKey`
ECDSA signature authentication for relayer-to-relayer requests.
Requires five headers: the three legacy headers below plus `X-Relayer-Nonce` and
`X-Relayer-Binding` (both described in the "Bound signature" section below).
- `X-Relayer-Wallet`: The relayer's wallet address (signer)
- `X-Relayer-Timestamp`: Unix timestamp (must be within 2 minutes)
- `X-Relayer-Signature`: ECDSA signature of `keccak256(timestamp|wallet)`
Operator endpoints (income, restart, sync management, notary reconcile) also
accept these headers as one authentication path; the operator dashboard
alternatively authenticates via its web session cookie, which is not
expressible as an OpenAPI scheme.

Bound signature (REQUIRED on all `RelayerAuth` requests — legacy-only requests are rejected):
- `X-Relayer-Nonce`: 32-hex-char (16-byte) per-request nonce (5-minute replay TTL)
- `X-Relayer-Binding`: ECDSA (personal_sign) signature over
  `chrono-bind-v1|timestamp|wallet|method|path|rawQuery|sha256(body)|nonce`
  (`wallet` is lowercased; `method` is uppercase; `rawQuery` is the raw query string
  without `?` or empty string when absent; `sha256(body)` is lowercase hex SHA-256 of
  the raw body bytes, `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
  for an empty body).
Verifiers fail closed (`401`) on missing, invalid, or replayed bindings (`REPLAY_ATTEMPT`
on replayed nonces, `REPLAY_CACHE_SATURATED` if the nonce cache is saturated).

**Location:** `header`

### BearerAuth
**Type:** `http`
Reserved for optional client identification (e.g., SecretMaker PWA, Vault SDK)
for client identification, usage tracking, and enhanced rate limiting. NOT enforced
today: no token-issuance endpoint exists and current handlers accept and ignore
this header. Clients may send it for forward compatibility; never rely on it for
authorization.

**Scheme:** `bearer`

## Tags

### Status & Monitoring
Service status, health, network configuration, and performance metrics.

### Events
Gateway event monitoring, history, and delivery tracking across chains.

### Secrets
Gasless secret access for heirs and recipients (crypto-native path).

### Vouchers
Fiat voucher redemption, credit balance management, and relayed secret operations.

### Sapphire
Oasis Sapphire TEE interaction endpoints and public key retrieval.

### Relayers & Staking
Relayer registry, health monitoring, and staking lifecycle management.

### Income
Relayer gas reimbursement tracking and reward analytics.

### Messages
Wallet message inbox — encrypted messages sent via the Sapphire vault.

### Notary
Decentralized document timestamping on Polygon via ChronoloxNotary contract.

### Journal
Secret Journal entries — encrypted, append-only diary records anchored on-chain via the relayer.

### Pricing
Protocol operation pricing (Credit/USDC prices per operation type).

## Endpoints

### /api/blocks

#### GET /api/blocks
**Get Block Information**

Returns the latest known block numbers for Oasis Sapphire and each monitored EVM Gateway
chain. Useful for dashboards and monitoring tools to quickly verify the relayer is keeping
up with chain head and has not stalled behind a reorg or RPC outage.


**Responses:**

**200**
Block information retrieved successfully


**400**

**429**

**500**

**Tags:** Status & Monitoring

---

### /api/contracts/addresses

#### GET /api/contracts/addresses
**Get contract addresses for a blockchain**

Returns the Gateway and PricingRegistry contract addresses for a specified blockchain.
The Gateway address is discovered dynamically from the Sapphire contract's `authorizedGateways()`
mapping, while the PricingRegistry address is returned from SDK configuration (if deployed).

**Supported networks:** polygon, polygonAmoy, bsc, bscTestnet


**Parameters:**

- `blockchain` (query, `string`) (optional)
  Blockchain network identifier
  **Enum:** `polygon`, `polygonAmoy`, `bsc`, `bscTestnet`

**Responses:**

**200**
Contract addresses retrieved successfully


**400**

**429**

**500**

**Tags:** Status & Monitoring

---

### /api/events

#### GET /api/events
**List Processed Events**

Returns Gateway events that this relayer has processed or is tracking, along with a list
of empty blocks that were scanned. The `scanned_blocks` array lets peer relayers skip
RPC queries for blocks already confirmed to contain no relevant events, reducing
unnecessary Polygon/BSC node load during P2P sync.

Each event record carries `is_flash` and `is_voucher` flags so consumers can distinguish
protocol-paid secrets from flash secrets (one per wallet, capped at 7 days) and
voucher-credit secrets (paid via fiat CreditLedger balance rather than a direct USDC
wallet transfer).

Supported event names: `SecretCreatedFromGateway`, `SecretUpdatedFromGateway`,
`SecretCancelledFromGateway`, `SecretMetadataStored`, `SecretMetadataUpdated`,
`FlashSecretCreated`, `VoucherRedeemed`.


**Parameters:**

- `status` (query, `string`) (optional)
  Filter by event status
  **Enum:** `pending`, `delivered`, `failed`, `permanently_failed`
- `secret_id` (query, `string`) (optional)
  Filter by secret ID (hex string, 1-64 chars)
- `chain` (query, `unknown`) (optional)
  Filter by blockchain network
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
List of event records


**400**

**429**

**500**

**Tags:** Events

---

### /api/events/secret/{secretId}

#### GET /api/events/secret/{secretId}
**Get Events by Secret ID**

Returns all events for a given on-chain secret ID (bytes32 hex). Because a single secret
can generate multiple lifecycle events (creation, one or more updates, cancellation), this
endpoint provides a complete audit trail. The secret ID is truncated to uint64 internally
for Sapphire compatibility. Pass between 1 and 64 hex characters; the API pads or truncates
as needed.


**Parameters:**

- `secretId` (path, `string`) (required)
  Hex-encoded secret ID (1-64 characters)
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
List of events for the secret (may be empty)


**400**
Invalid secret ID format


**429**

**500**

**Tags:** Events

---

### /api/events/{txHash}

#### GET /api/events/{txHash}
**Get Event by Transaction Hash**

Retrieves all event records associated with a single Polygon (or BSC/Arbitrum) transaction
hash. A transaction may emit more than one event (e.g. `SecretMetadataStored` alongside
`SecretCreatedFromGateway`). The `sapphire_tx_hash` field is populated once the relayer
has successfully bridged the event to Oasis Sapphire. Poll this endpoint after submitting
a direct-wallet or voucher secret operation to confirm Sapphire delivery.


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
Event details


**400**

**429**

**500**

**Tags:** Events

---

### /api/health

#### GET /api/health
**Health Check**

Simple liveness probe that returns {"ok":true}. This endpoint has NO middleware,
NO authentication, NO response envelope, NO database query, and NO signature.
It's designed for container orchestrators (Kubernetes, Docker) to check if the
relayer process is running.


**Responses:**

**200**
Service is healthy
  **Type:** `object`
  **Properties:**
  - `ok`: `boolean`
    True when the relayer HTTP process is alive

**400**
Bad request
  **Type:** `object`
  **Properties:**
  - `error`: `string`
    Error message describing the invalid request

**Tags:** Status & Monitoring

---

### /api/income/claim-data

#### GET /api/income/claim-data
**Get claim data for wallet-signed reimbursement**

Requires operator authentication (session cookie or signed operator headers). Not intended for public clients.

Returns the range claim data, checkpoints, and contract details needed to build a
claimReimbursement transaction that will be signed by the user's wallet (MetaMask/Rabby).

Supports both Sapphire (ROSE) and Polygon (POL) reimbursement pools.


**Parameters:**

- `chain` (query, `string`) (optional)
  Chain to claim reimbursement on (sapphire or polygon)
  **Enum:** `sapphire`, `polygon`

**Responses:**

**200**
Claim data for wallet-signed reimbursement
  **Properties:**
  - `data`: `object`
    Checkpoint range, calldata, and pool details for submitting a reimbursement claim

**400**

**401**

**429**

**500**

**Tags:** Income

---

### /api/income/mark-reimbursed

#### POST /api/income/mark-reimbursed
**Mark transactions as reimbursed after wallet-signed claim**

Requires operator authentication (session cookie or signed operator headers). Not intended for public clients.

**Deprecated (operator-only):** This endpoint is operator-only, has been copied to
the private admin specification (`openapi.admin.yaml`), and will be removed from the
public specification once existing operator dashboard / SDK consumers migrate.

After the user signs and submits a claimReimbursement transaction via their wallet,
this endpoint marks the corresponding database records within [start_block, end_block]
as reimbursed.


**Request Body:**

**Content-Type:** `application/json`
  **Type:** `object`
  **Properties:**
  - `chain`: `string`
    Chain where claim was submitted
  - `tx_hash`: `string` (required)
    Hash of the confirmed claimReimbursement transaction
  - `start_block`: `integer` (required)
    Start block number of claimed range
  - `end_block`: `integer` (required)
    End block number of claimed range

**Responses:**

**200**
Transactions marked as reimbursed
  **Properties:**
  - `data`: `object`
    Result of marking claimed transactions as reimbursed

**400**

**401**

**429**

**500**

**Tags:** Income

---

### /api/income/transactions

#### GET /api/income/transactions
**Get relayer income transactions**

Requires operator authentication (session cookie or signed operator headers). Not intended for public clients.

Returns transactions signed by this relayer's wallet on the requested chain, showing gas
costs, event types, and reimbursement status. Includes summary KPIs (total gas cost, 5%
reward earned, reimbursed amount, pending reimbursement) covering both chains.

The relayer earns a 5% reward on all gas fees for transactions it signs
(create, update, cancel, access operations) on Sapphire (ROSE) and Polygon (POL).


**Parameters:**

- `chain` (query, `string`) (optional)
  Which chain's transactions to list (sapphire = ROSE gas, polygon = POL gateway gas). KPIs always cover both chains.
  **Enum:** `sapphire`, `polygon`
- `from_date` (query, `integer`) (optional)
  Unix timestamp start filter (inclusive, 0 = no filter)
- `to_date` (query, `integer`) (optional)
  Unix timestamp end filter (inclusive, 0 = no filter)
- `event_type` (query, `string`) (optional)
  Filter by event type (Sapphire vault or gateway operation)
  **Enum:** `created`, `accessed`, `extended`, `cancelled`, `updated`, `admin`, `reimbursement`, `journal`, `notary`
- `tx_status` (query, `string`) (optional)
  Filter by transaction status (empty = all). 'failed' means the transaction reverted on-chain.
  **Enum:** `success`, `failed`, `pending`
- `claimable` (query, `boolean`) (optional)
  When true, only return transactions eligible for reimbursement (on-chain OpRecord exists, not yet reimbursed, status success).
- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
Income transactions with KPIs
  **Properties:**
  - `data`: `object`
    Income transaction list, summary KPIs, and current wallet balances

**400**

**401**

**429**

**500**

**Tags:** Income

---

### /api/journal/entry

#### POST /api/journal/entry
**Append Encrypted Entry to Secret Journal**

Appends an encrypted journal entry to a secret's journal on Polygon on behalf of a
voucher user. The relayer sponsors POL gas and deducts the journal message fee
from the user's CreditLedger balance.

The fee is calculated per 5 KB chunk:
`chunks = ceil(len(payload) / 5120)`
`cost = chunks * pricingRegistry.journalMessagePriceUsd()` (default 1 Credit per chunk).

EIP-712 signature verification recovers the signer address, which must match the
secret owner on SecretVaultSapphireGateway. The secret must be active (not canceled),
mutable (not frozen), not a flash secret, and not expired (block.timestamp < unlockTime).

## Effective size limit

The server enforces a 128 KiB cap on the entire request body of this endpoint (checked
before validation) and returns HTTP 413 when it is exceeded. Because the fee is computed
per 5 KB chunk of `payload`, payloads that would push the body past the cap are rejected
with 413 rather than priced.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Journal entry appended successfully


**400**

**402**

**409**
Journal append conflict (audit G1, 2026-10-17): the server-side
reservation for the (secret_id, entry_index) slot was already
claimed. Error code is REPLAY_ATTEMPT. Details keys:
`reason` (`in_flight` — a pending append holds the slot, retry
after it expires or the signed deadline passes; or
`index_consumed` — the index was consumed by a different signed
payload, refetch the journal and sign the next index). A client
retry with the identical signed payload after a successful
append instead receives 200 with the original tx hash
(idempotent).



**413**

**429**

**500**

**Tags:** Journal, Vouchers

---

### /api/messages

#### GET /api/messages
**List All Messages**

Global paginated list of all messages captured by this relayer, newest first.
Used by the registry-tool management page. Supports optional sender/recipient
filters (checksummed or lowercase 0x addresses).


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)
- `sender` (query, `string`) (optional)
  Filter by sender address (recovered message owner)
- `recipient` (query, `string`) (optional)
  Filter by recipient address

**Responses:**

**200**
Paginated message list


**400**

**429**

**500**

**Tags:** Messages

---

### /api/metrics

#### GET /api/metrics
**Prometheus Metrics**

Exposes runtime metrics in Prometheus text format for scraping by a Prometheus server
or compatible tool (Grafana Agent, VictoriaMetrics, etc.). Suitable for setting up
alerting rules on ROSE/POL balance thresholds, event delivery failure rates, and
relayer uptime. Use `/api/metrics/summary` for the pre-aggregated JSON dashboard view.


**Responses:**

**200**
Metrics in Prometheus format
  **Type:** `string`

**400**

**429**

**500**

**Tags:** Status & Monitoring

---

### /api/metrics/summary

#### GET /api/metrics/summary
**Get relayer performance metrics**

Returns performance metrics for monitoring dashboards including:
- KPIs (24h call volume, success rates, latency)
- Time-series call volume data
- Breakdown by blockchain
- Breakdown by peer

For operational health (status, balances, gateway info), use `/api/status`.

**Response format:** Returns pre-aggregated dashboard metrics without the standard JSON
envelope, so the relayer's own frontend can consume them directly without an extra
unwrapping step.

**KPI fields:**
- `total_calls_24h` — all inbound + outbound API calls in the last 24 hours.
- `success_rate_1h` / `success_rate_24h` — fraction of calls that returned a non-error
  response, over 1-hour and 24-hour windows respectively.
- `median_latency_ms_5m` — p50 response latency over the last 5 minutes; useful for
  detecting Sapphire RPC degradation before it causes delivery failures.
- `active_peers_1h` — number of peer relayers that have successfully exchanged P2P
  records with this instance in the last hour.

**Time-series data:** `call_volume` returns per-bucket inbound/outbound/failed call
counts. The bucket interval is determined by the relayer's internal aggregation window.

**Per-blockchain and per-peer breakdowns** allow operators to identify which Gateway
chain or peer relayer is contributing disproportionate failures.


**Parameters:**

- `range` (query, `integer`) (optional)
  Time range in seconds to aggregate metrics over

**Responses:**

**200**
Metrics summary data


**400**

**429**

**500**

**Tags:** Status & Monitoring

---

### /api/notary/timestamp

#### POST /api/notary/timestamp
**Anchor a document hash to the blockchain via Voucher credit**

Allows a voucher user to publicly timestamp a document hash on Polygon.
The relayer acts as the gas sponsor. The fee is deducted from the user's CreditLedger.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Successfully anchored


**400**

**402**

**429**

**500**

**Tags:** Notary

---

### /api/notary/timestamp/{doc_hash}

#### GET /api/notary/timestamp/{doc_hash}
**Get details of a single notary timestamp record by SHA3-512 hash**

Returns details of a single anchored document timestamp record from the relayer's database.
Accepts the original 64-byte (128 hex characters) SHA3-512 document hash, computes its Keccak256
on the backend, and queries the local database.


**Parameters:**

- `doc_hash` (path, `string`) (required)
  The original 64-byte SHA3-512 document hash

**Responses:**

**200**
Notary timestamp record details


**400**

**404**
Notary timestamp record not found


**429**

**500**

**Tags:** Notary

---

### /api/notary/timestamps

#### GET /api/notary/timestamps
**List timestamp notary records**

Returns a paginated list of anchored document timestamp records from the relayer's
local database. Each record represents a document hash that was publicly
timestamped on Polygon via the ChronoloxNotary contract.

Supports optional filtering by `user_wallet` and `network`. Results are ordered
by `block_number` descending (most recent first).


**Parameters:**

- `author` (query, `string`) (optional)
  Filter by wallet address of the user who authorized the timestamp
- `chain` (query, `string`) (optional)
  Filter by blockchain network
  **Enum:** `polygon`, `polygonAmoy`
- `unknown` (unknown, `unknown`) (optional)
  Maximum number of records to return
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
List of timestamp notary records


**400**

**429**

**500**

**Tags:** Notary

---

### /api/pricing

#### GET /api/pricing
**Get system operation pricing**

Returns the current operation prices (USD-denominated, 6-decimal raw
integers) from the PricingRegistry contract on the requested Polygon network,
including the Flash Secret price.

Flash Secrets are **NOT** priced via `calculatePrice()` (which reverts on
`"flash"`); their price is read from `getFlashConfig()`. This endpoint exposes
all prices in a single call so clients can display them without each calling
the contract directly.

## Parameters

- `network`: Polygon network (polygonAmoy or polygon) (required)

## Response (`data`)

- `network`: Network identifier
- `prices`: Object of operation prices in USDC (6 decimals, raw integer strings)
  - `create`: base create price for 1 heir
  - `create_per_additional_heir`: price added per additional heir
  - `update`: update/extend price
  - `cancel`: cancel price
  - `timestamp`: notary timestamp price
  - `flash`: flash secret price
  - `journal`: journal message price per 5KB chunk
- `flash_max_duration_seconds`: Maximum flash secret lifetime (seconds)

If the network has no PricingRegistry deployed, the endpoint still returns HTTP 200
with an empty `prices` object and `flash_max_duration_seconds: 0` (no `cache` fields).


**Parameters:**

- `network` (query, `unknown`) (required)
  Network to read pricing from

**Responses:**

**200**
Pricing retrieved successfully


**429**

**500**

**Tags:** Pricing

---

### /api/relayers

#### GET /api/relayers
**List Known Relayers**

Returns all relayer instances this relayer knows about, discovered via the on-chain
`RelayerStakingRegistry` on Polygon. At startup, each relayer queries
`getOperationalRelayers()` on the registry, retrieves each operator's `name` and
`endpoint`, health-checks them, and stores them locally.

Use `healthyOnly=true` to filter out relayers that have not responded to a health check
within the last polling cycle. The `is_outdated` flag indicates the relayer is running a
binary version older than the most common version advertised by healthy peers (determined
by the P2P auto-update quorum mechanism).

This list is the authoritative source for P2P sync target selection — both event records
and voucher records are exchanged with the healthy peers returned here.


**Parameters:**

- `healthyOnly` (query, `boolean`) (optional)
  When true, return only peers currently marked healthy by health checks

**Responses:**

**200**
Relayer list retrieved


**400**

**429**

**500**

**Tags:** Relayers & Staking

---

### /api/reprocess-pending

#### POST /api/reprocess-pending
**Process All Pending Events**

Requires operator authentication (session cookie or signed operator headers). Not intended for public clients.

**Deprecated (operator-only):** This endpoint is operator-only, has been copied to
the private admin specification (`openapi.admin.yaml`), and will be removed from the
public specification once existing operator dashboard / SDK consumers migrate.

Immediately processes all pending events, bypassing the normal polling cycle.
Pending events are those detected on source chains but not yet delivered to Sapphire.
This endpoint forces immediate processing with detailed per-event results.

Unlike the normal polling cycle, this bypasses shard ownership checks and processes
all pending events in the same HTTP request. Returns detailed results for each event.


**Responses:**

**200**
All pending events processed with detailed results.


**400**

**401**

**429**

**500**

**Tags:** Events

---

### /api/reprocess/gateway

#### POST /api/reprocess/gateway
**Force reprocessing/capture of a gateway transaction to Sapphire**

Forces the relayer to capture a gateway transaction from the blockchain if not yet in the database,
determine if it needs to be processed to Sapphire, and if so, submit it to Sapphire.

### Direct Mode Workflow & Load Balancing
1. **Direct Gateway Execution**: The user/CLI performs a create, update, or cancel operation directly on the Polygon Gateway, paying the transaction gas locally.
2. **Random Relayer Selection**: To ensure fair load distribution and prevent overloading a single relayer, the CLI queries the SDK registry for all active/operational relayers, and randomly selects one.
3. **Reprocess Trigger**: The CLI sends a `POST` request to the selected relayer's `/api/reprocess/gateway` endpoint with the `secretId` and the `txHash` (Gateway transaction ID).
4. **Shard Bypass & Authorization**: Normally, relayers coordinate processing priority using a sharding scheme. Calling this endpoint on the selected relayer forces it to bypass its shard priority check and immediately assume responsibility for executing the Sapphire transaction.
5. **Validation and Execution**: The relayer retrieves the transaction receipt from Polygon to validate it. It then checks on-chain (using read-only Sapphire queries) to verify if the Sapphire transaction already exists. If not, it executes the confidential Sapphire transaction.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Force reprocessing completed successfully


**400**

**429**

**500**

**Tags:** Events

---

### /api/reprocess/{txHash}

#### POST /api/reprocess/{txHash}
**Reprocess Event**

Marks a single failed event as `pending` so it will be retried on the next polling cycle.
Use this when a Sapphire RPC outage or insufficient ROSE balance caused delivery failure
but the underlying issue has since been resolved. For bulk recovery of all failed
events, operators use the force-reprocess admin endpoint instead.


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
Reprocessing triggered


**400**

**429**

**500**

**Tags:** Events

---

### /api/sapphire/public-keys

#### GET /api/sapphire/public-keys
**Get TEE public keys for encryption**

Returns the X25519 and ML-KEM-1024 public keys from the Sapphire TEE.
Required for encrypting secret content before creating a secret.

The X25519 key is used for NaCl-style encryption (X25519 + XSalsa20-Poly1305).
The ML-KEM-1024 key is used for hybrid KEM encryption (Kyber1024 + X25519).

Returns cached values with a 5-minute freshness guarantee.

This endpoint is public and does not require any authentication.

## Usage

Clients call this endpoint before creating a secret to obtain the current
public keys for encrypting the secret content. The keys are fetched from the
Sapphire contract's `getPublicKeys()` method.

The `x25519_key` is always present (32 bytes). The `kyber_key` may be
empty or null if the Sapphire contract doesn't support post-quantum
encryption yet.


**Responses:**

**200**
Public keys retrieved successfully


**429**

**500**

**Tags:** Sapphire

---

### /api/sapphire/secrets/owner

#### GET /api/sapphire/secrets/owner
**List all secret IDs for an owner address**

Returns all secret IDs (bytes32 values) that were created by the given owner address
on the Sapphire contract. This is a direct proxy to the Solidity view function
`listSecretsForOwner(address owner)`.

The endpoint queries the Sapphire contract's `_ownerSecrets` mapping, which stores
an array of secret IDs for each owner address that has created secrets.

This is useful for:
- Wallet apps to show all secrets created by a user
- Dashboard views of user's secret portfolio
- Auditing/verification of secret ownership

## Usage

Pass the owner's Ethereum address as a query parameter. The address must be a
valid 20-byte Ethereum address (0x-prefixed hex string).

Returns an empty array if the owner has no secrets or the address has never
created any secrets on Sapphire.

This endpoint is public and does not require any authentication.


**Parameters:**

- `owner` (query, `string`) (required)
  Owner address to query (20-byte Ethereum address)

**Responses:**

**200**
Secret IDs retrieved successfully


**400**

**429**

**500**

**Tags:** Sapphire

---

### /api/secrets/access

#### POST /api/secrets/access
**Gasless Secret Access**

Core endpoint for recipient (heir) access to a secret. Enables retrieval of the encrypted
secret payload and associated encrypted Shamir shares without requiring the recipient
to hold cryptocurrency or interact directly with the blockchain.

This endpoint implements a password-based authentication scheme combined with
Trusted Execution Environment (TEE) verification and relayer-mediated gas abstraction.

## Cryptographic model

Each recipient is assigned:
- A password (distributed off-chain by the creator)
- Two salts:
  - `salt_A` (authentication)
  - `salt_S` (share decryption)

Authentication is based on:
  Auth = Argon2id(password, salt_A)
  Stored on-chain as: keccak256(Auth)

Shamir shares are protected independently using:
  key = Argon2id(password, salt_S)

The relayer and blockchain never receive the password or decryption keys.

## Access flow

1. The client retrieves public parameters (`salt_A`, `salt_S`) from the relayer or chain.

2. The client derives:
    argon2Hash = Argon2id(password, salt_A)

3. The relayer performs preliminary checks:
    - Unlock time pre-validation
    - Rate limiting
    - Request structure validation

  The relayer does not verify the password or derive any secrets.

4. The relayer forwards the encrypted payload to the Sapphire contract
  (`accessSecret`) and pays the execution gas.

5. Inside the TEE, the following validations are enforced:

  - Authentication check:
      keccak256(argon2Hash) == stored Auth hash

6. If all checks pass, the TEE returns:
    - The encrypted secret payload
    - The set of encrypted Shamir shares `[Enc(s₁), Enc(s₂), ...]`

## Client-side reconstruction

After receiving the response, the client:

1. Derives the share decryption key:
    key = Argon2id(password, salt_S)

2. Attempts decryption of each encrypted share:
    Enc(sᵢ) → sᵢ (only one will succeed)

3. Collaborates off-chain with other recipients (if required by threshold)
  to reconstruct the original `content_key` via Shamir.Combine.

4. Decrypts the secret using the reconstructed `content_key`.

## Security properties

- The password is never transmitted or exposed.
- Authentication and decryption use separate salts and derivations.
- The relayer cannot impersonate recipients or derive shares.
- Requests are bound to a specific relayer instance.
- Secret access is enforced inside a TEE, not at the API layer.

## Relayer responsibilities

- Gas sponsorship for TEE execution
- Request forwarding and availability
- Basic anti-abuse protections:
  - Rate limiting (per recipient / vault)
  - Unlock time pre-check
  - Access count limits


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Secret content retrieved


**400**

**429**

**500**

**Tags:** Secrets

---

### /api/secrets/autocomplete

#### GET /api/secrets/autocomplete
**Autocomplete Secret IDs**

Returns a list of up to 6 secret IDs that start with the provided prefix.
A minimum of 2 characters (not including 0x) is required for the prefix.
This is used for autocomplete functionality in the recipient's UI.

**Caching:** Responses are cached for 3 minutes (180 seconds). Within this
window, identical requests return the same signed response including the
original timestamp. This reduces database queries and ECDSA signing overhead.
The cache key includes the full query string, so different prefixes are
cached independently.


**Parameters:**

- `prefix` (query, `string`) (required)
  The prefix to match secret IDs against (minimum 2 characters).

**Responses:**

**200**
List of matching secret IDs


**400**

**429**

**500**

**Tags:** Secrets

---

### /api/secrets/gateway-history/{id}

#### GET /api/secrets/gateway-history/{id}
**Get secret gateway history**

Returns the gateway transaction history for a secret by calling the Sapphire contract's
`getSecretHistory(secretId, 0, 10)` function. This provides the complete cross-chain
operation history including gateway transaction IDs, operation types, timestamps,
gateway message hashes, and chain IDs.

The history includes all operations performed on the secret:
- Create operations (from Gateway)
- Update operations (from Gateway)  
- Cancel operations (from Gateway)
- Access operations (on Sapphire)
- Destroy operations (on Sapphire)

Each history record contains:
- `op_type`: Operation type (0=Access, 1=Create, 2=Update, 3=Cancel, 4=Destroy)
- `timestamp`: Block timestamp when the operation occurred
- `actor`: Address that performed the operation (relayer or direct caller)
- `gateway_message_hash`: Gateway message hash for cross-chain correlation
- `gateway_chain_id`: Origin EVM chain ID (e.g., 80002 for Polygon Amoy)


**Parameters:**

- `id` (path, `string`) (required)
  The secret ID (hex string, with or without 0x prefix)

**Responses:**

**200**
Secret gateway history retrieved successfully


**400**

**429**

**500**

**Tags:** Secrets

---

### /api/secrets/info/{id}

#### GET /api/secrets/info/{id}
**Get secret info by ID**

Returns detailed information about a secret including the secretSalt.
The secretSalt is a 16-byte hex string used for deriving recipient-specific
salts in multi-recipient secrets.

This endpoint queries the events database for the most recent event
matching the given secret ID.

## On-demand peer resolution & recursion guard

If the secret is missing locally or only stored with a placeholder transaction
hash, the relayer resolves it on demand from eligible peer relayers
(`GET /api/records` first, then `GET /api/secrets/info/{id}` as fallback) and
persists the recovered event before answering. The whole resolution is bounded
by a 3-second timeout; each individual peer call by 5 seconds.

**Relayer-to-relayer callers MUST send `X-No-Peer-Resolve: true`.** A receiving
relayer that sees this header (or valid relayer signature headers) serves local
data only and never triggers its own peer resolution — this breaks the mutual
recursive fan-out loop that previously exhausted origin connections when
multiple relayers queried a secret none of them held. End-user clients never
send the header and get the full on-demand resolution behavior.


**Parameters:**

- `id` (path, `string`) (required)
  The secret ID (hex string, with or without 0x prefix)

**Responses:**

**200**
Secret info retrieved successfully


**404**
Secret not found


**429**

**500**

**Tags:** Secrets

---

### /api/secrets/status

#### GET /api/secrets/status
**Get Secret Status from Sapphire**

Queries the SecretVaultSapphire contract directly (not the local database) to
check if a secret exists on-chain and its current lifecycle status.

Calls the Solidity view function `getSecretStatus(bytes32)` which returns:
- `exists`: whether a secret with this ID has ever been created
- `status`: the current SecretStatus enum value

**SecretStatus enum:**
- `0` = Locked (initial state after creation, unlock time not yet reached)
- `1` = Unlocked (unlock time reached, secret is retrievable)
- `2` = Cancelled (cancelled by owner before unlock)
- `3` = Viewed (secret content was accessed/read by a recipient)
- `4` = WindowExpired (access window deadline has passed)

**⚠️ Gas-saving pre-flight check:** Always call this endpoint **before**
attempting any of these gas-expensive operations:
- `POST /api/secrets/access` (costs ROSE gas on Sapphire)
- `POST /api/voucher/secret/cancel` (costs POL + ROSE gas)
- `POST /api/voucher/secret/update` (costs POL + ROSE gas)
- `POST /api/reprocess/{txHash}` (costs ROSE gas)

If `exists` is `false`, there is no secret on Sapphire — skip the
expensive operation and avoid wasting gas. If `status` is `Cancelled`,
`Viewed`, or `WindowExpired`, certain operations may fail or be pointless.

This endpoint makes an `eth_call` (a free read) to Sapphire. It does NOT
query the relayer's local event database, so it reflects the authoritative
on-chain state even if the relayer has not yet synced the event.

**⚠️ Cross-chain delay:** Secret operations (create, update, cancel,
access) originate on a Gateway chain (Polygon/BSC/Arbitrum) and are
only relayed to Sapphire asynchronously by the relayer. This means the
Sapphire status may lag behind the Gateway state by up to a few minutes:
- A secret cancelled on the Gateway may still show `status: 0` (Locked)
  on Sapphire until the relayer processes the cancellation event.
- A secret created on the Gateway may show `exists: false` until the
  relayer delivers it to Sapphire.

If you need the Gateway-side status (i.e. the Polygon event history),
use `/api/events/secret/{secretId}` or `/api/secrets/info/{id}` instead,
which query the relayer's local event database.


**Parameters:**

- `secretId` (query, `string`) (required)
  Secret ID (bytes32 hex, 64 hex chars + 0x prefix)

**Responses:**

**200**
Secret status retrieved successfully


**400**

**429**

**500**

**Tags:** Secrets

---

### /api/status

#### GET /api/status
**Get relayer operational status**

Returns operational health information about the relayer including:
- Uptime and version information
- Relayer wallet address and balances
- Gateway monitoring status
- Staking information
- Event processing statistics

For performance metrics (call volume, success rates, peer performance),
use `/api/metrics/summary`.

**Binary hash (P2P auto-update):** The `binary_hash` field is the SHA-256 of the running
executable. Peer relayers include this in their quorum check before auto-updating: a minimum
of MAX(1, MIN(2, peers)) must advertise the same `(version, binary_hash)` tuple before an
update is downloaded and applied.

**Relayer staking:** The `relayer_staking` block reports the on-chain state from the
Polygon-side `RelayerStakingRegistry` contract: minimum required stake, total POL currently
locked across all operators, and how many relayers are in OPERATIONAL state.

**Wallet balances:** The `rose_balance` and `pol_balance` fields are the relayer's own
wallet balances. `rose_balance` funds Sapphire gas for bridging secrets. `pol_balance` funds
Polygon gas for voucher operations (relayed create/update/cancel/redeem). Alert thresholds are
configurable via `TELEGRAM_ROSE_WARN_THRESHOLD` / `TELEGRAM_POL_WARN_THRESHOLD`.


**Responses:**

**200**
Current status information retrieved successfully


**400**

**429**

**500**

**Tags:** Status & Monitoring

---

### /api/submissions/{submissionId}

#### GET /api/submissions/{submissionId}
**Get Submission Status**

Returns the status of a voucher secret submission (create/update/cancel). Use this endpoint to track
the progress of a cross-chain secret operation from the initial Polygon submission through to the
final Sapphire delivery. The submissionId is returned in the response from voucher operations:
- Create: `{network}-{timestamp}` (e.g., "polygonAmoy-20240307-153000")
- Update: `update-{txHash[:16]}`
- Cancel: `cancel-{txHash[:16]}`


**Parameters:**

- `submissionId` (path, `string`) (required)
  Submission ID from voucher operation response

**Responses:**

**200**
Submission status retrieved


**404**
Submission not found


**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/check

#### GET /api/voucher/check
**Check Voucher Code (Pre-Validation)**

Pre-validates a voucher code before redemption. Performs a read-only
`checkVoucher()` call on the VoucherRegistry contract to verify the code
is valid, unused, and not expired.

This endpoint is intended for client-side validation before the user
initiates the actual redemption flow. It allows the app to display
voucher information (amount, expiry) and appropriate error messages
without burning the code or requiring blockchain RPC knowledge.

**Rate limit:** 10 checks per minute per IP address to prevent enumeration.

**Response fields:**
- `valid`: true if code exists, unused, and not expired
- `amount_usdc`: voucher amount in USDC (6 decimals), only if valid
- `expiry_timestamp`: Unix timestamp when voucher expires, only if valid
- `already_redeemed`: true if code was already redeemed
- `expired`: true if current time > expiry_timestamp


**Parameters:**

- `code` (query, `string`) (required)
  Voucher code (e.g., "CLX-0D8B-BF72-45D8-4A8B")
- `network` (query, `unknown`) (required)
  Network to check voucher on

**Responses:**

**200**
Voucher check result


**400**

**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/credit-balance

#### GET /api/voucher/credit-balance
**Get voucher credit balance**

Returns the current credit balance for a wallet from the CreditLedger contract,
along with the required amount for a specific operation.

Use this endpoint to:
- Display persistent balance widget in UI
- Pre-flight check before creating/updating/cancelling secrets
- Show "insufficient credit" warnings before operations

The endpoint queries the CreditLedger contract for the current balance and
the PricingRegistry contract for the required amount based on operation type
and number of heirs.

## Parameters

- `userWallet`: The wallet address to check balance for (required)
- `network`: Polygon network (polygonAmoy or polygon) (required)
- `operation`: Operation type (create, update, cancel) (optional, default: create)
- `numHeirs`: Number of heirs for pricing calculation (optional, default: 1)

## Response

- `balance`: Current credit balance in USDC (6 decimals)
- `required`: Required amount for the operation in USDC (6 decimals)
- `sufficient`: Whether balance is sufficient for the operation
- `has_voucher_system`: Whether the network has a deployed voucher system

If the network doesn't support the voucher system, the endpoint still returns HTTP 200
with `balance: "0"`, `required: "0"`, `sufficient: false`, and `has_voucher_system: false`.


**Parameters:**

- `userWallet` (query, `string`) (required)
  User's wallet address
- `network` (query, `unknown`) (required)
  Network to check balance on
- `operation` (query, `string`) (optional)
  Operation type for required balance calculation
  **Enum:** `create`, `update`, `cancel`, `journal`
- `numHeirs` (query, `integer`) (optional)
  Number of heirs for pricing calculation

**Responses:**

**200**
Balance retrieved successfully


**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/redeem

#### POST /api/voucher/redeem
**Redeem Voucher**

Redeems a fiat-purchased voucher code and loads the equivalent USDC credit into the
user's on-chain `CreditLedger` balance. This is the entry point for the fiat-to-credit
flow: a non-technical user buys a voucher via Stripe, receives a code by email (format
`CLX-XXXX-XXXX-XXXX-XXXX`, 128-bit entropy), enters it in the Secret Maker app, and
calls this endpoint.

**Sapphire TEE encryption (v0.9):** The voucher code MUST be encrypted client-side using
`EncryptForSapphire()` before being sent to this endpoint. The encrypted payload is a
hex-encoded X25519+Deoxys-II envelope that can only be decrypted inside the Sapphire TEE.
The relayer forwards the payload to `SecretVaultSapphire.tssDecryptCode()`, receives
the plaintext code back via the `VoucherCodeDecrypted` event (visible only to the relayer),
and then calls `VoucherRegistry.redeem()` on Polygon. The relayer never sees the plaintext
voucher code.

**On-chain hashing:** The `VoucherRegistry.redeem(string,address)` function hashes the
plaintext code on-chain via `keccak256(abi.encodePacked(code))` rather than accepting a
pre-computed `bytes32` hash — preventing a compromised relayer from sweeping unredeemed
vouchers by substituting arbitrary code hashes.

The relayer:
1. Forwards the encrypted payload to the Sapphire TEE via `tssDecryptCode()`.
2. Parses the `VoucherCodeDecrypted` event to obtain the plaintext code.
3. Calls `VoucherRegistry.redeem(code, wallet)` on Polygon with the raw code string,
   **sponsoring 100% of POL gas** from its own wallet. The user holds zero POL.
4. The `VoucherRegistry` contract hashes the code and marks it as redeemed, then calls
   `CreditLedger.credit(wallet, amount)`, loading USDC into the user's persistent balance.
5. The user can now create, update, and cancel secrets by calling `/api/voucher/secret/*`
   endpoints — all gas is sponsored by the relayer, deducting from this credit balance.

**Rate limit:** 3 redemption attempts per hour per wallet address to prevent code brute-force.

**Credit persistence:** The `CreditLedger` balance is on-chain and permanent. Unused
credit carries over indefinitely. Topping up is possible by redeeming a second voucher.

**Staking requirement:** Only relayer instances in OPERATIONAL state (staked ≥ 300 POL on
`RelayerStakingRegistry`) may call `VoucherRegistry.redeem()`. If the relayer's staking
status changes, voucher redemption will fail until re-staked.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Voucher redeemed successfully


**400**

**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/secret/cancel

#### POST /api/voucher/secret/cancel
**Cancel Secret for Voucher User**

Permanently revokes a time-locked secret on behalf of a voucher user. The relayer
sponsors POL gas and deducts `PricingRegistry.cancelPriceUsd` (~$1.00 USDC) from the
user's `CreditLedger` balance. Cancellation is irreversible — after this call the
Sapphire TEE sets the secret status to `Cancelled` and all future access attempts are
rejected.

**Sapphire TEE constraints (authoritative):**
- `isFrozen = true` → cancellation is permanently blocked; call reverts. Frozen secrets
  give recipients a cryptographic guarantee that the creator cannot revoke access.
- `isFlash = true` → cancellation is permanently blocked. Flash secrets cannot be
  cancelled by design (they expire naturally after their short unlock window).
- Only the original creator (`userWallet == secret.creator`) may cancel.
- Only possible **before** `unlockTime`. Once the time-lock has expired, the creator
  loses the right to cancel — the secret is already accessible to the recipient.

HTTP 402 is returned if the `CreditLedger` balance is below the cancellation fee.

**Note on `cancelSecret()` being paid (v6+):** Prior to Gateway v6 cancellation was free.
The fee now applies on both the direct-wallet path and this relayed path.

## Effective size limit

The server enforces a 128 KiB cap on the entire request body of this endpoint (checked
before validation) and returns HTTP 413 when it is exceeded; the practical ceiling is
the body cap, not the schema-level field constraints.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Secret cancelled successfully


**400**

**402**

**413**

**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/secret/create

#### POST /api/voucher/secret/create
**Create Secret for Voucher User**

Creates a time-locked secret on behalf of a voucher user who holds a USDC balance in
`CreditLedger`. This is the voucher-path equivalent of a direct wallet `createSecret()`
call. The user never interacts with the blockchain directly.

## Payment flow

The relayer calls `SecretVaultSapphireGateway.relayedCreateSecret(userWallet, ...)` on
Polygon, sponsoring the POL gas itself. Inside that contract call:
1. `CreditLedger.deduct(userWallet, price, "create", numHeirs)` validates the amount
   against `PricingRegistry.calculatePrice("create", numHeirs)` and deducts from the
   user's on-chain balance.
2. `CreditLedger.transferToSplit(price)` forwards the USDC to `SplitRegistry`, which
   distributes revenue identically to the direct-wallet path
   (40% team / 20% investors / 37% llc ops / 3% gas pool).
3. The relayer picks up the emitted `SecretCreatedFromGateway` event and bridges the
   encrypted payload to Oasis Sapphire.

The `userWallet` parameter replaces `msg.sender` throughout — all rate limiting, nonce
tracking, and `secretId` derivation use the user's wallet, not the relayer's.

## Content encryption

`encryptedContent` must be a V1 Hybrid KEM blob (1-byte version tag `0x01` + 1568-byte
ML-KEM-1024 ciphertext + 32-byte X25519 ephemeral key + 24-byte XSalsa20 nonce +
ciphertext). Fetch the TEE's X25519 and ML-KEM-1024 public keys before encrypting;
the 100 KB plaintext limit applies to the pre-encryption content.

## HTTP 402 — Insufficient credit

If the user's `CreditLedger` balance is below `PricingRegistry.calculatePrice("create", numHeirs)`,
the endpoint returns HTTP 402 with `balance` (current balance, 6 decimals) and `required`
(required amount, 6 decimals) so the app can prompt the user to top up via another voucher.

## Effective size limit

The server enforces a 128 KiB cap on the entire request body of this endpoint (checked
before validation) and returns HTTP 413 when it is exceeded. Requests approaching the
schema-level maximum for `encryptedContent` (204,800 characters) therefore cannot fit
and are rejected with 413; the practical ceiling is the body cap.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Secret created successfully


**400**

**402**

**413**

**429**

**500**

**Tags:** Vouchers

---

### /api/voucher/secret/update

#### POST /api/voucher/secret/update
**Update Secret for Voucher User**

Updates a time-locked secret (content, unlock time, or access window end) on behalf of a
voucher user. The relayer sponsors POL gas and deducts `PricingRegistry.updatePriceUsd`
(~$2.00 USDC) from the user's `CreditLedger` balance.

This endpoint wraps the unified `updateSecret()` Gateway function (v6+). Pass `0` or empty
bytes for any field you do not want to change — at least one of `newEncryptedContent`,
`newUnlockTime`, or `newAccessWindowEnd` must be non-zero/non-empty.

**Sapphire TEE constraints (authoritative — Gateway-side checks are convenience only):**
- `isFrozen = true` → all mutations permanently blocked; call reverts.
- `isFlash = true` → all mutations permanently blocked.
- Content update → only for `SimpleSecret` type; `ExternalCapsule` content is immutable
  (the decryption key is already bound to physically distributed encrypted files).
- `newUnlockTime` → must be strictly greater than the current stored value (forward-only).
- `newAccessWindowEnd` → must be strictly greater than the current value and strictly
  after the (possibly updated) `newUnlockTime`. Cannot be added post-creation.

HTTP 402 is returned if the `CreditLedger` balance is below the update fee.

## Effective size limit

The server enforces a 128 KiB cap on the entire request body of this endpoint (checked
before validation) and returns HTTP 413 when it is exceeded. Requests approaching the
schema-level maximum for `newEncryptedContent` (204,800 characters) therefore cannot fit
and are rejected with 413; the practical ceiling is the body cap.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Secret updated successfully


**400**

**402**

**413**

**429**

**500**

**Tags:** Vouchers

---

### /api/vouchers

#### GET /api/vouchers
**List Stored Vouchers**

Returns voucher records stored in this relayer's local database. The relayer
records a voucher entry whenever it processes a `VoucherRedeemed` event from Polygon or
receives a voucher via P2P sync from a peer relayer.

Only the `keccak256` hash of each voucher code is ever stored — the plaintext code is
never logged or persisted after delivery. This ensures that even if the relayer's database
is compromised, actual voucher codes cannot be extracted and reused.

**Discovery sources:** `redemption` (relayer processed it directly), `peer_sync`
(received from a peer via `/api/vouchers/sync`), `api_discovery` (reported by a client
via `/api/vouchers/discover`).

**Status lifecycle:** `discovered` → `redeemed`. Once a voucher is marked `redeemed`,
that status cannot be reversed — conflict resolution in P2P sync always preserves the
`redeemed` state over `discovered`.


**Parameters:**

- `network` (query, `unknown`) (optional)
  Filter vouchers by Polygon network
- `status` (query, `string`) (optional)
  Filter vouchers by lifecycle status
  **Enum:** `discovered`, `redeemed`, `invalid`, `expired`, `reimbursed`
- `discovered_by` (query, `string`) (optional)
  Filter by relayer address that discovered the voucher
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
Voucher list retrieved


**400**

**429**

**500**

**Tags:** Vouchers

---

### /api/vouchers/batches

#### GET /api/vouchers/batches
**Get Voucher Batch Creator**

Retrieves the creator wallet address for a voucher batch by the `registerBatch()`
transaction hash on Polygon. This is used by the registry-tool to track which wallet
(the company operator hot-wallet or the TimelockMultisig owner) registered each batch.

Batches are registered on `VoucherRegistry.sol` by calling `registerBatch(hashes[],
amounts[], expiry)`. Each batch pre-loads USDC on-chain atomically — no uncollateralised
vouchers are ever registered. The `creator_wallet` field identifies the `msg.sender` of
that transaction.

Returns `404` if the transaction hash is not in the relayer's database (the relayer may
not have scanned back far enough, or the transaction is on a different network).


**Parameters:**

- `tx_hash` (query, `string`) (required)
  Transaction hash of the batch registration

**Responses:**

**200**
Batch creator retrieved successfully
  **Properties:**
  - `data`: `object`
    Voucher batch creator lookup result

**400**
Bad request - tx_hash parameter is missing or invalid


**404**
Batch not found - transaction hash not in database


**429**

**500**

**Tags:** Vouchers

---

### /api/vouchers/discover

#### POST /api/vouchers/discover
**Report Voucher Discovery**

Allows a client to inform the relayer that it has encountered a voucher code hash —
typically before the code has been redeemed. The relayer stores the hash with status
`discovered` and propagates it to peers via `/api/vouchers/sync`. This gives the relayer
network early visibility of voucher distribution so redemption events can be correlated
to their original batch.

Only the `keccak256(code)` hash is submitted — the plaintext code is never sent. If the
hash already exists in the database, the response status is `already_exists` and no
duplicate record is created.

This endpoint is intended for use by the Secret Maker app or reseller tooling. It is
not required for normal voucher redemption — the `/api/voucher/redeem` endpoint records
the voucher automatically upon successful redemption.


**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Discovery recorded
  **Type:** `object`
  **Properties:**
  - `success`: `boolean`
    Whether the voucher discovery request succeeded
  - `data`: `object`
    Voucher discovery outcome payload
  - `timestamp`: `integer`
    Unix timestamp of the response

**400**

**429**

**500**

**Tags:** Vouchers

---

### /api/wallet/{wallet}/messages

#### GET /api/wallet/{wallet}/messages
**Get Wallet Messages**

Returns encrypted messages sent to the given wallet address via the Sapphire vault's
sendMessage function. Messages are ordered by block number descending.


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
List of messages for the wallet


**400**

**429**

**500**

**Tags:** Messages

---

### /api/wallet/{wallet}/pubkey

#### GET /api/wallet/{wallet}/pubkey
**Get Wallet Public Key**

Returns the secp256k1 public key registered for the wallet, used by senders
for client-side ECIES message encryption. Returns 404 when the wallet has not
registered a public key (senders fall back to plaintext with a warning).


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
Registered public key


**404**
No public key registered for this wallet


**429**

**500**

**Tags:** Messages

---

#### POST /api/wallet/{wallet}/pubkey
**Register Wallet Public Key**

Registers (or updates) the public key for a wallet. The registration must be
signed by the wallet (EIP-191 personal_sign of MessagePubKeyRegistrationText);
the signature is verified against the wallet address before the upsert.


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)

**Request Body:**

**Content-Type:** `application/json`


**Responses:**

**200**
Public key registered


**400**

**429**

**500**

**Tags:** Messages

---

### /api/wallet/{wallet}/transactions

#### GET /api/wallet/{wallet}/transactions
**Get Wallet Transactions**

Returns a unified history of transactions related to our contracts for a given wallet address.
Queries across events, gateway transactions, and sapphire transactions where the wallet is
involved as a creator, owner, recipient, or sender.


**Parameters:**

- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)
- `unknown` (unknown, `unknown`) (optional)

**Responses:**

**200**
List of wallet transactions


**400**

**429**

**500**

**Tags:** Events

---
