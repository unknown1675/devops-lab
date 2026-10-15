# MediChain: Backend Schema (v1.2, revised draft)

Oct 14, 2026 · @Abc

## Scope and how to read this

MediChain has no SQL or NoSQL server. The TRD splits the backend into three planes (on-chain state, off-chain Node.js services, encrypted storage), so the backend schema is the set of data shapes across eight stores.

| Item | Value |
| --- | --- |
| Version | 1.2, revised draft. Decisions D10 to D23 are accepted. |
| Date | 14 October 2026 (v1.1 was dated 10 October 2026) |
| Consolidated from | TRD v1.9, PRD v1.6, App Flow v1.7, and errata 1 to 19 |
| Companion documents | Architecture v1.2 |
| Team | Group 02, Department of CSE, BGS College of Engineering and Technology, Bengaluru. Course: Project Phase-I (BCS685), confirm the label is still right |

| # | Store | Plane | Technology | Described in |
| --- | --- | --- | --- | --- |
| 1 | Contract storage | On-chain | Solidity structs and mappings on Hardhat, chain 31337 | On-chain storage schemas |
| 2 | Event log | On-chain | AuditLogged, KeysWrapped, ClaimCommitted, ClaimRevoked | Events and audit mapping |
| 3 | Record envelope | Storage | AES-GCM JSON pinned to IPFS through Pinata, reached through the IPFS proxy, which also keeps a local cache of ciphertext | Off-chain data schemas |
| 4 | Share bundle | Custody | ML-KEM-encrypted JSON held by each trustee | Off-chain data schemas |
| 5 | Service messages | Off-chain | HTTP on 127.0.0.1: coordinator and dealer launcher (port 4100), IPFS proxy (port 4101), and the relayer | Service interfaces |
| 6 | Coordinator memory | Off-chain | In-process only, wiped on close, never written to disk | Service interfaces |
| 7 | Device storage | Client | IndexedDB keystore and address book | Off-chain data schemas |
| 8 | Config and fixtures | Off-chain | `.env` file, demo identity fixture | Configuration schema |

### Labels used below

Unmarked means the TRD v1.9 states it. *errata N* means the shape comes from item N of the errata file, which the TRD now folds in. *inferred* means a function signature or flow implies it, but neither the TRD nor the errata file spells it out. *(v1.2)* marks a row that changed in this version.

### Precedence

The TRD now agrees with the errata file, so the two can be read together. The TRD supersedes the errata file, which stays in the repository for history only. Where any difference remains, this schema wins on exact field and function shapes, and the TRD wins on behavior. Older plan files name liboqs, a Python secret-sharing library, and Ganache. Those are superseded by `@noble/post-quantum`, secrets.js-grempe, and Hardhat's built-in network. One module-plan file (`team_members.md`) still names liboqs and should be corrected.

This schema also carries small additions the TRD did not state in v1.8, all accepted: the share-bundle `iv` and `authTag`, the Consent projection, the relayer request, the address book entry, the demo fixture shape, the Activity chip mapping, and the extra commitment checks. The last section lists the 19 resolved items, the decisions (D10 to D23, all accepted).

## Identifiers, constants, and vocabularies

Every id, vocabulary, and limit used by the schemas below is defined once here.

### Identifiers

| Name | Type | Meaning |
| --- | --- | --- |
| patientId | bytes32 | The `didHash` passed to `PatientIdentity.register`. `patientId = keccak256(utf8(didString))`, where the DID string is `did:medichain:` followed by the lowercase `0x` hex address of the registering account (errata 5). `register` checks this binding on-chain, so nobody can register another party's identifier (PAT-8, D15) *(v1.2)* |
| account | address | The patient's `PatientAccount` contract. It is `msg.sender` in every other contract, so it is the patient's on-chain identity. A `PatientAccountFactory` creates it and resolves wallet to account (PAT-7, D10) *(v1.2)* |
| owner | address | The wallet key (EOA) that owns a `PatientAccount` and signs EIP-712 messages |
| party | address | Any registered identity: patient account, doctor, lab, Hospital B. Key lookups are by address |
| resourceType | bytes32 | `keccak256` of a FHIR-style type name |
| claimType | bytes32 | `keccak256` of a short claim name |
| roleId | bytes32 | `keccak256(bytes(roleName))`, for example `keccak256(bytes('Doctor'))`. `checkAccess` maps it back to the `Role` enum by name (errata 3) |
| tokenId, entryId, requestId | bytes32 | Ids of consent tokens, registry entries, and break-glass or recovery requests. Each is `keccak256(abi.encode(block.chainid, address(this), msg.sender, counter))` with a per-contract counter (errata 6) |
| recordId | string | The client-side FHIR resource id. It is known before upload and is the input to key derivation. It matches the FHIR id pattern (letters, digits, hyphen, dot, at most 64 characters). It is not the same as `entryId` |
| epoch | uint32 | Generation counter of a patient's share set *(inferred type)* |

### Vocabularies

- **Resource types** (`keccak256` of the name): Patient, Observation, Consent, EmergencyCapsule, Claim. Consent and Claim are permission categories. A consent token can name them, but no record is stored under them. "Claim" is MediChain's category for issued facts such as age over a threshold, and it has nothing to do with the FHIR billing resource of the same name *(v1.2)*.
- **Claim types:** AgeOver. A diagnostic-threshold claim is the stretch case (ZKP-6).
- **Roles:** Doctor, Lab, Insurer, IoTDevice, EmergencyResponder.
- **Capsule fields:** bloodType, allergies, currentMedications, emergencyContacts.

EmergencyCapsule is never granted in a consent token. A Claim token carries an empty wrapped-key entry.

### Constants and limits

| Constant | Value | Where it applies |
| --- | --- | --- |
| Share commitment domain tag | `medichain-share-v1` | `sha256(utf8(shareString) \|\| salt \|\| utf8(domain tag))`. `shareString` is the exact string `secrets.share()` returned, never hex-decoded (errata 4) |
| BN254 scalar field | 21888242871839275222246405745257275088548364400416034343698204186575808495617 | Claim commitment range, nonce reduction |
| `WINDOW_DURATION` | 180 s (demo) | BreakGlassCapsule constructor |
| `TIER2_DELAY` | 60 s (demo) | RecoveryVault constructor |
| `COLLECTION_TIMEOUT` | 20 s (demo) | Coordinator config, off-chain. Read against chain time (D11) |
| `FULL_MODE_TTL` | 300 s (demo) | Coordinator config, off-chain. The deadline for a lost-key (full mode) request, which has no on-chain window. It covers the 20 s collection and the 60 s Tier 2 delay (errata 18) |
| Mining interval | 1000 ms (demo) | Hardhat node. One block a second, so block time follows wall-clock time (D11) *(v1.2)* |
| Deploy rule | `WINDOW_DURATION` at least 2 x `TIER2_DELAY` | The deploy script asserts it |
| Threshold k | 2 <= k <= n+1, enforced on-chain | RecoveryVault |
| Category cap | No `TrusteeCategory` on k or more rows (D14) | RecoveryVault *(v1.2)* |
| Demo configuration | n = 4 guardians plus the custodian, k = 3 | Every screen and demo beat *(v1.2)* |
| CID length | 1 to 100 bytes | `RecordRegistry.addRecord` |
| Claim value and threshold | Fit in 8 bits | Circuit, ProofGateway |
| Claim salt | 31 random bytes | Poseidon input, always a valid field element |
| Share salt | At least 16 random bytes | Share commitment |
| PBKDF2 iterations | At least 600,000 | Device keystore |
| AES-GCM | 256-bit key, 12-byte IV, 16-byte tag | Records, keystore, key wrapping |
| ML-DSA-65 | Public key about 1,952 B, signature about 3,309 B | Record and audit signatures |
| ML-KEM-768 | Public key about 1,184 B, ciphertext about 1,088 B | Share transport, type-key wrapping |

## On-chain storage schemas

Solidity ^0.8.24, OpenZeppelin 5.x, `evmVersion: 'cancun'`. Every patient action reaches these contracts through `PatientAccount`, so patient addresses below are account addresses. Ten contracts are covered, one more than in v1.1.

### PatientIdentity (Anush)

```solidity
struct Identity {
    bytes32 patientId;       // didHash = keccak256(utf8(didString)), errata 5
    bytes dilithiumPubKey;   // ML-DSA-65
    bytes mlkemPubKey;       // ML-KEM-768
}
mapping(address => Identity) identityOf;   // inferred container
mapping(bytes32 => address) accountOf;     // patientId to account
mapping(address => bytes32) patientIdOf;   // account to patientId
```

Rules: one identity per address and per patientId. `register` requires `didHash == keccak256(abi.encodePacked('did:medichain:', Strings.toHexString(msg.sender)))` (PAT-8, D15) *(v1.2)*. There is no `updateKeys`, because keys derive from S. Doctors, labs, and Hospital B register the same way, from their own address, with keys derived from their own secret (DAT-9). An address counts as registered when `getDilithiumKey` returns a non-empty key. For a patient the lookup is by account address, not wallet address (SCR-01 routing, v1.2).

### PatientAccountFactory (Anush, new, D10) *(v1.2)*

```solidity
mapping(address => address) accountOfOwner;   // owner wallet to its PatientAccount
```

`createAccount(owner)` deploys a `PatientAccount` with `CREATE2` using the owner as the salt, records it, and returns the address. If the account exists it returns the existing one. Anyone can call it, because the owner is fixed in the constructor and the address is deterministic. `predictAccount(owner)` returns the address before deployment. The relayer creates the account on a patient's first connection and pays the gas.

### PatientAccount (Anush)

One instance per patient.

```solidity
address owner;    // wallet key
uint256 nonce;    // incremented before each executeSigned call
```

### RoleRegistry (Anush)

```solidity
enum Role { None, Doctor, Lab, Insurer, IoTDevice, EmergencyResponder }
mapping(address => Role) roleOf;   // inferred: one role per account, set by assignRole(account, role)
```

Assignment is admin-only through OpenZeppelin `AccessControl`. A consent token's `roleId` is `keccak256(bytes(roleName))` over the names Doctor, Lab, Insurer, IoTDevice, and EmergencyResponder, and `ConsentRegistry.checkAccess` maps it back to the `Role` by name (errata 3).

### ConsentRegistry (Anush)

```solidity
struct ConsentToken {
    bytes32 tokenId;          // keccak256(abi.encode(block.chainid, address(this), msg.sender, tokenCounter)), errata 6
    address patient;          // the patient's account address, not the owner wallet
    address grantee;
    bytes32 roleId;           // keccak256(bytes(roleName)), errata 3
    bytes32[] resourceTypes;
    uint64 issuedAt;
    uint64 expiresAt;
    bool revoked;
}
mapping(bytes32 => ConsentToken) tokens;
mapping(address => bytes32[]) tokenIdsByPatient;   // inferred, serves tokenIdsOf
mapping(address => bytes32[]) tokenIdsByGrantee;   // inferred, serves tokenIdsOfGrantee
uint256 tokenCounter;                              // errata 6
address proofVerifier;                             // setProofVerifier, admin only. Holds the ProofGateway address
```

Rules:

- A token is valid when it is not revoked and not expired. Expiry needs no action from the patient.
- `expiresAt` must be in the future when issued. `wrappedTypeKeys.length` must equal `resourceTypes.length`.
- Wrapped keys are emitted in `KeysWrapped` and never stored.
- Only a registered patient account issues. Only the issuing patient revokes.
- `canAccess` is a free view call. `checkAccess` is a transaction that logs the outcome and does not revert on a denial. It requires `msg.sender == requester` (D13) *(v1.2)*.

### RecordRegistry (Anush)

```solidity
struct RecordRef {
    bytes32 entryId;       // registry id, assigned on add (errata 6)
    bytes32 patientId;
    bytes32 resourceType;
    string  cid;           // real CID string, v0 or v1, 1 to 100 bytes
    bytes32 contentHash;   // sha256(ciphertext)
    address creator;       // the signer (decision D9)
    uint64  createdAt;
}
mapping(bytes32 => RecordRef) records;                               // by entryId
mapping(bytes32 => mapping(bytes32 => bytes32[])) entryIds;         // patientId, resourceType, in order added (serves recordsOf)
uint256 entryCounter;                                               // errata 6
```

Rules: only the patient's account may add (`accountOf(patientId) == msg.sender`). There is no revoked flag and no delete. The metadata is public by design (known limit 7).

### ProofGateway (Punith)

The verifier contract is generated by snarkjs. The gateway holds the state.

```solidity
struct ClaimEntry { uint256 commitment; address issuer; }       // commitment = Poseidon(value, salt)
mapping(bytes32 => mapping(bytes32 => ClaimEntry)) claims;      // patientId, claimType
mapping(uint256 => bool) usedNonce;
// public signals, in this order: [threshold, commitment, nonce]
// proof: uint256[8] = a[2], b[4], c[2]
```

Rules:

- `setCommitment` needs the Doctor or Lab role and an active Claim token from the patient that names the caller. Only the original issuer may replace a commitment. The patient's account or the issuer may revoke.
- The commitment is non-zero and below the field modulus.
- `nonce = keccak256(abi.encode(chainid, gateway address, requestId)) mod field`. Each nonce works once. An invalid proof returns false and does not use up the nonce.
- The gateway resolves the patient's account through `PatientIdentity.accountOf(patientId)` (dependency added in v1.2).

### AuditLog (Punith)

```solidity
enum ActionType {
    ConsentIssued, ConsentRevoked, AccessGranted, AccessDenied,
    ProofVerified, BreakGlassTriggered, BreakGlassExpired,
    RecoveryInitiated, TierEscalated, RecoveryCompleted,
    RecoveryCancelled, RecordAdded, ClaimIssued, ClaimRevoked   // new values are appended only
}
mapping(address => bool) isEmitter;   // registerEmitter, admin only
```

The log has no other storage. It is events only. If `log()` reverts, the calling transaction reverts, which is how AUD-2 is enforced.

### RecoveryVault (Rohith and Yagna)

```solidity
enum TrusteeCategory { Family, HospitalCustodian, GuardianService, InstitutionalCustodian }
enum ReleaseMode { None, CapsuleOnly, FullRecord }   // errata 11: None = 0. Tests and UI use names, never numbers

struct ShareCommitment {              // v1.2: fields reordered so trustee, category, and index share one storage slot
    bytes32 commitment;               // sha256(utf8(shareString) || salt || utf8('medichain-share-v1')), errata 4
    address trustee;
    TrusteeCategory category;
    uint8 index;                      // 1..n guardians, n+1 = institutional custodian
}
struct GuardianSet {                  // inferred container
    uint8 threshold;                  // k
    uint32 epoch;                     // +1 on every rotateShareCommitments
    uint64 updatedAt;                 // errata 10, drives last refreshed and Refresh due
    ShareCommitment[] commitments;
}
mapping(bytes32 => GuardianSet) guardianSets;   // by patientId

struct RecoveryRequest {              // errata 2: the vault is the source of truth for this state
    bytes32 patientId;
    address requester;
    ReleaseMode mode;
    bytes32 clientKeyHash;            // sha256 of the one-time ML-KEM public key, non-zero in FullRecord
    bool completed;
    bool cancelled;
    uint64 tier2EndsAt;               // 0 until custodianCoSign
}
mapping(bytes32 => RecoveryRequest) requests;   // by requestId
uint256 requestCounter;                          // errata 6
address breakGlassCapsule;                       // set once through setBreakGlassCapsule, errata 16
uint64 immutable TIER2_DELAY;
```

Rules:

- `registerShareCommitments` is patient-only and once per patient. `rotateShareCommitments` replaces the set and increments `epoch`. Both enforce `2 <= threshold <= commitments.length`.
- Both revert if every guardian row (index 1 to n) has the same category. The custodian row is excluded from that check (T1-5). The UI warns before submitting, so the revert is the backstop (errata 12).
- **Category cap (D14)** *(v1.2)*: both also revert if any one category appears on k or more rows, custodian row included. The first rule only catches a set where all guardians match. Three family members and one friend at k = 3 would pass it, yet the three family members could rebuild S.
- Both also require indexes that are unique and contiguous 1 to n+1, exactly one `InstitutionalCustodian` row at index n+1, and unique trustee addresses.
- No function parameter or event ever carries a raw share, salt, or key.
- `setReleaseMode(requestId, patientId, requester, mode)` is external with an `onlyBreakGlassCapsule` check, and the capsule contract's address is set once through `setBreakGlassCapsule` (errata 16). The TRD labelled it internal in v1.8, which was an error, because an internal function cannot be called from another contract (errata 1). It writes the request row, so the vault holds patientId, requester, release mode, cancelled, and tier2EndsAt for every request (errata 2).
- `setBreakGlassCapsule(address)` is admin-only, callable once, and reverts if the address is already set. The capsule contract is deployed after the vault, because its constructor takes the vault address, so the deploy script calls this once both contracts exist (errata 16).
- With `patientId` stored, `cancelRecovery` can check its caller against the patient and the registered guardians, and `custodianCoSign` can check that `requestId` belongs to `patientId` (errata 2).
- A lost-key `requestId` is `keccak256(abi.encode(block.chainid, address(this), msg.sender, requestCounter))` (errata 6).
- `startRecovery` is for the patient's account only and rejects a zero `clientKeyHash`. `completeRecovery` is for the requester only, once, not after a cancellation, and only when the mode is FullRecord, so a break-glass responder cannot log RecoveryCompleted (errata 17). `cancelRecovery` is for the patient or a registered guardian.
- Views: `getCommitments(patientId)`, `getGuardianSet(patientId)` returning threshold, epoch, and updatedAt (errata 10), `getRequestStatus(requestId)` returning release mode, cancelled, and `tier2EndsAt`, and `getRecovery(requestId)` returning the whole request row *(v1.2: the v1.1 schema omitted the last two)*.

### BreakGlassCapsule (Yagna)

```solidity
struct Capsule {                      // inferred from preAuthorizeCapsule and capsuleStatus. By patientId
    bool preAuthorized;
    uint32 version;                   // bumped on each preAuthorizeCapsule (inferred)
    uint64 preAuthorizedAt;
    bytes32[] fields;                 // keccak256 of the field names in Vocabularies
}
struct BGRequest { uint64 windowExpiry; bool closed; }   // errata 2: only the window state lives here. By requestId
uint256 requestCounter;                                  // errata 6, used by _newRequestId
uint64 immutable WINDOW_DURATION;
enum RequestStatus { None, WindowOpen, Tier2Waiting, Cancelled, Closed }
struct RequestInfo {                  // returned by getRequest
    bytes32 patientId; address requester; ReleaseMode releaseMode;
    RequestStatus status; uint64 windowExpiry; uint64 tier2EndsAt;
}
```

Rules: `triggerEmergencyAccess` runs the role check, then the capsule check, then `recoveryVault.setReleaseMode(requestId, patientId, requester, CapsuleOnly)`, then sets `windowExpiry`, then logs, all in one transaction. `closeWindow` is permissionless, only after expiry, and only once. `revokeCapsule` clears `preAuthorized`. `preAuthorizeCapsule` resolves the patient through `PatientIdentity.patientIdOf(msg.sender)` (dependency added in v1.2). `getRequest` reads patientId, requester, release mode, cancelled, and tier2EndsAt from RecoveryVault and combines them with its own `windowExpiry` and `closed` to build status (errata 2). When a cancelled request is also closed, `Cancelled` wins, so the screen keeps showing why it ended *(v1.2)*. `closeWindow` reads patientId from the vault for its audit call, and its unknown-request check uses `windowExpiry != 0`, because `requester` no longer lives here (errata 19). A lost-key `requestId` has no entry here, so `closeWindow` reverts it as unknown. `requestId` comes from `_newRequestId`: `keccak256(abi.encode(block.chainid, address(this), msg.sender, requestCounter))` (errata 6).

## Events and audit mapping

Four events carry everything the chain announces, and fourteen action types say which function logged what.

### Events

| Event | Emitted by | Arguments | Read by |
| --- | --- | --- | --- |
| `AuditLogged` | AuditLog, called by every logging contract | `patientId` (indexed), `actor` (indexed), `action` (indexed), `refId`, `timestamp` | Activity (SCR-09), Patient home (SCR-03) |
| `KeysWrapped` | `ConsentRegistry.issueConsent` | `tokenId` (indexed), `grantee` (indexed), `resourceTypes[]`, `wrappedTypeKeys[]` | Grantee client, SCR-12, SCR-13 |
| `ClaimCommitted` | `ProofGateway.setCommitment` | `patientId`, `claimType`, `commitment`, `issuer` | Verifiers |
| `ClaimRevoked` | `ProofGateway.revokeClaim` | `patientId`, `claimType`, `revokedBy` | Verifiers |

`ClaimRevoked` is both an event name and an `ActionType` value. They are separate things.

### Which function logs which ActionType

| ActionType | Logged by | actor | refId |
| --- | --- | --- | --- |
| ConsentIssued | `issueConsent` | patient account | tokenId |
| ConsentRevoked | `revokeConsent` | patient account | tokenId |
| AccessGranted, AccessDenied | `checkAccess` (the logged transaction, never `canAccess`) | requester, which must equal `msg.sender` (D13) *(v1.2)* | tokenId |
| ProofVerified | `verifyEligibility`, `verifyBatch` | caller | requestId |
| RecordAdded | `addRecord` | patient account | entryId |
| ClaimIssued | `setCommitment` | issuer | tokenId of the Claim token (errata 8) |
| ClaimRevoked | `revokeClaim` | patient account or issuer | claimType (errata 8) |
| BreakGlassTriggered | `triggerEmergencyAccess` | responder | requestId |
| BreakGlassExpired | `closeWindow` | caller | requestId |
| RecoveryInitiated | `startRecovery` | patient account | requestId |
| TierEscalated | `custodianCoSign` (errata 8) | custodian | requestId |
| RecoveryCancelled | `cancelRecovery` | patient or guardian | requestId |
| RecoveryCompleted | `completeRecovery` | patient account | requestId |

### Activity filter categories

SCR-09 has the chips All, Sharing, Emergency, and Recovery. The App Flow does not map each `ActionType` to a chip, so this schema defines the mapping.

- **Sharing:** ConsentIssued, ConsentRevoked, AccessGranted, AccessDenied, ProofVerified, RecordAdded, ClaimIssued, ClaimRevoked.
- **Emergency:** BreakGlassTriggered, BreakGlassExpired.
- **Recovery:** RecoveryInitiated, RecoveryCompleted.
- **TierEscalated and RecoveryCancelled:** Emergency when the request's release mode is CapsuleOnly, Recovery when it is FullRecord.

Sentences such as "3 of 4 guardians responded" have no on-chain event. They come from the signed off-chain audit record (errata 9).

## Off-chain data schemas

Types are TypeScript. `b64` means a base64 string, and `hex` means a `0x`-prefixed hex string.

### FHIR-inspired resources (DAT-1) *(v1.2)*

This is the plaintext that gets encrypted. Every resource carries `resourceType`, `id`, and a `subject` reference, except Patient, where `id` plays the subject's role (errata 13). Ids match the FHIR id pattern, so a DID can never be an id. A party's DID travels as an `identifier` (D16). The issued-claim extension is a valid complex extension, where v1.1 used loose properties that a FHIR validator would reject.

```ts
interface Observation {
  resourceType: 'Observation';
  id: string;                                   // becomes recordId. [A-Za-z0-9\-.]{1,64}
  status: 'final';
  subject: { reference: string };               // Patient/pat-0001
  code: { text: string };                       // e.g. Blood glucose
  valueQuantity: { value: number; unit: string };
  effectiveDateTime: string;                    // ISO 8601
  performer: { type: 'Practitioner'; identifier: { system: string; value: string } }[];
                                                // identifier.value = did:medichain:0xb2...
}
interface Patient {
  resourceType: 'Patient';
  id: string;                                   // plays the subject's role (errata 13)
  identifier: { system: 'https://medichain.example/did'; value: string }[];   // did:medichain:0xa1...
  extension?: IssuedClaimExtension[];           // present when an issuer created the record
}
interface IssuedClaimExtension {
  url: 'https://medichain.example/fhir/StructureDefinition/issued-claim';
  extension: (
    | { url: 'claimType'; valueString: string }     // e.g. AgeOver
    | { url: 'value'; valueInteger: number }        // fits in 8 bits
    | { url: 'salt'; valueString: string }          // hex, 31 bytes
    | { url: 'issuer'; valueString: string }        // did:medichain:0xb2...
  )[];
}
// Read-only projection of an on-chain ConsentToken. Not stored on IPFS and never encrypted.
interface Consent {
  resourceType: 'Consent';
  id: string;                                   // tokenId
  status: 'active' | 'inactive';                // inactive = revoked or expired
  subject: { reference: string };
  grantee: { reference: string };
  provision: { period: { start: string; end: string }; class: string[] };   // resource type names
  chain: { chainId: number; registry: string; tokenId: string };
}
```

The seed script gives each Synthea resource a valid id (Synthea's UUIDs already fit), rewrites `urn:uuid:` references to `Patient/<id>`, and adds the DID identifier.

### Record envelope (DAT-2, DAT-4)

```ts
interface RecordEnvelope {                // what is uploaded to IPFS
  schemaVersion: '1.0';
  resourceType: string;                   // the name, not the hash
  recordId: string;
  ciphertext: b64;                        // AES-256-GCM of the FHIR JSON under recordKey
  iv: b64;                                // 12 bytes, never reused under one key
  authTag: b64;                           // 16 bytes
  signature: { alg: 'ML-DSA-65'; signerId: string; value: b64 };   // signerId is the creator's did:medichain:...
}
```

On-chain `contentHash` is `sha256(ciphertext)`. A reader checks the hash first, then the signature against `getDilithiumKey(creator)`, then decrypts. A failed GCM tag aborts with no partial output. The signature covers the ciphertext only. This is a documented limitation, kept as in the TRD (errata 14).

### Wrapped type key (DAT-5)

```ts
interface WrappedTypeKey {                // one per resource type, serialized to bytes inside KeysWrapped
  kemAlg: 'ML-KEM-768';
  encapsulatedKey: b64;
  iv: b64;
  wrappedTypeKey: b64;
  authTag: b64;
}
```

`wrappedTypeKeys[i]` matches `resourceTypes[i]`. The entry is empty for the Claim type. Each entry serializes to bytes as UTF-8 JSON in the shape above (errata 5).

### Share bundle and commitment (T1-2, T1-4, T1-7)

```ts
interface ShareBundle {                   // held by the trustee, never on-chain
  trusteeId: string;                      // e.g. guardian-1
  shareIndex: number;                     // 1..n+1
  kemAlg: 'ML-KEM-768';
  encapsulatedKey: b64;                   // wrapped to the trustee's ML-KEM public key
  iv: b64;                                // 12 bytes, added to the TRD bundle shape
  ciphertext: b64;                        // AES-256-GCM of ShareSecret under the KEM shared secret
  authTag: b64;                           // 16 bytes, added to the TRD bundle shape
}
interface ShareSecret { share: string; salt: b64 }   // plaintext, in memory only for a moment. share is the exact string from secrets.share() (errata 4)
// on-chain: commitment = sha256(utf8(shareString) || salt || utf8('medichain-share-v1')), with salt as raw bytes (errata 4)
```

Shares and salts are checked against the on-chain commitments off-chain, in the coordinator. They never appear in calldata or events. A share is never hex-decoded: secrets.js-grempe shares are odd-length hex strings, 99 characters for a 32-byte secret, and decoding drops the last nibble (errata 4). The v1.2 review confirmed both facts on secrets.js-grempe 2.0.0 under Node 22.

### Emergency capsule (BG-4, decision D1)

```ts
interface CapsuleMeta {                   // TRD schema
  capsuleId: string; patientId: string; fields: string[]; preAuthorizedAt: string; version: number;
}
```

The TRD shows field names only. The values live in `CapsulePayload`, which the client library encrypts under `capsuleKey` and registers in `RecordRegistry` under the `EmergencyCapsule` type (errata 7):

```ts
interface CapsulePayload {                // the plaintext inside the capsule envelope (errata 7)
  capsuleId: string; patientId: string; version: number; preAuthorizedAt: string;
  fields: {
    bloodType?: string;
    allergies?: string[];
    currentMedications?: { name: string; dose?: string }[];
    emergencyContacts?: { name: string; relationship: string; phone: string }[];
  };
}
```

Only the fields the patient ticked are present. Break-glass releases this object and nothing else.

### Signed audit record (AUD-3, AF-9)

```ts
interface SignedAuditRecord {
  patientId: string; actor: string;
  action: string;                         // an ActionType name
  refId: string; timestamp: number;       // unix seconds
  attestation?: boolean;                  // responder ticked that the patient cannot consent. UI-only, not enforced by the contract
  trustees?: { index: number; category: string; status: 'waiting' | 'responded' | 'invalid' | 'timeout' }[];   // errata 9
  signature: { alg: 'ML-DSA-65'; signerId: string; value: b64 };
}
```

Records are stored as an append-only JSONL file per patient. The coordinator assembles each record and signs it with a registered demo Coordinator identity from the fixture (errata 9).

### Device storage (AF-3, AF-8, AF-12)

```ts
interface Keystore {                      // IndexedDB record, also exportable as a backup file
  version: 1; kdf: 'PBKDF2-SHA256'; iterations: number;   // at least 600000
  salt: b64;                              // 16 bytes
  iv: b64; ciphertext: b64; authTag: b64; // AES-256-GCM of { recordSecret }
}
interface AddressBookEntry {              // non-secret, local only
  address: string; displayName: string;
  group?: 'Family' | 'Hospital' | 'GuardianService'; savedAt: number;
}
```

The keystore holds one secret: S for a patient, the party's own secret for a clinician, lab, or Hospital B. A wrong passphrase fails the GCM tag, so there is no separate password check. Key pairs are derived at unlock and held in React state only. Guardian display names live in the address book, not on-chain. The emergency card or bracelet carries `medichain://patient/<patientId as 0x hex>` and never medical data.

### Demo identity fixture (DAT-9) *(v1.2)*

`fixtures/demo-identities.DEMO-ONLY.json` is git-ignored and never committed. Each identity is one of Hardhat's unlocked accounts, so a script or screen signs with a JSON-RPC signer for that address. The fixture holds each party's secret S but no transaction-signing key (D13).

```ts
interface DemoFixture {
  label: 'DEMO ONLY'; chainId: 31337;
  identities: {
    name: string;      // e.g. Dr. Rao, Dr. Mehta, Hospital B, Guardian 1, Coordinator
    role: 'Patient' | 'Doctor' | 'Lab' | 'EmergencyResponder' | 'Guardian' | 'Custodian' | 'Coordinator' | 'HospitalB';
    address: string; did: string;
    secret: b64;       // 32 bytes, the party's own S
  }[];
}
```

The Coordinator identity signs the audit records the coordinator assembles (errata 9).

## Service interfaces

Three local services and one library interface carry the off-chain messages. The dealer launcher and coordinator share one process on port 4100, and the IPFS proxy is a separate process on port 4101.

### Rules shared by the local services *(v1.2, D17)*

- Each binds to 127.0.0.1 and rejects any other source address.
- Each generates a session bearer token at startup and hands it to the frontend through local config. A call without it gets 401, except the proxy's read-only GET /ipfs/:cid. The coordinator adds a per-request token (D22).
- CORS allows only `http://localhost:5173`. The Hospital B viewer on port 5174 may call only the IPFS proxy's read-only GET /ipfs/:cid route (D18).
- Browsers read the event stream with a fetch stream that sends the token in an `Authorization` header, because `EventSource` cannot send headers.

### Dealer launcher (AF-16)

Loopback only, with the per-session bearer token. The dealer is a one-shot process that overwrites what it can and exits.

```ts
// POST /dealer/run
interface DealerRunRequest {
  n: number; k: number;
  trustees: { index: number; category: TrusteeCategory; kemPublicKey: b64 }[];   // includes the custodian at n+1
}
interface DealerRunResponse { sessionId: string; dealerKemPublicKey: b64 }

// POST /dealer/:sessionId/secret
type DealerSecretRequest = KemWrapped;    // S wrapped to dealerKemPublicKey, never plaintext
interface DealerSecretResponse {
  bundles: ShareBundle[];                 // n+1
  commitments: { index: number; commitment: hex }[];   // the client adds trustee address and category to build ShareCommitment[]
}
interface KemWrapped { kemCipherText: b64; iv: b64; ciphertext: b64; authTag: b64 }
```

Re-sharing uses the same two calls, with S taken from the keystore. Each commitment is `sha256(utf8(shareString) || salt || utf8('medichain-share-v1'))`, computed over the exact string `secrets.share()` returned (errata 4).

### RecoveryCoordinator API (AF-7)

Bound to `127.0.0.1:4100`. Each request gets a random bearer token when it is created, and every other call must present it.

| Call | Body | Response |
| --- | --- | --- |
| `POST /requests` | `patientId`, `mode` (capsule or full), `requestId`, plus `clientKemPublicKey` in full mode | `coordinatorKemPublicKey` and `requestToken` |
| `POST /requests/:id/shares` | `trusteeIndex`, `kemCipherText`, `iv`, `ciphertext`, `authTag`, wrapped to the coordinator's one-time key | Accepted, or rejected if not wrapped to that key |
| `GET /requests/:id/events` | none | Server-Sent Events of `CoordinatorEvent` |
| `GET /requests/:id/result` | none | Capsule mode: the capsule fields. Full mode: S as `KemWrapped`, once. 409 until released, 410 after close |
| `POST /requests/:id/close` | none | Wipes the one-time key and anything released |

```ts
interface CoordinatorEvent {
  requestId: string;
  phase: 'collecting' | 'tier2-wait' | 'reconstructing' | 'released' | 'failed' | 'cancelled' | 'closed';
  needed: number;                         // k
  validCount: number;                     // shares that passed the commitment check
  trustees: { index: number; category: string; status: 'waiting' | 'responded' | 'invalid' | 'timeout' }[];
  windowExpiry: number;                   // unix seconds. Break-glass: mirrors the chain. Full mode: the off-chain FULL_MODE_TTL deadline (errata 18)
  tier2EndsAt?: number;                   // unix seconds, mirrors the chain
}
```

Full-mode preconditions: `getRecovery` shows FullRecord, the requester is the patient's account, the request is neither cancelled nor completed, and `sha256(clientKemPublicKey)` equals the stored `clientKeyHash`. The coordinator never releases after `windowExpiry`. In full mode there is no on-chain window, so `windowExpiry` is the coordinator's own deadline, `FULL_MODE_TTL` after the request starts, and the coordinator wipes its keys at that point (errata 18). All deadlines read chain time (D11).

### Coordinator memory (inferred, never persisted)

```ts
interface CoordinatorRequest {
  requestId: string; patientId: string; mode: 'capsule' | 'full';
  bearerToken: string;
  ephemeralKem: { publicKey: Uint8Array; secretKey: Uint8Array };   // one-time, dropped on close
  clientKemPublicKey?: Uint8Array;                                  // full mode
  commitments: ShareCommitment[];                                   // read from the chain
  shares: Map<number, { share: Uint8Array; salt: Uint8Array; valid: boolean }>;   // overwritten after use, best effort
  phase: CoordinatorEvent['phase'];
  windowExpiry: number; tier2EndsAt?: number;                       // full mode: windowExpiry is the FULL_MODE_TTL deadline (errata 18)
  released?: Uint8Array;                                            // capsule fields or wrapped S, wiped after one read or at expiry
}
```

The coordinator briefly holds S in both modes (known limit 8). Overwriting is best effort in JavaScript.

### IPFS proxy *(v1.2, D12)*

The browser cannot safely hold a Pinata credential, and a service that handled keys would break the rule that no key reaches a service. The proxy holds only the credentials and sees only ciphertext envelopes.

```ts
// POST /ipfs/pin: body is a RecordEnvelope. Rejected unless it is a well-formed envelope under the size cap
interface PinResponse { cid: string }
// GET /ipfs/:cid: returns the RecordEnvelope, from the local cache first, otherwise through the gateway, then cached
```

### Capsule mode, proof request, and tokens *(D18 to D23)*

This subsection adds the shapes the second review found missing. ProofService is a benchmark and seed helper, not one of the demo's processes, because the browser generates the demo's proofs (D23).

```ts
// IPFS proxy origins and tokens (D18, D22)
// POST /ipfs/pin : session token, frontend origin only
// GET  /ipfs/:cid : no token, frontend and Hospital B origins (5173, 5174). Serves ciphertext only

// Proof request (D19). Handed to the patient as a link or code. Scripted in the demo.
interface ProofRequest { claimType: string; requiredThreshold: number; requestId: hex; verifier: string }
// ProofGateway emits this on every successful verifyEligibility and verifyBatch proof:
// event EligibilityVerified(bytes32 indexed patientId, bytes32 claimType, bytes32 requestId, uint256 requiredThreshold, address indexed verifier)

// Coordinator requester check (D20). Both modes.
// POST /requests/challenge { requestId } -> { challenge: string }   // random, single-use
// POST /requests adds: { requesterSignature: hex }                  // wallet signature over the challenge
// The signer must be getRecovery(requestId).requester. For a PatientAccount requester, the account's owner signs.
```

**Capsule-mode checks (D20).** The coordinator requires `getRecovery` to show CapsuleOnly, not cancelled, not completed, and `BreakGlassCapsule.getRequest` to show WindowOpen or Tier2Waiting. It reads the newest `EmergencyCapsule` record through `GET /ipfs/:cid`, checks `contentHash` and the creator's signature, requires the payload `version` to equal the on-chain version, and releases only the fields whose `keccak256(name)` appears in `capsuleFields(patientId)`. In this mode it decrypts the emergency capsule only, and never any other patient record. Saving emergency info registers the capsule first and pre-authorizes it second.

**On-chain additions.** `BreakGlassCapsule.capsuleFields(patientId)` returns the pre-authorized field hashes and the version. `RecoveryVault.custodianCoSign` reverts unless the caller is the registered custodian, the request exists and belongs to `patientId`, its mode is not None, it is not cancelled or completed, and `tier2EndsAt == 0` (D21). `ProofGateway.verifyEligibility` and `verifyBatch` revert for a required threshold of 256 or more (D23), and the circuit range-checks `value` and `threshold` with `Num2Bits(8)`.

| Token | Created | Authorizes |
| --- | --- | --- |
| Session token | At service startup | `POST /requests/challenge`, `POST /requests`, the dealer calls, and `POST /ipfs/pin` |
| Request token | Returned by `POST /requests` | `GET /requests/:id/events`, `GET /requests/:id/result`, `POST /requests/:id/close`, and `POST /requests/:id/shares` |
| One-time key wrap | The request's coordinator key | A share must also be wrapped to it. The guardian script gets the request token with the key |

If the k-th valid guardian share arrives during the Tier 2 delay, the coordinator rebuilds S at once and the delay is dropped (D23).

### Relayer and sponsored actions (AF-11)

The patient signs and the relayer submits and pays gas. The relayer can delay or drop an action but cannot forge one. It also creates a patient's account on first connection through `PatientAccountFactory.createAccount(owner)`.

```ts
// EIP-712
domain:  { name: 'MediChainAccount', version: '1', chainId: 31337, verifyingContract: accountAddress }
message: Execute(address target, bytes data, uint256 nonce, uint256 deadline)
// relayer request
interface RelayRequest { account: string; target: string; data: string; deadline: number; signature: string }
interface RelayResponse { txHash: string }
```

The relayer calls `PatientAccount.executeSigned(target, data, deadline, sig)`. A replay, a wrong signer, or an expired deadline reverts.

### PQC provider

All ML-DSA and ML-KEM calls go through one interface: `dsaKeygen(seed?)`, `sign(message, secretKey)`, `verify(sig, message, publicKey)`, `kemKeygen(seed?)`, `encapsulate(publicKey)`, and `decapsulate(cipherText, secretKey)`. The default backend is `NoblePQCProvider` over `@noble/post-quantum`, pinned to an exact version. The argument orders above were confirmed against version 0.7.1 in the v1.2 review.

## Key hierarchy and relationships

One root secret S creates every key a party owns, and the entity table shows how the stores join.

### Key hierarchy (HKDF-SHA256 from S)

&#91;embedded content: key hierarchy · one root, five derived keys\]

Non-patient parties (clinicians, labs, Hospital B, the demo Coordinator) derive their two key pairs the same way from their own secret. Keys cannot rotate without a new secret.

| Key | Derivation | Used for | Where it lives |
| --- | --- | --- | --- |
| S | 32 random bytes from a CSPRNG at vault creation | Root secret, split into n+1 shares | Passphrase keystore |
| Signing key pair | `ml_dsa65.keygen(HKDF(S, 'medichain-dsa-v1', 32))` | Record and audit signatures | Public key on-chain, secret derived at unlock |
| ML-KEM key pair | `ml_kem768.keygen(HKDF(S, 'medichain-kem-v1', 64))` | Receiving wrapped keys and shares | Public key on-chain, secret derived at unlock |
| typeKey | `HKDF(S, 'medichain-type-v1:' + resourceTypeName)`, for example `Observation`, never its hash (errata 5) | One per resource type, wrapped to grantees | The grantee reads it from `KeysWrapped` |
| recordKey | `HKDF(typeKey, 'medichain-record-v1:' + recordId)` | AES-GCM key for one record | Derived on demand |
| capsuleKey | `HKDF(S, 'medichain-capsule-v1')` | Emergency capsule only, never shared | Derived on demand |

A grantee holding one `typeKey` can read every record of that type, including later ones. It cannot derive S, another type's key, or `capsuleKey`. The v1.2 review confirmed on `@noble/post-quantum` 0.7.1 that key generation is deterministic from a seed, which is what lets recovery re-derive the same public keys, and that ML-KEM needs a 64-byte seed.

### Entity relationships

| From | To | Cardinality | Join |
| --- | --- | --- | --- |
| Owner wallet (v1.2) | PatientAccount | 1 to 1 | `PatientAccountFactory.accountOfOwner` |
| Identity | PatientAccount | 1 to 1 | `accountOf`, `patientIdOf` |
| Patient account | ConsentToken | 1 to many | `token.patient` |
| Grantee | ConsentToken | 1 to many | `tokenIdsOfGrantee` |
| ConsentToken | Resource type | many to many | `resourceTypes[]` |
| ConsentToken | KeysWrapped entry | 1 to 1 per type | `tokenId` in the event |
| Patient | RecordRef | 1 to many | `patientId` and `resourceType` |
| RecordRef | IPFS envelope | 1 to 1 | `cid`, checked with `contentHash` |
| RecordRef | Creator's identity | many to 1 | `creator`, then `getDilithiumKey` |
| Patient | GuardianSet | 1 to 1 | `patientId`, current `epoch` |
| ShareCommitment | ShareBundle | 1 to 1 | `index`, held off-chain by the trustee |
| Patient | Capsule | 1 to 1 | `patientId`. The encrypted payload is a RecordRef of type EmergencyCapsule |
| Patient | ClaimEntry | 1 to many | `patientId` and `claimType`. Authorised by a Claim ConsentToken |
| Patient | Break-glass or recovery request (held by RecoveryVault, errata 2) | 1 to many | `requestId` |
| Break-glass request in RecoveryVault | BGRequest in BreakGlassCapsule | 1 to 1, break-glass only | `requestId`. BGRequest keeps only `windowExpiry` and `closed` (errata 2) |
| Request | AuditLogged event | 1 to many | `refId` |
| Request | CoordinatorRequest | 1 to 1 while open | `requestId` |

## Configuration schema

Ports are fixed: frontend 5173, Hospital B viewer 5174, coordinator and dealer launcher 4100, IPFS proxy 4101 *(v1.2)*, and Hardhat node 8545. The dealer launcher's token is generated at startup and handed to the frontend through local config, so it is not an `.env` value. Both Pinata variables appear with blank values in `.env.example`, and real values stay out of git (errata 15).

| Variable | Default | Notes |
| --- | --- | --- |
| `RPC_URL` | `http://localhost:8545` | Chain ID 31337 |
| Contract addresses | Set by the deploy script | One per contract, written after deploy. Includes `PatientAccountFactory` *(v1.2)* |
| Relayer account key | None | Never commit |
| `PINATA_JWT` | None | Pinata API token (errata 15). Read by the IPFS proxy only. Never commit, and never place it in a frontend build |
| `PINATA_GATEWAY` | None | Pinata gateway setting (errata 15). Read by the IPFS proxy only |
| `DEMO_WINDOW_DURATION` | 180 | Seconds |
| `DEMO_TIER2_DELAY` | 60 | Seconds |
| `COLLECTION_TIMEOUT` | 20 | Seconds, off-chain only |
| `FULL_MODE_TTL` | 300 | Seconds, off-chain only. Lost-key (full mode) requests (errata 18) |

Hardhat's own config, not the environment, sets the block clock: `networks.hardhat.mining = { auto: true, interval: 1000 }` (D11) *(v1.2)*. Automine stays on, so transactions confirm at once, and the interval only adds empty blocks so block time follows wall-clock time. The demo never advances the clock. Tests that need to run on a separate in-process network.

## Resolved decisions

Items 1 to 19 of the errata file are resolved and applied in this version, and the TRD v1.9 now carries them too. Nothing from the errata file remains open. Decisions D10 to D23 are accepted.

### Errata items 1 to 19

| # | Item | Owner | Outcome | Applied in |
| --- | --- | --- | --- | --- |
| 1 | `RecoveryVault.setReleaseMode` was labelled internal | Rohith, Yagna | External, with an `onlyBreakGlassCapsule` check. The capsule contract's address is set at deploy through `setBreakGlassCapsule` (errata 16) | RecoveryVault |
| 2 | Break-glass request state split across two contracts | Yagna, Rohith | `setReleaseMode(requestId, patientId, requester, mode)`. RecoveryVault is the source of truth for patientId, requester, release mode, cancelled, and tier2EndsAt. BreakGlassCapsule keeps `windowExpiry` and `closed`, and `getRequest` reads the rest from the vault | RecoveryVault, BreakGlassCapsule, entity relationships |
| 3 | `roleId` is a hash but RoleRegistry uses an enum | Anush | `roleId = keccak256(bytes(roleName))`, with a name lookup from the enum in `checkAccess` | Identifiers, RoleRegistry, ConsentRegistry |
| 4 | Share encoding | Rohith | `shareBytes` is the UTF-8 bytes of the exact string from `secrets.share()`, never hex-decoded. Commitment is SHA-256 of `utf8(shareString)`, salt, and `utf8('medichain-share-v1')`, concatenated in that order | Constants, RecoveryVault, share bundle, dealer launcher |
| 5 | Other byte-level choices | Rohith, Anush | `patientId = keccak256(utf8(didString))`. HKDF info uses the resource type name. WrappedTypeKey serializes as UTF-8 JSON | Identifiers, wrapped type key, key hierarchy |
| 6 | Id formula | Anush, Yagna, Punith | `keccak256(abi.encode(block.chainid, address(this), msg.sender, counter))` with a per-contract counter | Identifiers, ConsentRegistry, RecordRegistry, RecoveryVault, BreakGlassCapsule |
| 7 | Capsule values | Yagna, Rohith | `CapsulePayload` adopted. The client library encrypts it under `capsuleKey` and registers it under EmergencyCapsule | Emergency capsule |
| 8 | Audit refIds and TierEscalated emitter | Punith, Yagna | refIds as in the logging table. `custodianCoSign` emits TierEscalated | Events and audit mapping |
| 9 | Signed audit record | Rohith | Optional `trustees` field. Append-only JSONL file per patient. The coordinator signs with a registered demo Coordinator identity, listed in the fixture | Signed audit record, demo fixture |
| 10 | Guardian set metadata | Rohith, Yagna | `getGuardianSet(patientId)` returns threshold, epoch, and updatedAt | RecoveryVault |
| 11 | ReleaseMode has no unset value | Rohith, Yagna | `None = 0`, `CapsuleOnly`, `FullRecord`. Tests and UI use names, never numbers | RecoveryVault |
| 12 | Trustee mix: warn or revert | Rohith | The contract rejects and the UI warns first. PRD T1-5 now says so (reworded in PRD v1.6) | RecoveryVault |
| 13 | FHIR Patient has no subject element | Rohith | The subject rule applies to every resource except Patient, where `id` plays that role | FHIR resources |
| 14 | Signature covers the ciphertext only | Rohith | Kept as in the TRD, a documented limitation. Limit 18 covers records authored by others | Record envelope |
| 15 | Pinata variable names | Rohith | `PINATA_JWT` and `PINATA_GATEWAY` in `.env.example` | Configuration |
| 16 | `breakGlassCapsule` has no setter, because the capsule contract is deployed after the vault | Rohith, Yagna | `setBreakGlassCapsule(address)`: admin-only, callable once, reverts if already set. The deploy script calls it after both contracts exist | RecoveryVault |
| 17 | `completeRecovery` has no mode check, so a break-glass responder could log RecoveryCompleted | Rohith | `completeRecovery` requires the mode to be FullRecord | RecoveryVault |
| 18 | A lost-key (full mode) request has no on-chain window | Rohith, Yagna | Coordinator-side `FULL_MODE_TTL`, demo default 300 s. The event's `windowExpiry` mirrors that off-chain deadline, and the coordinator wipes keys at the TTL | Constants, coordinator API, coordinator memory, configuration |
| 19 | `closeWindow`'s unknown-request check is `windowExpiry != 0`, because `requester` no longer lives in BreakGlassCapsule | Yagna | Accepted. A lost-key `requestId` reverts as unknown | BreakGlassCapsule |

### Decisions D10 to D23 *(v1.2)*

| ID | Decision | Status | Owner | Applied in |
| --- | --- | --- | --- | --- |
| D10 | A `PatientAccountFactory` creates one account per owner wallet and resolves wallet to account | Accepted | Anush | Identifiers, PatientAccountFactory, entity relationships, relayer |
| D11 | One clock: one-second block mining, deadlines read block time, no clock jumps in the demo | Accepted | Yagna, Rohith | Constants, coordinator API, configuration |
| D12 | Client library for record crypto, and a loopback IPFS proxy that sees only ciphertext | Accepted | Rohith | Scope, IPFS proxy, configuration |
| D13 | `canAccess` view plus the logged `checkAccess`, and demo identities on Hardhat's unlocked accounts | Accepted | Anush | ConsentRegistry, logging table, demo fixture |
| D14 | A category cap: no category on k or more of the n+1 trustees | Accepted | Rohith, Yagna | Constants, RecoveryVault |
| D15 | `register` binds the DID hash to the caller's own address | Accepted | Anush | Identifiers, PatientIdentity |
| D16 | FHIR validity: id pattern, DID as `identifier`, valid complex extension, Consent as projection | Accepted | Rohith | Identifiers, vocabularies, FHIR resources |
| D17 | Loopback services use a session token and an origin check, and the browser reads the stream with a fetch stream | Accepted | Rohith, Yagna | Service interfaces |
| D18 | The IPFS proxy's read-only GET route accepts the Hospital B origin. Pinning stays frontend-only | Accepted | Rohith | Service interfaces |
| D19 | A proof request shape, and an EligibilityVerified event emitted by ProofGateway on success | Accepted | Punith, Anush | Capsule mode, proof request, and tokens |
| D20 | Capsule-mode authorization: a requester signature over a challenge, on-chain mode and window checks, a capsuleFields view, and register-then-authorize saving order | Accepted | Rohith, Yagna | Capsule mode, proof request, and tokens |
| D21 | Guards on custodianCoSign, and limit 20 (the Tier 2 delay is not cryptographic) | Accepted | Yagna, Rohith | Capsule mode, proof request, and tokens |
| D22 | A session token and a per-request token, and shares also wrapped to the request's one-time key | Accepted | Rohith, Yagna | Service interfaces |
| D23 | 8-bit range checks and a threshold below 256, ProofService as a benchmark and seed helper, and a late k-th share rebuilding at once | Accepted | Punith, Rohith | Capsule mode, proof request, and tokens |
