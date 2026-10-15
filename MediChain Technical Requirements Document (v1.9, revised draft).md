# MediChain: Technical Requirements Document (v1.9, revised draft)

Oct 13, 2026 · @Abc

## Purpose, scope, and conventions

This TRD says how MediChain is built: concrete data structures, contract interfaces, cryptographic parameters, and module interface contracts, so the four contributors implement one shared design. It is the technical companion to PRD v1.6. The PRD says what and why.

| Item | Value |
| --- | --- |
| Version | 1.9, revised draft. Folds in errata 1 to 19 and the accepted decisions D10 to D23 |
| Date | 13 October 2026 (v1.8 was dated 3 October 2026) |
| Companion documents | PRD v1.6, App Flow v1.7, Backend Schema v1.2, Architecture v1.2 |
| Team | Group 02, Department of CSE, BGS College of Engineering and Technology, Bengaluru |
| Course | Project Phase-I (BCS685). Confirm the label is still right for the current semester |
| Authors | Anush G, Punith A, Rohith Gowda R, Yagna Maurya M V |

Precedence: this TRD now agrees with the errata file, so it can be read alone. This TRD supersedes the errata file, which stays in the repository for history only. Where documents still differ, this TRD wins on behavior, and the Backend Schema wins on exact field and function shapes.

Every specification traces back to PRD requirement IDs (PAT-, CON-, ZKP-, AUD-, DAT-, T1-, BG-, UI-, INT-, SEC-, E2E-). The last section maps every requirement to the section that specifies it.

### Conventions

- MUST, SHOULD, and MAY are used informally in the RFC 2119 sense. MUST is a hard constraint the demo and tests enforce, SHOULD is a strong default the team can override with reason, and MAY is optional.
- *(specified here)* marks a detail this document fixed that the module plans left open: exact parameters, function signatures, and gas or latency budgets. The team accepted these in v1.2.
- *(check live)* marks something that depends on a third-party package's current API or version. Check it against the live package before coding, and repeat the first-day checks on each team machine.
- AF-n is a resolution accepted in the App Flow. Each is specified here.
- *(v1.9)* marks a row or heading that changed in this version.
- A TypeName is a Solidity type. snake\_case or camelCase in a code block is JavaScript, TypeScript, or JSON.
- v1.8 described IDs prefixed TR-. None were ever used, so the convention is dropped.

### Environment assumption

Everything here targets one local Hardhat network on one developer machine, per the PRD's non-goals. There is no testnet, mainnet, or multi-node target in this phase.

### What was and was not run

- **Run, on skeletons:** the claim circuit compiled with circom 2.2.3 to 531 constraints (253 non-linear, 278 linear; 3 public inputs, 2 private, no outputs). A local single-contributor setup produced a proving key. Real proofs verified off-chain, and a value below the threshold, a mismatched commitment, a wrong salt, or a value or threshold outside 8 bits each failed to produce a proof. The public signal order is \[threshold, commitment, nonce\].
- **Run, on skeletons:** the generated verifier and a gateway skeleton compiled with solc 0.8.24 (Cancun) against OpenZeppelin 5.6.1. 29 Hardhat tests passed on Hardhat 2.29.1 against skeleton contracts. The measured skeleton gas was 268,683 for one proof and 513,275 for a batch of two.
- **Not run:** the real contracts, the real RecoveryCoordinator and dealer launcher, the frontend, and the demo identities. Nothing here was tested against final code, and the skeleton files are not part of this document set. Add them to the repository next to these documents.

## System architecture

The system has three planes: state on-chain, computation off-chain, and encrypted bytes in storage. Nothing that touches plaintext PHI, a raw secret share, a salt, or a private key crosses from the client and service plane into the on-chain plane.

| Plane | Runs on | Holds | Never holds |
| --- | --- | --- | --- |
| On-chain | Hardhat local network, Solidity | Identity, roles, and the public-key registry. Consent tokens and access checks. Record references (CID and hash). Proof verification, claim commitments, and audit events. Share commitments only. Break-glass window and Tier 2 co-sign and delay state | Plaintext records, raw shares, salts, private keys |
| Off-chain service | Node.js processes, run by whoever needs them, plus the React client | FHIR construction, AES-GCM, Dilithium sign and verify, ML-KEM wrap and unwrap, circom witness and snarkjs proofs, Shamir dealing (one-shot) and reconstruction (on demand). The root secret S sits in a passphrase-encrypted device keystore. Derived keys live in tab memory only | Any persistent plaintext secret. Nothing reaches a service unless it is wrapped to a one-time key |
| Storage and custody | IPFS through Pinata, and trustee devices | Encrypted record envelopes. Encrypted share bundles, one per trustee | Anything readable without a key held somewhere else |

See Recovery and break-glass technical flow for why shares specifically must never reach the on-chain plane.

### Component catalog

Twenty components; two are new in v1.9.

| Component | Plane | Responsibility | Owner |
| --- | --- | --- | --- |
| PatientIdentity.sol | On-chain | Wallet and DID link, public-key registry (PAT-1, PAT-2, PAT-5, PAT-6, PAT-8). Doctors, labs, and Hospital B register the same way | Anush |
| PatientAccount.sol | On-chain | ERC-4337-style account: the patient signs, a relayer pays gas (PAT-3, AF-11) | Anush |
| PatientAccountFactory.sol (new, D10) | On-chain | Deploys one PatientAccount per owner wallet at a precomputable address and resolves wallet to account (PAT-7) | Anush |
| Relayer | Off-chain, Node.js | Submits patient-signed actions, creates accounts through the factory, and pays gas on the local network (AF-11) | Anush |
| Device keystore | Client | Passphrase-encrypted store for one secret: the patient's record secret S, or a clinician's, lab's, or second institution's own secret (AF-3, DAT-9, AF-19) | Rohith, Yagna |
| Hospital B viewer | Client, React | Second small app on its own port that consumes the same tokens, using a demo identity (INT-1, INT-3, AF-2, AF-19) | Rohith |
| Seed script and demo identities | Off-chain, Node.js | Creates the demo identities (Dr. Rao, a lab, Hospital B) on Hardhat's unlocked accounts, registers their public keys, has the scripted patient share a Claim permission with Dr. Rao, has Dr. Rao issue the demo claim, and writes their secrets to a git-ignored, demo-only fixture (DAT-9, ZKP-7, AF-19) | Rohith |
| RecordRegistry.sol | On-chain | Registers and lists encrypted-record references by patient and resource type (DAT-2, DAT-8, AF-13) | Anush |
| RoleRegistry.sol | On-chain | Minimal ABAC role registry; ships early per the PAT-4 ordering fix | Anush |
| ConsentRegistry.sol | On-chain | Consent tokens, full ABAC wiring, proof-verification hook (CON-1 to CON-6) | Anush |
| ProofVerifier.sol (generated) and ProofGateway.sol | On-chain | Groth16 verification of an issued-claim proof, claim commitments, nonce replay protection, batching (ZKP-2 to ZKP-4, ZKP-7) | Punith |
| AuditLog.sol | On-chain | Canonical append-only event log (AUD-1) | Punith |
| RecoveryVault.sol | On-chain | k-of-(n+1) share commitments, Tier 2 co-sign and delay gate, lost-key requests, and the source of truth for request state (T1-1 to T1-10, BG-6) | Rohith, Yagna |
| BreakGlassCapsule.sol | On-chain | Emergency role check, capsule schema, time-boxed window (BG-1 to BG-5, BG-8) | Yagna |
| Client data library (was DataService) (v1.9, D12) | Client and Node scripts | FHIR build, AES-GCM, Dilithium sign and verify, ML-KEM type-key wrapping (DAT-1 to DAT-6), the record registry calls (DAT-8), and keystore handling for non-patient parties (DAT-9). Runs wherever the keys are, never as a service that receives them | Rohith |
| IPFS proxy (new, D12) | Off-chain, Node.js, 127.0.0.1:4101 | Holds `PINATA_JWT` and `PINATA_GATEWAY`, pins ciphertext envelopes, and fetches them by CID. Sees ciphertext only (DAT-10) | Rohith |
| ProofService | Off-chain, Node.js | A benchmark and seed helper for witness and proof generation and batching (ZKP-1, ZKP-5). Not one of the demo's processes, because the browser generates the demo's proofs (D23) | Punith |
| RecoveryDealer | Off-chain, one-shot CLI | Setup-phase dealing only, never a running service (T1-1 to T1-4) | Rohith |
| RecoveryCoordinator | Off-chain, Node.js | Recovery-phase share collection, commitment check, Lagrange reconstruction (T1-7, T1-8, T1-10) | Rohith, Yagna |
| Frontend SPA | Client, React | Patient dashboard, records list, responder view, clinician view, scripted guardian flow (UI-1 to UI-9) | Yagna |

### How the contracts depend on each other

Every state change that matters ends in an `AuditLog` call. If that call reverts, the whole transaction reverts, so nothing succeeds without being logged. Rows marked v1.9 add dependencies the v1.8 catalog missed.

| Contract | Reads or calls | Called by |
| --- | --- | --- |
| PatientAccountFactory (v1.9) | Deploys PatientAccount | The relayer, the deploy script, anyone (the owner is a parameter) |
| PatientAccount | Any target contract, on the patient's behalf | The relayer (`executeSigned`), the owner wallet (`execute`) |
| PatientIdentity | Nothing | RecordRegistry, RecoveryVault, ConsentRegistry, ProofGateway (v1.9), BreakGlassCapsule (v1.9), and every client that verifies a signature or wraps a key |
| RoleRegistry | Nothing | ConsentRegistry, ProofGateway, BreakGlassCapsule |
| ConsentRegistry | PatientIdentity, RoleRegistry, AuditLog, ProofGateway (the proof hook set by `setProofVerifier`) | Patients, grantees, ProofGateway (token reads) |
| RecordRegistry | PatientIdentity, AuditLog | The patient's account, every reader |
| ProofGateway | ProofVerifier, ConsentRegistry, RoleRegistry, PatientIdentity (v1.9), AuditLog | Issuers, verifiers, the patient |
| RecoveryVault | PatientIdentity, AuditLog | BreakGlassCapsule (`setReleaseMode`), the deploy script (`setBreakGlassCapsule`, once), the coordinator (reads), patients, guardians, the custodian |
| BreakGlassCapsule | RoleRegistry, PatientIdentity (v1.9), AuditLog, RecoveryVault (`setReleaseMode` and request-state reads) | Emergency responders, anyone for `closeWindow` |
| AuditLog | Nothing | Every allow-listed logging contract |

The `proofVerifier` field in ConsentRegistry holds the ProofGateway's address, not the generated verifier's. Renaming it to `proofGateway` removes the confusion and is recommended.

## Technology stack and versions

The stack is Solidity 0.8.24 on Hardhat 2, circom with snarkjs, `@noble/post-quantum` for ML-DSA and ML-KEM, and React 18 with Vite. Pin exact patch versions in each module's `package.json` at implementation time. The table gives the recommended line and a reason.

| Area | Choice | Notes |
| --- | --- | --- |
| Solidity | 0.8.24 (or any current 0.8.x) | Built-in overflow and underflow checks, so no SafeMath |
| Contract framework | Hardhat 2.x, pinned: `hardhat@^2.28`, `@nomicfoundation/hardhat-toolbox@^6`, `@nomicfoundation/hardhat-ethers@^3.1.3`, `ethers@^6.14` | Pin the major explicitly, because an unpinned `npm install hardhat` now resolves to Hardhat 3. Hardhat 3 is a rewrite, not a version bump: ESM-first projects, `defineConfig()` with an explicit plugins array, ethers from `network.connect()`, changed Chai matchers, and its own plugin lines. A clean install of exactly this set, with no `--legacy-peer-deps`, resolved to hardhat 2.29.1, toolbox 6.1.2, hardhat-ethers 3.1.3, and ethers 6.17.0 on 1 October 2026. Every config, test, and deploy pattern here assumes Hardhat 2, and there is no testnet target, so migrating buys nothing |
| Chain clock (v1.9, D11) | `networks.hardhat.mining = { auto: true, interval: 1000 }` | Mines a block every second even with no transactions, so `block.timestamp` follows wall-clock time and countdowns do not freeze. The demo node uses this setting. Tests that call `evm_increaseTime` run on a separate in-process network *(check live)* |
| Contract libraries | ethers.js ^6 and OpenZeppelin Contracts ^5 (`AccessControl`, `ReentrancyGuard`, `EIP712`, `ECDSA`, `Strings`) | OpenZeppelin is the base for access control, reentrancy protection, the account's signing, and the DID string check. Recent 5.x releases (tested with 5.6.1) use the Cancun `mcopy` opcode, so Hardhat's Solidity settings MUST set `evmVersion: 'cancun'` with compiler 0.8.24 or later. Otherwise compilation fails with "Function mcopy not found" |
| Zero-knowledge circuits | circom 2.1.x or later, snarkjs 0.7.x or later, circomlib and circomlibjs (Poseidon), Groth16, curve bn128 (alt\_bn128) | Compiled with circom 2.2.3, snarkjs 0.7.6, circomlib 2.0.5, and circomlibjs 0.1.7 in October 2026. Punith should still confirm on his machine with `circom --version` and `npm view snarkjs version` *(check live)* |
| Storage | IPFS via the Pinata SDK, behind the IPFS proxy | Credentials live only in the proxy's environment (v1.9, D12). Confirm current plan limits on file count and size before the demo, and do not assume a specific quota |
| Data model | HL7 FHIR R4-inspired JSON | Patient and Observation resources. Consent is a read-only projection of the on-chain token (DAT-1) |
| Evaluation data | Synthea, exported as FHIR R4 | Primary dataset per PRD decision D6 |
| Symmetric encryption | AES-256-GCM via Node's built-in `node:crypto` | No external dependency needed |
| Hash commitments | SHA-256 via `node:crypto` | See Cryptographic specification for the exact input format |
| Secret sharing | secrets.js-grempe (Shamir over GF(2^8)) | The default 8-bit share-index space (up to 255 shares) is far more than n+1 ever needs. It reports a CSPRNG backed by Node's `crypto.randomBytes`, which satisfies T1-1. Maintenance: the last release (2.0.0) was published in September 2019, so it is effectively unmaintained. It is small and passed an end-to-end test of this TRD's recovery flow, which is acceptable for a prototype. List it as a limitation in the report |
| Post-quantum cryptography | `@noble/post-quantum` (pure TypeScript and JavaScript, no WASM build step) implementing ML-KEM and ML-DSA | Pre-1.0 (0.7.x at time of writing), so pin an exact version and expect API changes between minors. Requires Node.js 20.19 or later. Subpath imports must include the `.js` extension: `@noble/post-quantum/ml-kem.js` and `/ml-dsa.js` export `ml_kem512/768/1024` and `ml_dsa44/65/87`. Without the extension Node throws `ERR_PACKAGE_PATH_NOT_EXPORTED` |
| Device keystore | PBKDF2-HMAC-SHA256 with at least 600,000 iterations, then AES-256-GCM (Web Crypto in the browser, `node:crypto` in tests) | 600,000 is OWASP's published minimum for PBKDF2-HMAC-SHA256 at time of writing; check current guidance. Measured at about 120 ms on one Node 22 run |
| Local services | Plain Node `http` with Server-Sent Events, bound to 127.0.0.1 | No extra dependency. Browsers read the event stream with a fetch stream, not `EventSource`, so the bearer token can go in a header (v1.9, D17). See Module interface contracts |
| Frontend | React 18, Vite as the build tool, ethers.js ^6, an injected wallet (MetaMask or equivalent) | The module plan only said "React/HTML/JS". Vite is the chosen build tool |
| Testing | Hardhat with Chai and Mocha matchers for contracts, Jest for Node services | Jest is named in Rohith's plan. `@noble/post-quantum` is ESM-only and Jest's default CommonJS mode cannot load it, so set up Jest's ESM mode in the Foundations phase and confirm on day one (v1.9, *check live*) |
| Runtime | Node.js 20.19 or later (22.x also works) | 20.19 is the minimum `@noble/post-quantum` supports. It is also the first 20.x release where `require()` can load an ES module without a flag, so CommonJS Hardhat 2 scripts can import the package directly. Older 20.x releases fail |

**Change from the module plans: post-quantum library.** The original plan named a WASM-based library (liboqs). This TRD recommends `@noble/post-quantum` as the primary backend, because it is pure TypeScript with no WASM toolchain to build or ship, is far smaller (tens of KB gzipped), and directly implements ML-KEM (FIPS 203) and ML-DSA (FIPS 204). The calls sit behind one small interface (see Cryptographic specification), so a WASM-based binding can still be swapped in later without touching call sites. That keeps the library-maturity risk mitigation intact. The module plan files that still name liboqs should be corrected.

## Data schemas

These are the concrete shapes every contract and service section refers to. Field lists are the minimum for the prototype: extend rather than rename. The Backend Schema v1.2 repeats them in storage-by-storage form.

### Identifiers

| Name | Type | Meaning |
| --- | --- | --- |
| patientId | bytes32 | `keccak256(utf8(didString))`, where the DID string is `did:medichain:` followed by the lowercase `0x` hex address of the registering account. `register` checks this binding on-chain (PAT-8, D15) |
| account | address | The patient's `PatientAccount`. It is `msg.sender` in every other contract, so it is the patient's on-chain identity |
| owner | address | The wallet key that owns a `PatientAccount` and signs EIP-712 messages |
| party | address | Any registered identity: patient account, doctor, lab, Hospital B. Key lookups are by address |
| resourceType | bytes32 | `keccak256` of a name: Patient, Observation, Consent, EmergencyCapsule, Claim. Consent and Claim are permission categories, never stored records. "Claim" is MediChain's category and has nothing to do with the FHIR billing resource |
| roleId | bytes32 | `keccak256(bytes(roleName))` over Doctor, Lab, Insurer, IoTDevice, EmergencyResponder. `checkAccess` maps it back to the `Role` enum by name (errata 3) |
| tokenId, entryId, requestId | bytes32 | `keccak256(abi.encode(block.chainid, address(this), msg.sender, counter))` with a per-contract counter (errata 6) |
| recordId | string | The client-side FHIR resource id. It is known before upload and is the input to key derivation. It is not the registry's `entryId` |
| epoch | uint32 | Generation counter of a patient's share set |

### FHIR-inspired resources (DAT-1)

Trimmed R4-style JSON, with three rules. `resourceType` and `id` are mandatory on every resource. A `subject` reference is mandatory on every resource except Patient, where `id` plays that role (errata 13). Ids match the FHIR id pattern, letters, digits, hyphen, and dot, at most 64 characters, so a DID can never be an id. A party's DID travels as an `identifier` (v1.9, D16). Synthea's UUID ids already fit the pattern, and the seed script rewrites `urn:uuid:` references to `Patient/<id>`.

```json
{
  "resourceType": "Observation",
  "id": "obs-0142",
  "status": "final",
  "subject": { "reference": "Patient/pat-0001" },
  "code": { "text": "Blood glucose" },
  "valueQuantity": { "value": 98, "unit": "mg/dL" },
  "effectiveDateTime": "2026-09-12T08:15:00Z",
  "performer": [{
    "type": "Practitioner",
    "identifier": { "system": "https://medichain.example/did", "value": "did:medichain:0xb2..." }
  }]
}
```

Consent is not a stored resource. It is a read-only projection of an on-chain `ConsentToken` that screens and the Hospital B viewer can show: `id` is the tokenId, `status` is `active` or `inactive` (revoked or expired), and a `chain` block carries the chain id, registry address, and tokenId.

### Off-chain record envelope (DAT-2, DAT-4)

This is what gets uploaded to IPFS. The `signerId` lets a reader look up the right public key through `PatientIdentity.getDilithiumKey()` (PAT-5) without guessing who signed.

```json
{
  "schemaVersion": "1.0",
  "resourceType": "Observation",
  "recordId": "obs-0142",
  "ciphertext": "<base64, AES-256-GCM of the FHIR JSON above, under recordKey>",
  "iv": "<base64, 12 bytes, never reused under one key>",
  "authTag": "<base64, 16 bytes>",
  "signature": { "alg": "ML-DSA-65", "signerId": "did:medichain:0xb2...", "value": "<base64>" }
}
```

`resourceType` is the name, not the hash. The on-chain `contentHash` is `sha256(ciphertext)`. A reader checks the hash first, then the signature against `getDilithiumKey(creator)`, then decrypts. A failed GCM tag aborts with no partial output. The signature covers the ciphertext only. That is a documented limitation (errata 14), and limit 18 covers records authored by someone other than the patient.

### On-chain record reference (DAT-2, DAT-8, AF-13)

Registered by `RecordRegistry.addRecord` and listed by `recordsOf`. `entryId` is the registry's own id. The client-side `recordId` inside the envelope is separate.

```solidity
struct RecordRef {
    bytes32 entryId;      // registry id, assigned on add (errata 6)
    bytes32 patientId;
    bytes32 resourceType; // keccak256('Observation'), keccak256('EmergencyCapsule')
    string  cid;          // the real CID string, v0 or v1, 1 to 100 bytes
    bytes32 contentHash;  // sha256(ciphertext), an integrity check independent of IPFS
    address creator;      // the signer, per DAT-4 and decision D9
    uint64  createdAt;
}
```

`revoked` was dropped: nothing sets it, and deleting a record is out of scope.

### Consent token (CON-1 to CON-6)

```solidity
struct ConsentToken {
    bytes32 tokenId;          // errata 6
    address patient;          // the patient's account address, not the owner wallet
    address grantee;
    bytes32 roleId;           // keccak256(bytes(roleName)), errata 3
    bytes32[] resourceTypes;  // e.g. keccak256('Observation')
    uint64 issuedAt;
    uint64 expiresAt;
    bool revoked;
}
```

A token is valid when it is not revoked and not expired. `EmergencyCapsule` can never be granted. A Claim token carries an empty wrapped-key entry.

### Wrapped type keys (DAT-5)

Sharing is by record type, so the grantee receives one wrapped `typeKey` per resource type in the token, not one key per record. `ConsentRegistry.issueConsent` emits them in a `KeysWrapped` event (never in contract storage), and the grantee reads the event by `tokenId`. Each entry is an ML-KEM encapsulation plus the AES-GCM-wrapped key, serialized as UTF-8 JSON (errata 5).

```solidity
event KeysWrapped(
    bytes32 indexed tokenId,
    address indexed grantee,
    bytes32[] resourceTypes,
    bytes[] wrappedTypeKeys   // same length and order as resourceTypes
);
```

```json
{ "kemAlg": "ML-KEM-768", "encapsulatedKey": "<base64>", "iv": "<base64>", "wrappedTypeKey": "<base64>", "authTag": "<base64>" }
```

A proof-based share has no resource types and so no wrapped keys.

### Share commitment (on-chain) and share bundle (off-chain): T1-2, T1-4, T1-7, decision D4

The commitment is public and safe on-chain, because a hiding and binding hash commitment reveals nothing about the share. The bundle, including the salt, is never on-chain. It lives only at the trustee, encrypted to that trustee's ML-KEM key.

```solidity
// On-chain, one row per trustee including the institutional custodian.
// v1.9: fields reordered so trustee, category, and index share one storage slot.
struct ShareCommitment {
    bytes32 commitment;         // sha256(utf8(shareString) || salt || utf8('medichain-share-v1')), errata 4
    address trustee;
    TrusteeCategory category;   // Family, HospitalCustodian, GuardianService, InstitutionalCustodian
    uint8 index;                // 1..n guardians, n+1 = institutional custodian
}
```

```json
{
  "trusteeId": "guardian-1",
  "shareIndex": 1,
  "kemAlg": "ML-KEM-768",
  "encapsulatedKey": "<base64, wrapped to the trustee's ML-KEM public key>",
  "iv": "<base64, 12 bytes>",
  "ciphertext": "<base64, AES-256-GCM of { share, salt } under the KEM shared secret>",
  "authTag": "<base64, 16 bytes>"
}
```

`share` in the plaintext is the exact string `secrets.share()` returned. It is never hex-decoded, because those strings have odd length (99 characters for a 32-byte secret) and `Buffer.from(share, 'hex')` silently drops the last nibble (errata 4). The salt is raw bytes, at least 16 of them.

### Audit event (AUD-1 to AUD-3)

```solidity
enum ActionType {
    ConsentIssued, ConsentRevoked, AccessGranted, AccessDenied,
    ProofVerified, BreakGlassTriggered, BreakGlassExpired,
    RecoveryInitiated, TierEscalated, RecoveryCompleted,
    RecoveryCancelled,  // AF-6
    RecordAdded,        // RecordRegistry.addRecord
    ClaimIssued,        // ProofGateway.setCommitment
    ClaimRevoked        // ProofGateway.revokeClaim. Appended last so existing values keep their numbers
}

event AuditLogged(
    bytes32 indexed patientId,
    address indexed actor,
    ActionType indexed action,
    bytes32 refId,
    uint64 timestamp
);
```

`ClaimRevoked` is both an event name (ProofGateway) and an `ActionType` value. They are separate things. The `refId` for each action is fixed (errata 8):

| refId | Actions |
| --- | --- |
| tokenId | ConsentIssued, ConsentRevoked, AccessGranted, AccessDenied, and ClaimIssued (the tokenId of the Claim token) |
| entryId | RecordAdded |
| requestId | ProofVerified, BreakGlassTriggered, BreakGlassExpired, RecoveryInitiated, TierEscalated, RecoveryCancelled, RecoveryCompleted |
| claimType | ClaimRevoked |

AUD-3's Dilithium-signed off-chain mirror carries the same fields plus a signature (see Signed audit record).

### Emergency capsule (BG-4, decision D1)

The capsule metadata names the fields. The values live in a separate payload that the client library encrypts under `capsuleKey` and registers in `RecordRegistry` under the `EmergencyCapsule` type (errata 7).

```json
{
  "capsuleId": "0x...",
  "patientId": "did:medichain:0xa1...",
  "fields": ["bloodType", "allergies", "currentMedications", "emergencyContacts"],
  "preAuthorizedAt": "2026-01-14T10:02:00Z",
  "version": 1
}
```

```ts
interface CapsulePayload {  // the plaintext inside the capsule envelope
  capsuleId: string; patientId: string; version: number; preAuthorizedAt: string;
  fields: {
    bloodType?: string;
    allergies?: string[];
    currentMedications?: { name: string; dose?: string }[];
    emergencyContacts?: { name: string; relationship: string; phone: string }[];
  };
}
```

Only the fields the patient ticked are present. Break-glass releases this object and nothing else. See Recovery and break-glass technical flow for how the release mode is enforced.

### Device keystore file (AF-3, DAT-9)

Held in the browser (IndexedDB) and exportable as a backup file. It holds one secret: the record secret S. The signing key pair and the ML-KEM key pair are derived from S (see Key hierarchy), so they are not stored.

```json
{
  "version": 1,
  "kdf": "PBKDF2-SHA256",
  "iterations": 600000,
  "salt": "<base64, 16 bytes>",
  "iv": "<base64, 12 bytes>",
  "ciphertext": "<base64, AES-256-GCM of { recordSecret }>",
  "authTag": "<base64, 16 bytes>"
}
```

A wrong passphrase fails the GCM authentication tag, so there is no separate password check to attack. The file is a few hundred bytes. A backup file is a convenience for the same or a replacement device, not a substitute for guardians: a lost passphrase with no backup goes through the guardian recovery flow. A clinician, lab, or second-institution endpoint uses the same format with its own secret in place of S (DAT-9), derives its two key pairs by the same HKDF scheme, and registers the public keys like any party. Keys cannot rotate without a new secret.

### Issued claim (ZKP-7)

A claim is a number an issuer commits to for a patient, such as an age. The issuer, an account holding the Doctor or Lab role and an active Claim permission from the patient, registers only the commitment on-chain. The value and salt reach the patient inside the encrypted Patient record the issuer creates, as a valid FHIR complex extension (v1.9, D16):

```json
{
  "resourceType": "Patient",
  "id": "pat-0001",
  "identifier": [{ "system": "https://medichain.example/did", "value": "did:medichain:0xa1..." }],
  "extension": [{
    "url": "https://medichain.example/fhir/StructureDefinition/issued-claim",
    "extension": [
      { "url": "claimType", "valueString": "AgeOver" },
      { "url": "value", "valueInteger": 34 },
      { "url": "salt", "valueString": "0x..." },
      { "url": "issuer", "valueString": "did:medichain:0xb2..." }
    ]
  }]
}
```

The salt is 31 random bytes, so it is always a valid field element. The commitment is `Poseidon(value, salt)` over the BN254 scalar field, registered by `ProofGateway.setCommitment(patientId, claimType, commitment, tokenId)`. `claimType` is `keccak256` of a short name such as `AgeOver`. The value and any threshold must each fit in 8 bits.

**Permission to issue.** Before an issuer can register a commitment, the patient issues a consent token that names the issuer for the resource type `Claim` (`keccak256('Claim')`). The issuer reads nothing, so for this type the wrapped-key entry in `issueConsent` is empty. Only the original issuer may replace a commitment, and the patient or the issuer can call `revokeClaim`. Issuing and revoking are logged as `ClaimIssued` and `ClaimRevoked`. Records authored by someone other than the patient are simulated by the seed script in the demo. Known limits describes the production route.

### Signed audit record, off-chain (AUD-3, AF-9)

Records are stored as an append-only JSONL file per patient. The coordinator assembles each record and signs it with a registered demo Coordinator identity from the fixture (errata 9).

```json
{
  "patientId": "0x...",
  "actor": "0x...",
  "action": "BreakGlassTriggered",
  "refId": "0x...",
  "timestamp": 1790000000,
  "attestation": true,
  "trustees": [{ "index": 1, "category": "Family", "status": "responded" }],
  "signature": { "alg": "ML-DSA-65", "signerId": "did:medichain:0x...", "value": "<base64>" }
}
```

`attestation` records that the responder ticked "this patient cannot give consent". It is stored here, not enforced by the contract, because `triggerEmergencyAccess` takes only the patient ID. The report must not claim the contract enforces it. The optional `trustees` list carries each trustee's index, category, and a status of waiting, responded, invalid, or timeout. Sentences such as "2 of 4 guardians responded" come from this list, because no on-chain event carries them.

### Emergency card payload (AF-8)

The code on a patient's emergency card or bracelet carries only the patient ID, never medical data: `medichain://patient/<patientId as 0x-prefixed hex>`. The responder screen scans it where the device camera allows, and offers manual entry as a fallback.

### Demo identity fixture (DAT-9)

`fixtures/demo-identities.DEMO-ONLY.json` is git-ignored and never committed. Each identity is one of Hardhat's unlocked accounts, so a script or screen signs with a JSON-RPC signer for that address and the fixture holds no transaction-signing key (v1.9, D13).

```ts
interface DemoFixture {
  label: 'DEMO ONLY'; chainId: 31337;
  identities: {
    name: string;   // Dr. Rao, Dr. Mehta, Hospital B, Guardian 1, Coordinator
    role: 'Patient' | 'Doctor' | 'Lab' | 'EmergencyResponder' | 'Guardian' | 'Custodian' | 'Coordinator' | 'HospitalB';
    address: string; did: string;
    secret: string; // base64, 32 bytes: the party's own S
  }[];
}
```

## Smart contract specifications

Ten hand-written contracts are specified below, plus the generated verifier, with errata 1 to 19 folded in. Function tables give the signature, who may call it, and the rules. Solidity is ^0.8.24 with OpenZeppelin 5.x and `evmVersion: 'cancun'`. Every patient action reaches these contracts through `PatientAccount`, so a patient address below is an account address. Structs, enums, and the places where atomicity matters are code. The rest are tables, because a table is easier to keep in sync across four people.

### PatientIdentity.sol (Anush)

| Function | Access | Rules |
| --- | --- | --- |
| `register(bytes32 didHash, bytes dilithiumPubKey, bytes mlkemPubKey)` | external, any caller | One identity per address and per patientId. `didHash` MUST equal `keccak256(abi.encodePacked('did:medichain:', Strings.toHexString(msg.sender)))`, so nobody can register another party's identifier (PAT-8, D15). Stores ML-DSA-65 and ML-KEM-768 public keys |
| `updateKeys(...)` | not implemented | Keys derive from S, so they never change without a new secret |
| `accountOf(bytes32 patientId) view returns (address)` | external view | patientId to account |
| `patientIdOf(address account) view returns (bytes32)` | external view | Account to patientId |
| `getDilithiumKey(address party) view returns (bytes)` | external view | Empty bytes mean the address is not registered |
| `getMLKEMKey(address party) view returns (bytes)` | external view | Used when wrapping type keys and share bundles |

Doctors, labs, and Hospital B register the same way, from their own address, with keys derived from their own secret (DAT-9). Storage is `mapping(address => Identity)`, `mapping(bytes32 => address) accountOf`, and `mapping(address => bytes32) patientIdOf`.

### PatientAccountFactory.sol (Anush, new, D10)

The v1.8 documents said "one account per patient" without saying how an account is created. Without this contract the first screen cannot tell a patient from a stranger, because identity lives under the account address and the screen only knows the wallet.

| Function | Access | Rules |
| --- | --- | --- |
| `createAccount(address owner) returns (address account)` | external, anyone | Deploys a `PatientAccount` with `CREATE2`, using the owner as the salt. The owner is fixed in the constructor, so a caller gains no control. If the account already exists it returns the existing address. The relayer calls it on first connection and pays the gas |
| `predictAccount(address owner) view returns (address)` | external view | The address the account has or will have |
| `accountOfOwner(address owner) view returns (address)` | external view | The deployed account, or the zero address |

Routing rule for SCR-01: resolve `accountOfOwner(wallet)`. If it exists and `getDilithiumKey(account)` is non-empty, the wallet is a patient. A wallet that holds a role in `RoleRegistry` is a doctor, lab, or responder, and those register from their own address.

### PatientAccount.sol (Anush; PAT-3, AF-11)

The patient's on-chain account is a small contract owned by their wallet key. The patient signs an EIP-712 message, and a relayer submits it and pays gas, so the app never asks the patient to handle gas. This is the simplified, ERC-4337-style design from decision D7: no bundler or paymaster, with sponsorship simulated by a funded local account.

| Function | Access | Rules |
| --- | --- | --- |
| `execute(address target, bytes data)` | owner wallet only | Direct call, the fallback if sponsorship is dropped |
| `executeSigned(address target, bytes data, uint256 deadline, bytes sig)` | anyone (the relayer) | Reverts if the deadline passed, the signature is not the owner's, or the nonce is wrong. Increments the nonce before the call, so a replay reverts |
| `nonce() view returns (uint256)` | external view | Incremented before each `executeSigned` call |

```
// EIP-712 typed data the patient signs
domain:  { name: 'MediChainAccount', version: '1', chainId: 31337, verifyingContract: <account> }
message: Execute(address target, bytes data, uint256 nonce, uint256 deadline)
// In the struct hash, `data` enters as keccak256(data), as EIP-712 requires for bytes.
```

Every other contract sees the account's address as `msg.sender`, not the owner's wallet. So `PatientIdentity.register`, `ConsentRegistry.issueConsent`, `RecoveryVault`, and the rest MUST treat the account address as the patient's identity. The relayer can delay an action but cannot forge one, because it holds no key that can produce the signature. PAT-3 is P1. If the team drops it, patient actions fall back to direct wallet transactions and the wallet's own pop-up may show gas, which the app cannot hide.

### RoleRegistry.sol (Anush; ships early per the ABAC ordering fix)

```solidity
enum Role { None, Doctor, Lab, Insurer, IoTDevice, EmergencyResponder }
```

| Function | Access | Rules |
| --- | --- | --- |
| `assignRole(address account, Role role)` | external, admin only | Built on OpenZeppelin `AccessControl`. Minimal registry that ships in Core build so Yagna can build against it (PAT-4, BG-1). One role per account |
| `hasRole(address account, Role role) view returns (bool)` | external view | Called by BreakGlassCapsule and ProofGateway, and once wired, by ConsentRegistry |

ConsentRegistry's Advanced-phase ABAC wiring extends this registry rather than replacing it, so BreakGlassCapsule's Core-build dependency on `hasRole` keeps working unchanged.

### ConsentRegistry.sol (Anush)

| Function | Access | Rules |
| --- | --- | --- |
| `issueConsent(address grantee, bytes32 roleId, bytes32[] resourceTypes, uint64 expiresAt, bytes[] wrappedTypeKeys) returns (bytes32 tokenId)` | registered patient account only | `expiresAt` must be in the future. `wrappedTypeKeys.length` must equal `resourceTypes.length`. Rejects `EmergencyCapsule`. A Claim entry is empty. Emits `KeysWrapped` (keys are emitted, never stored). Logs ConsentIssued. `tokenId` per errata 6 |
| `revokeConsent(bytes32 tokenId)` | the issuing patient only | One storage update plus an audit event, ConsentRevoked |
| `isValid(bytes32 tokenId) view returns (bool)` | external view | Not revoked and not expired. Expiry needs no action, because it compares against the stored `expiresAt` |
| `canAccess(bytes32 tokenId, address requester, bytes32 resourceType) view returns (bool)` | external view, free | **New (D13).** The same checks as `checkAccess`, with no event. Screens use it to decide what to show |
| `checkAccess(bytes32 tokenId, address requester, bytes32 resourceType) returns (bool)` | external, `msg.sender` MUST equal `requester` | A transaction. True only if the token is valid, includes the resource type, and the requester's role matches. Maps `roleId` back to the `Role` enum by name (errata 3). Emits AccessGranted or AccessDenied through AuditLog and never reverts on a denial, so the denial is logged. Clients read the outcome from the receipt's event. Requiring `msg.sender == requester` stops one party logging denials in another's name |
| `getConsent(bytes32 tokenId) view returns (ConsentToken)` | external view | AF-4 |
| `tokenIdsOf(address patient) view returns (bytes32[])` | external view | Serves the patient's token list |
| `tokenIdsOfGrantee(address grantee) view returns (bytes32[])` | external view | Serves Shared with me and the Hospital B viewer |
| `setProofVerifier(address gateway)` | admin only | Sets the proof hook to the ProofGateway address. The field name is misleading, because it holds the gateway, not the generated verifier |

Only a registered patient account issues. Only the issuing patient revokes.

### RecordRegistry.sol (Anush; DAT-2, DAT-8, AF-13)

Without this contract nothing on-chain created, stored, or listed a `RecordRef`, although DAT-2 requires the hash on-chain and three screens (My records, Shared with me, Hospital B viewer) need to find a patient's records. A consent token scopes by resource type, so listing by type is exactly the lookup the retrieval path needs. The encrypted emergency capsule registers here too, under its own resource type.

| Function | Access | Rules |
| --- | --- | --- |
| `addRecord(bytes32 patientId, bytes32 resourceType, string cid, bytes32 contentHash, address creator) returns (bytes32 entryId)` | only `accountOf(patientId) == msg.sender` | `cid` is 1 to 100 bytes. There is no revoked flag and no delete. `creator` is the signer (decision D9), and readers verify the signature, so a false attribution fails on read. Logs RecordAdded |
| `recordsOf(bytes32 patientId, bytes32 resourceType) view returns (RecordRef[])` | external view | In the order added |
| `getRecord(bytes32 entryId) view returns (RecordRef)` | external view |  |

**Privacy note for the threat model (SEC-1).** The registry makes metadata public: which record types a patient has, how many, when each was added, and who created it. Resource types are hashes of a small public vocabulary, so hashing does not hide them. A per-patient salt would, but it would have to be shared with every grantee, so it is out of scope. The CIDs are readable by anyone too, which matters for the revoked-grantee limit under Known limits.

### ProofVerifier.sol (generated by snarkjs) and ProofGateway.sol (Punith; ZKP-2 to ZKP-4, ZKP-7)

The proof is bound to an issued claim three ways. The commitment in the proof must equal the one an issuer registered for that patient and claim type. The threshold must be the one the verifier requires. And the nonce must be the one derived for this request, and each nonce works once. The generated verifier holds no state. The gateway holds the state.

```solidity
struct ClaimEntry { uint256 commitment; address issuer; }   // commitment = Poseidon(value, salt)
mapping(bytes32 => mapping(bytes32 => ClaimEntry)) claims;   // patientId, claimType
mapping(uint256 => bool) usedNonce;
```

| Function | Access | Rules |
| --- | --- | --- |
| `setCommitment(bytes32 patientId, bytes32 claimType, uint256 commitment, bytes32 tokenId)` | Doctor or Lab role, plus an active Claim token | The token must be unexpired and unrevoked, belong to the patient's account (`accountOf(patientId)`), name the caller, and include the Claim type. The commitment is non-zero and below the BN254 field modulus. Only the original issuer may replace a commitment. Emits `ClaimCommitted`, logs ClaimIssued with the Claim token's id |
| `revokeClaim(bytes32 patientId, bytes32 claimType)` | the patient's account or the issuer | Deletes the claim. Emits `ClaimRevoked`, logs ClaimRevoked with the claimType as refId |
| `getClaim(bytes32 patientId, bytes32 claimType) view returns (uint256 commitment, address issuer)` | external view |  |
| `expectedNonce(bytes32 requestId) view returns (uint256)` | external view | `keccak256(abi.encode(chainid, address(this), requestId))` reduced modulo the BN254 scalar field |
| `verifyEligibility(bytes32 patientId, bytes32 claimType, bytes32 requestId, uint256 requiredThreshold, uint256[8] proof, uint256[3] publicSignals) returns (bool)` | external, anyone | Checks in order: a claim exists, `publicSignals[0]` equals `requiredThreshold`, `publicSignals[1]` equals the stored commitment, `publicSignals[2]` equals `expectedNonce(requestId)`, the nonce is unused, and the generated verifier accepts. On success the nonce is marked used and ProofVerified is logged. An invalid proof returns false and does not use up the nonce |
| `verifyBatch(bytes32[] patientIds, bytes32[] claimTypes, bytes32[] requestIds, uint256[] requiredThresholds, uint256[8][] proofs, uint256[3][] publicSignals) returns (bool[])` | external, anyone | The same checks for each proof inside one transaction. It saves transactions, not per-proof cost, because nothing is aggregated |

**Public signal order.** `publicSignals` is `[threshold, commitment, nonce]`. This was confirmed by reading the proof output after a real compile: the order is the declaration order of the public inputs in the template, not the order in the `public [...]` clause of the main component.

A Groth16 proof is 8 field elements: a\[2\], b\[4\], c\[2\]. Produce the call arguments with `snarkjs exportSolidityCallData`, which orders b's coordinates the way the generated verifier expects. The gateway rebuilds a, b, and c from the flat array. Each successful call emits `AuditLogged(..., ProofVerified, ...)`, so AUD-1 holds without a separate logging call. The gateway takes `requiredThreshold` so a prover cannot choose its own threshold, and the nonce uses the gateway's own address as the verifier address. Both readings are accepted.

One limit worth knowing: a proof is not bound to the caller, so someone who sees a pending proof could submit it first and use up its nonce. For the local demo this is acceptable, and the audit event names the caller.

### AuditLog.sol (Punith)

Design choice: a single shared contract that every other contract calls internally, rather than each contract emitting its own independent event type. This gives one canonical, chronologically ordered feed and makes AUD-1 and AUD-2 trivial to test. If the internal call reverts, the whole transaction reverts, so a contract cannot succeed without logging.

| Function | Access | Rules |
| --- | --- | --- |
| `log(bytes32 patientId, address actor, ActionType action, bytes32 refId)` | external, registered emitter contracts only | Emits `AuditLogged`. The calling contract passes the real actor, so the log names the person, not a relayer or a contract. Emitters are trusted to pass it honestly. The log has no other storage |
| `registerEmitter(address contractAddr)` | external, admin only | Allow-lists which contracts may call `log`. The deploy script registers every logging contract, including RecordRegistry and ProofGateway |

### RecoveryVault.sol (Rohith, Yagna)

The vault holds the share commitments and is the source of truth for every request (errata 2): `patientId`, requester, release mode, cancelled, and Tier 2 timing, for break-glass and lost-key requests alike.

```solidity
enum TrusteeCategory { Family, HospitalCustodian, GuardianService, InstitutionalCustodian }
enum ReleaseMode { None, CapsuleOnly, FullRecord }   // errata 11: None = 0. Tests and UI use names, never numbers

struct GuardianSet {                                  // by patientId
    uint8 threshold;                                  // k
    uint32 epoch;                                     // +1 on every rotateShareCommitments
    uint64 updatedAt;                                 // errata 10: drives last refreshed and Refresh due
    ShareCommitment[] commitments;
}

struct RecoveryRequest {                              // by requestId
    bytes32 patientId;
    address requester;
    ReleaseMode mode;
    bytes32 clientKeyHash;   // sha256 of the one-time ML-KEM public key, non-zero in FullRecord
    bool completed;
    bool cancelled;
    uint64 tier2EndsAt;      // 0 until custodianCoSign
}
// address breakGlassCapsule (set once); uint256 requestCounter (errata 6); uint64 immutable TIER2_DELAY
```

| Function | Access | Rules |
| --- | --- | --- |
| `registerShareCommitments(ShareCommitment[] commitments, uint8 threshold)` | the patient's account, once per patient | Enforces every rule under Guardian set rules below. Stores the set at epoch 1 |
| `rotateShareCommitments(ShareCommitment[] fresh, uint8 threshold)` | the patient's account | Replaces the set, increments `epoch`, sets `updatedAt`. Same rules as registration |
| `getCommitments(bytes32 patientId) view returns (ShareCommitment[])` | external view | A free call the coordinator uses before collecting |
| `getGuardianSet(bytes32 patientId) view returns (uint8 threshold, uint32 epoch, uint64 updatedAt)` | external view | Serves Refresh due and last refreshed (errata 10) |
| `setBreakGlassCapsule(address)` | admin only, callable once | Reverts if already set. The capsule is deployed after the vault, because its constructor takes the vault address, so the deploy script calls this once both exist (errata 16) |
| `setReleaseMode(bytes32 requestId, bytes32 patientId, address requester, ReleaseMode mode)` | external, `onlyBreakGlassCapsule` | **Not internal.** An internal function cannot be called from another contract (errata 1). Writes the request row, so the vault holds patientId, requester, mode, cancelled, and tier2EndsAt for every request (errata 2) |
| `custodianCoSign(bytes32 patientId, bytes32 requestId)` | the registered custodian for that patient | `requestId` must belong to `patientId`. Sets `tier2EndsAt = now + TIER2_DELAY`. Logs TierEscalated with the custodian as actor |
| `cancelRecovery(bytes32 requestId)` | the patient's account or a registered guardian | Logs RecoveryCancelled. A cancelled request can no longer be completed |
| `startRecovery(bytes32 patientId, bytes32 clientKeyHash) returns (bytes32 requestId)` | the patient's account only | Rejects a zero `clientKeyHash`. Opens a FullRecord request. Logs RecoveryInitiated |
| `completeRecovery(bytes32 requestId)` | the requester only, once, not after a cancellation | Requires mode FullRecord, so a break-glass responder cannot log RecoveryCompleted (errata 17). Logs RecoveryCompleted |
| `getRequestStatus(bytes32 requestId) view returns (ReleaseMode, bool cancelled, uint64 tier2EndsAt)` | external view | Used by the countdowns |
| `getRecovery(bytes32 requestId) view returns (bytes32 patientId, address requester, ReleaseMode mode, bytes32 clientKeyHash, bool completed, bool cancelled, uint64 tier2EndsAt)` | external view | The coordinator's full-mode preconditions |

**Guardian set rules.** Both registration and rotation MUST revert with a clear reason unless all of these hold:

- `2 <= threshold <= commitments.length`, which is `2 <= k <= n+1` (D2).
- Indexes are unique and contiguous from 1 to n+1, exactly one `InstitutionalCustodian` row sits at index n+1, and trustee addresses are unique.
- **Trustee mix (T1-5).** Not every guardian row (indexes 1 to n) has the same category. The custodian row is excluded from this check. T1-5 is accepted, so this is a revert, not a warning (errata 12). The UI warns first.
- **Category cap (D14, v1.9).** No `TrusteeCategory` appears on k or more rows, custodian row included. The first rule only catches a set where all guardians match. A set of 3 family members and 1 friend at k = 3 would pass it, yet the three family members could rebuild S. The cap closes that, and costs one counting loop over at most n+1 rows. A hospital guardian and the institutional custodian are separate categories in the enum. If they are the same hospital in practice, the UI SHOULD say so.

No function parameter or event ever carries a raw share, salt, or key. `startRecovery` and the lost-key flow are covered in Recovery and break-glass technical flow.

### BreakGlassCapsule.sol (Yagna)

The capsule contract keeps only the window. Everything else about a request lives in the vault (errata 2).

```solidity
struct Capsule {                  // by patientId
    bool preAuthorized;
    uint32 version;               // bumped on each preAuthorizeCapsule
    uint64 preAuthorizedAt;
    bytes32[] fields;             // keccak256 of the field names: bloodType, allergies, currentMedications, emergencyContacts
}
struct BGRequest { uint64 windowExpiry; bool closed; }       // by requestId
enum RequestStatus { None, WindowOpen, Tier2Waiting, Cancelled, Closed }
struct RequestInfo {
    bytes32 patientId; address requester; ReleaseMode releaseMode;
    RequestStatus status; uint64 windowExpiry; uint64 tier2EndsAt;
}
// uint256 requestCounter (errata 6); uint64 immutable WINDOW_DURATION
```

| Function | Access | Rules |
| --- | --- | --- |
| `preAuthorizeCapsule(bytes32[] fields)` | the patient's account | Resolves the patient through `PatientIdentity.patientIdOf(msg.sender)`. Stores field hashes and bumps `version` |
| `triggerEmergencyAccess(bytes32 patientId) returns (bytes32 requestId)` | `EmergencyResponder` role (RoleRegistry) | Role check, then capsule check, then `setReleaseMode`, then window, then log, all in one transaction |
| `revokeCapsule()` | the patient's account | Clears `preAuthorized` |
| `capsuleStatus(bytes32 patientId) view returns (bool preAuthorized, uint32 version)` | external view | AF-4 |
| `closeWindow(bytes32 requestId)` | anyone, only after expiry, only once | Logs BreakGlassExpired. Succeeds for a released, a cancelled, and a failed request alike |
| `getRequest(bytes32 requestId) view returns (RequestInfo)` | external view | Reads the vault's fields and combines them with `windowExpiry` and `closed` into one status |

```solidity
// The atomicity argument for AUD-2: if AuditLog.log(...) reverts, the entire
// triggerEmergencyAccess transaction reverts, so windowExpiry is never set without the event.
// setReleaseMode runs before the window is stored. The vault is a trusted contract wired at
// deploy time, so this order is acceptable. Move the window write first if that ever changes.
function triggerEmergencyAccess(bytes32 patientId) external returns (bytes32 requestId) {
    require(roleRegistry.hasRole(msg.sender, Role.EmergencyResponder), "not authorized");
    require(capsule[patientId].preAuthorized, "no capsule");
    requestId = _newRequestId();   // keccak256(abi.encode(block.chainid, address(this), msg.sender, requestCounter++))
    recoveryVault.setReleaseMode(requestId, patientId, msg.sender, ReleaseMode.CapsuleOnly);
    requests[requestId].windowExpiry = uint64(block.timestamp) + WINDOW_DURATION;
    auditLog.log(patientId, msg.sender, ActionType.BreakGlassTriggered, requestId);
}

// AF-5: nothing happens on-chain when a timestamp passes, so closing needs a call.
function closeWindow(bytes32 requestId) external {
    BGRequest storage r = requests[requestId];
    require(r.windowExpiry != 0, "unknown request");   // errata 19: requester no longer lives here
    require(!r.closed, "already closed");
    require(block.timestamp >= r.windowExpiry, "window still open");
    r.closed = true;
    (bytes32 patientId, , , , , , ) = recoveryVault.getRecovery(requestId);
    auditLog.log(patientId, msg.sender, ActionType.BreakGlassExpired, requestId);
}
```

A lost-key `requestId` has no entry here, so `closeWindow` reverts it as unknown. When `getRequest` combines a cancelled request that was then closed, `Cancelled` wins over `Closed`, so the screen keeps showing why it ended.

The `closeWindow` timing and the request-status logic were run on a local Hardhat network with a 180-second window: closing early reverts, closing after expiry succeeds for any caller and logs once, and a second close reverts. That run used the v1.8 skeleton, so repeat it against the errata-2 version.

### Contract changes (D19 to D23)

| Contract | Change | Why |
| --- | --- | --- |
| ProofGateway | New event `EligibilityVerified(bytes32 indexed patientId, bytes32 claimType, bytes32 requestId, uint256 requiredThreshold, address indexed verifier)`, emitted by `verifyEligibility` and by each successful proof in `verifyBatch`, next to the ProofVerified audit entry. Both functions revert when `requiredThreshold` is 256 or more | The functions change state, so a client cannot read their boolean. The event carries the verified fact and who asked for it (D19, D23) |
| RecoveryVault | `custodianCoSign(patientId, requestId)` reverts unless the caller is the registered custodian for that patient, the request exists and belongs to `patientId`, its mode is not None, it is not cancelled or completed, and `tier2EndsAt == 0`. A second call cannot reset the delay | Closes the reset and late-call gaps (D21) |
| BreakGlassCapsule | New view `capsuleFields(patientId) returns (bytes32[] fieldHashes, uint32 version)` with the pre-authorized field hashes and the version | The coordinator releases exactly these fields (D20) |
| Circuit | `Num2Bits(8)` range checks on `value` and `threshold`, beside the comparator | Makes the 8-bit limit a constraint, not comparator behavior. About 16 more constraints, so recompile (D23) |

A request whose window already closed can still be co-signed on-chain, because the vault cannot see the capsule contract's `closed` flag. That is harmless, since the coordinator never releases after `windowExpiry`, so the delay cannot finish (D21).

## Cryptographic and zero-knowledge specification

One root secret S derives every key a party owns, ML-DSA-65 signs, ML-KEM-768 transports, and Groth16 proves. This section pins the exact parameters, so all four implementations compute the same values.

| Purpose | Algorithm and parameter set | Sizes | Library | PRD link |
| --- | --- | --- | --- | --- |
| Record encryption | AES-256-GCM | 256-bit key, 96-bit IV, 128-bit tag | `node:crypto` | DAT-2 |
| Record and event signatures | ML-DSA-65 (FIPS 204, the standardized successor to Dilithium3) | public key 1,952 B, signature 3,309 B | `@noble/post-quantum/ml-dsa.js` | DAT-4, D9 |
| Share transport and type-key wrapping | ML-KEM-768 (FIPS 203) | public key 1,184 B, ciphertext 1,088 B | `@noble/post-quantum/ml-kem.js` | T1-4, DAT-5 |
| Share commitment | SHA-256 over the share string's UTF-8 bytes, the salt, and the domain tag | 32-byte digest, salt of at least 16 random bytes | `node:crypto` | T1-2, T1-7, D4 |
| Secret splitting | Shamir's Secret Sharing, k-of-(n+1) over GF(2^8), k at least 2 | share strings of 99 characters for a 32-byte secret | secrets.js-grempe | T1-2, T1-8, D2 |
| Zero-knowledge | Groth16 on BN254 | proof of 8 field elements: a\[2\], b\[4\], c\[2\]; 3 public signals | circom and snarkjs | ZKP-1 to ZKP-4 |
| Claim commitment | Poseidon(value, salt) over the BN254 scalar field | one field element; the salt is 31 random bytes | circomlib in the circuit, circomlibjs off-chain | ZKP-2, ZKP-7 |
| Identity, type, record, and capsule keys | HKDF-SHA256 from the 32-byte record secret S | 32-byte keys, a 32-byte seed for ML-DSA, a 64-byte seed for ML-KEM | `node:crypto` `hkdfSync`, or Web Crypto HKDF | PAT-5, DAT-5 |
| Device keystore | PBKDF2-HMAC-SHA256 (at least 600,000 iterations), then AES-256-GCM | 32-byte key, 16-byte salt, 12-byte IV | Web Crypto or `node:crypto` | AF-3 |
| Signed account actions | EIP-712 typed data, secp256k1 owner key, ECDSA | n/a | ethers, OpenZeppelin EIP712 | PAT-3 |
| Randomness | CSPRNG only | n/a | `crypto.randomBytes` in Node, `crypto.getRandomValues` in the browser | T1-1 (reworded) |

Key, salt, nonce, and share generation MUST NEVER use `Math.random` anywhere. Anything else is a hygiene failure.

### ML-DSA and ML-KEM parameter choice

The module plans named a post-quantum library without a parameter set. This TRD specifies ML-DSA-65 and ML-KEM-768, both NIST security category 3, as the default for signatures and key encapsulation. It is a mid-tier choice: stronger than the category-1 options (ML-DSA-44, ML-KEM-512) without the larger keys and slower operations of category 5 (ML-DSA-87, ML-KEM-1024). The team can move to category 5 with a one-line change, since `@noble/post-quantum` exposes all three sizes under the same call shape. The snippet below was run against `@noble/post-quantum` 0.7.1. The sign and verify argument order and the sizes in the table matched. The v1.9 review repeated this on Node 22: ML-KEM key generation needs a 64-byte seed and rejects a 32-byte one, and key generation from the same seed is deterministic, which the identity design depends on.

```ts
import { ml_kem768 } from '@noble/post-quantum/ml-kem.js';
import { ml_dsa65 } from '@noble/post-quantum/ml-dsa.js';
import { randomBytes } from '@noble/post-quantum/utils.js';

// Vault creation (PAT-5, PAT-6): S is random, the key pairs are derived from it.
// hkdf(ikm, info, length) is HKDF-SHA256: node:crypto hkdfSync, or Web Crypto in the browser.
const S = randomBytes(32);
const dsaKeys = ml_dsa65.keygen(hkdf(S, 'medichain-dsa-v1', 32));
const kemKeys = ml_kem768.keygen(hkdf(S, 'medichain-kem-v1', 64));

// Sign a record on write (DAT-4)
const signature = ml_dsa65.sign(recordCiphertext, dsaKeys.secretKey);
const isValid = ml_dsa65.verify(signature, recordCiphertext, dsaKeys.publicKey);

// Wrap a share to a trustee, or a type key to a grantee (T1-4, DAT-5)
const { cipherText, sharedSecret } = ml_kem768.encapsulate(kemKeys.publicKey);
const recovered = ml_kem768.decapsulate(cipherText, kemKeys.secretKey);
// use sharedSecret / recovered as an AES-256-GCM key to wrap the actual payload
```

### Commitment format *(specified here)*

PRD decision D4 fixed where commitments and salts live. This pins the exact input to the hash, so all four implementations compute the same value. The domain tag is a fixed ASCII string, so a commitment cannot be reinterpreted for a different purpose if the scheme is ever reused.

```
commitment = SHA256( utf8(shareString) || saltBytes || utf8('medichain-share-v1') )
```

`shareString` is the exact string `secrets.share()` returns (errata 4). Never hex-decode it. These strings have odd length, and decoding drops the last nibble, so a commitment computed that way would still match the wrong share.

### Key hierarchy

One secret, S, sits at the root. The client creates it with a CSPRNG when the vault is created (SCR-02), and everything else derives from it with HKDF-SHA256.

```
dsaSeed    = HKDF(S, 'medichain-dsa-v1', 32)   -> ml_dsa65.keygen(dsaSeed)   (public key on-chain)
kemSeed    = HKDF(S, 'medichain-kem-v1', 64)   -> ml_kem768.keygen(kemSeed)   (public key on-chain)
typeKey    = HKDF(S, 'medichain-type-v1:' + resourceTypeName)   // the name, e.g. 'Observation', never its hash (errata 5)
recordKey  = HKDF(typeKey, 'medichain-record-v1:' + recordId)
capsuleKey = HKDF(S, 'medichain-capsule-v1')
```

`recordId` is the FHIR resource id carried in the envelope. It must be known before upload, when no CID or registry id exists yet.

- **Sharing.** Sharing is by record type, so the grantee gets one wrapped `typeKey` per type in the token (DAT-5). From it they derive the key for any record of that type, including ones added later. They cannot derive S, another type's key, or `capsuleKey`.
- **Recovery restores the whole identity.** Rebuilding S re-derives the same key pairs, so the published public keys still match, old signatures still verify, and old wrapped keys still unwrap. No `updateKeys` call is needed.
- **Cost, for the threat model.** Any k colluding trustees can forge the patient's signatures, not just read records. Keys also cannot rotate without a new S, which means re-encrypting every record and re-dealing. Deriving `capsuleKey` from S does not separate emergency access from full access, because anyone who rebuilds S can derive both.
- **Other parties (DAT-9).** A clinician, lab, or second-institution endpoint derives its signing and ML-KEM key pairs the same way from its own secret, with the same two info strings, and registers the public keys. It holds no type keys of its own: it receives them wrapped.
- **T1-1 reworded.** The patient's client generates S at vault creation with a CSPRNG and hands it to the dealer at guardian setup, wrapped to the dealer's one-time key. The dealer still runs in a minimal, short-lived context, makes the polynomial coefficients and salts with the CSPRNG, and overwrites its buffers after distribution as far as the runtime allows (T1-3, best effort).

Run in Node on `@noble/post-quantum` 0.7.1: all derived keys were distinct, a grantee holding one type's key read a record added later under that type but not a record of another type, the same public keys came back after rebuilding S from shares, and an old signature and an old wrapped key still worked.

### PQC provider abstraction

All ML-DSA and ML-KEM calls go through one small interface, not directly against `@noble/post-quantum`, so the backend can change without touching the key-wrapping or share-encryption calls. This mitigates the library-maturity risk.

```ts
interface PQCProvider {
  dsaKeygen(seed?: Uint8Array): { publicKey: Uint8Array; secretKey: Uint8Array };
  sign(message: Uint8Array, secretKey: Uint8Array): Uint8Array;
  verify(sig: Uint8Array, message: Uint8Array, publicKey: Uint8Array): boolean;
  kemKeygen(seed?: Uint8Array): { publicKey: Uint8Array; secretKey: Uint8Array };
  encapsulate(publicKey: Uint8Array): { cipherText: Uint8Array; sharedSecret: Uint8Array };
  decapsulate(cipherText: Uint8Array, secretKey: Uint8Array): Uint8Array;
}
```

The default implementation is `NoblePQCProvider`, wrapping `@noble/post-quantum`. A provider backed by a WASM binding can be added later against the same interface.

### Trusted setup for the proof system

Groth16 needs a circuit-specific trusted setup. Use a public Powers of Tau phase-1 file where one is available, or generate one locally for the prototype, and run the circuit-specific phase-2 contribution locally. For a circuit of 531 constraints, a Powers of Tau of power 12 (2^12 = 4,096 constraints) is enough. v1.8 wrote this one place as "size 2.12" and another as "power 12". The meaning is power 12. A local setup with a single contributor is not production-grade. This matches the PRD risk register and is item 3 in Known limits.

### Zero-knowledge circuit: a claim an issuer committed to (ZKP-2, ZKP-7)

The first version of this circuit took a private age and a public threshold, so anyone could prove "over 18" with any number they typed, and a valid proof could be replayed. The circuit now proves knowledge of a value and salt that match a commitment an issuer registered, and ties each proof to one request.

| Signal | Visibility | Meaning |
| --- | --- | --- |
| value | private | The issued claim value, for example an age. Must fit in 8 bits |
| salt | private | Blinds the commitment. 31 random bytes |
| threshold | public | What the verifier requires. Must fit in 8 bits |
| commitment | public | Must equal Poseidon(value, salt) and the commitment the issuer registered on-chain |
| nonce | public | Derived from the chain id, the gateway address, and the request or token id. Makes the proof single-use |

```circom
pragma circom 2.1.0;
include "circomlib/circuits/comparators.circom";
include "circomlib/circuits/poseidon.circom";

template ClaimEligibility() {
    signal input value;       // private: the issued claim value
    signal input salt;        // private: blinds the commitment
    signal input threshold;   // public: what the verifier requires
    signal input commitment;  // public: must equal the commitment the issuer registered
    signal input nonce;       // public: binds the proof to one request, chain, and gateway

    component h = Poseidon(2);
    h.inputs[0] <== value;
    h.inputs[1] <== salt;
    h.out === commitment;

    component gte = GreaterEqThan(8);   // value and threshold must fit in 8 bits
    gte.in[0] <== value;
    gte.in[1] <== threshold;
    gte.out === 1;

    signal nonceSq;                     // puts the nonce into a constraint so the compiler keeps it
    nonceSq <== nonce * nonce;
}
component main {public [threshold, commitment, nonce]} = ClaimEligibility();
```

The circuit has no output signal. A valid proof means every constraint held, so the value meets the threshold. `GreaterEqThan` is a standard circomlib comparator, and its inputs must fit in 8 bits, which is stated as a limit. The circuit also range-checks value and threshold with Num2Bits(8), which adds about 16 constraints. Recompile to get the new count, because the 531 figure is without them. The gateway rejects a required threshold of 256 or more.

**Size.** Compiled with circom 2.2.3: 531 constraints in total (253 non-linear and 278 linear), 3 public inputs, 2 private inputs, and no outputs. Poseidon accounts for most of the non-linear constraints, which fits the roughly 250 expected. The sub-second proof-generation figure from the PRD stays a target for Punith's benchmark (ZKP-5), not a result. Confirm the count again with `circom --inspect` after any change to the circuit.

### Issuance and verification flow (ZKP-7)

1. **Issuance.** The patient first issues a Claim permission that names the issuer. An account holding the Doctor or Lab role then computes Poseidon(value, salt) with circomlibjs and calls `ProofGateway.setCommitment(patientId, claimType, commitment, tokenId)` with that token. The issuer's address is stored and emitted, and ClaimIssued is logged. There is no issuer screen: issuing, and the patient's permission in the demo, are scripted. The issuer puts the value and salt into the encrypted Patient record for the patient (see Issued claim in Data schemas).
2. **Proving.** For a request, the patient's client reads the value and salt from the decrypted Patient record, computes the commitment and the nonce, builds the witness, and generates the proof.
3. **Verification.** The verifier, or the patient on its behalf, calls `verifyEligibility` or `verifyBatch` with the threshold the verifier requires and the request or token id. The gateway checks the claim, threshold, commitment, and nonce before it checks the proof.

```ts
nonce = BigInt(keccak256(abi.encode(chainId, gatewayAddress, requestId))) % BN254_SCALAR_FIELD
```

### Pipeline

1. Compile: `circom ClaimEligibility.circom --r1cs --wasm --sym -l node_modules`, so circomlib resolves from `node_modules`.
2. Trusted setup: Powers of Tau power 12 (2^12 constraints), enough for the 531-constraint circuit, and a circuit-specific phase 2.
3. Witness generation from the private value and salt and the public threshold, commitment, and nonce, using the compiled WASM. The salt must be below the field prime, which 31 random bytes guarantees.
4. Proof generation with snarkjs Groth16, producing an 8-element proof and the public signals.
5. Convert with `exportSolidityCallData`, then verify on-chain through `ProofGateway.verifyEligibility` (ZKP-3).

### Batching (ZKP-4)

`ProofGateway.verifyBatch` runs the single-proof checks inside one transaction, including the commitment, threshold, and nonce check for each proof. This reduces the number of transactions, not the per-proof verification cost. There is no proof aggregation or recursion. v1.8 described this as "proof batching with rollup-style aggregation", which overstated it, and the PRD now says batching with no aggregation.

### Limits to state

- The claim is as of issuance. A real system would commit to a date of birth and prove against the current date.
- The verifier trusts the issuer. The issuer's address is stored and emitted, so a verifier can see who vouched.
- The trusted setup is local and has a single contributor.
- The value and the threshold must each fit in 8 bits.
- A proof is not bound to its submitter. Someone who sees a pending proof could submit it first and use up its nonce.

### Stretch circuit (ZKP-6)

A diagnostic-threshold claim is another claim type issued the same way, and it can reuse this circuit for values that fit in 8 bits. A wider range, such as lab values above 255, would need a wider comparator. Build it only after the circuit and its on-chain integration work end to end.

## Module interface contracts and local services

These are the concrete shapes behind the PRD's "interfaces to agree early" table. Freeze them before the Extension phase. Changing a shape after another module has built against it is the most common source of late integration bugs in a four-person build. Rows marked v1.9 are new.

| Interface | Concrete shape | Produced by | Consumed by |
| --- | --- | --- | --- |
| ABAC role IDs | `Role` enum in RoleRegistry.sol | Anush | Yagna (BG-1), Punith |
| Consent token | `ConsentToken` struct, see Data schemas | Anush | Punith, Rohith, Yagna |
| Public-key lookup | `PatientIdentity.getDilithiumKey` and `getMLKEMKey` | Anush | Rohith (DAT-4, DAT-5) |
| Account resolution (v1.9) | `PatientAccountFactory.accountOfOwner`, `predictAccount`, and the DID rule in `register` | Anush | Yagna (routing), Rohith (seed script) |
| Access-check pair (v1.9) | `canAccess` (view) and `checkAccess` (logged transaction) | Anush | Rohith (read path), Yagna (screens), Hospital B viewer |
| Audit event | `AuditLogged` event, see Data schemas | Punith | Every contract, and the audit-trail view (UI-5) |
| Record envelope | JSON shape in Data schemas | Rohith | Anush (stores the hash), Yagna (renders in the UI) |
| Share commitment and bundle | `ShareCommitment` struct and bundle JSON, see Data schemas | Rohith | Yagna (Tier 2, BG-6) |
| Verified-share result | Boolean returned by the RecoveryCoordinator's local commitment check (recovery flow, step 5) | Rohith | Yagna |
| Record registry | `addRecord`, `recordsOf`, `getRecord` | Anush | Rohith (client library), Yagna (SCR-04, SCR-12), Hospital B viewer |
| Consent and request reads | `getConsent`, `tokenIdsOf`, `tokenIdsOfGrantee`, `getRequest`, `getRequestStatus`, `getRecovery`, `getGuardianSet`, `capsuleStatus` | Anush, Yagna, Rohith | Frontend (Yagna) |
| Coordinator API | Local HTTP and event stream, see below | Rohith | Yagna (responder and recovery screens), GuardianScript |
| IPFS proxy API (v1.9) | `POST /ipfs/pin` and `GET /ipfs/:cid`, see below | Rohith | Client library, seed script |
| Device keystore file | JSON shape in Data schemas | Rohith | Yagna (unlock, backup, restore) |
| Claim commitment and proof signals | `setCommitment` (with the Claim token id), `revokeClaim`, `getClaim`, `expectedNonce`, `verifyEligibility`, `verifyBatch`; signals `[threshold, commitment, nonce]` | Punith | Anush (consent hook), Yagna (SCR-05, SCR-12), Rohith (seed script, issuance) |
| Demo identity fixture | Git-ignored, demo-only file written by the seed script | Rohith | Yagna (unlock step on the clinician and Hospital B screens) |
| Signed account action | EIP-712 Execute message, see PatientAccount.sol | Anush | Yagna (signing prompt), Relayer |
| Capsule schema | JSON shape in Data schemas | Yagna | Rohith (policy-before-crypto ordering, T1-9), Anush (role check) |

### Rules shared by the three local services (v1.9, D17)

The dealer launcher and coordinator share one process on port 4100. The IPFS proxy is a separate process on port 4101, so the service that sees ciphertext never shares memory with the one that briefly holds S.

- Each binds to 127.0.0.1 and rejects any other source address.
- Each generates a session bearer token at startup and hands it to the frontend through local config. Every call without it is rejected with 401, except the proxy's read-only GET /ipfs/:cid, which serves ciphertext only. The coordinator adds a per-request token, described below (D22).
- CORS allows only the frontend's origin, `http://localhost:5173`. The Hospital B viewer on port 5174 may call only the IPFS proxy's read-only GET /ipfs/:cid route and no other service (D18).
- Browsers MUST NOT use `EventSource` for the event stream, because it cannot send an `Authorization` header. Use `fetch` with a streamed response body, or a small fetch-based event-source library, and send the token in the header. A token in a query string would end up in logs.

### Local dealer launcher (AF-16)

A browser tab cannot start a process, so the local Node service exposes one small endpoint that launches the one-shot dealer.

| Call | Purpose |
| --- | --- |
| `POST /dealer/run` with `{ n, k, trustees: [{ index, category, kemPublicKey }] }` | Starts the one-shot dealer process and returns `{ sessionId, dealerKemPublicKey }`. The dealer made that key just now. The trustee list includes the custodian at index n+1 |
| `POST /dealer/:sessionId/secret` with S wrapped to `dealerKemPublicKey` | S travels wrapped, never in plaintext, even on loopback. The dealer unwraps it, splits it, and returns the n+1 encrypted bundles and their commitments. Then it overwrites what it can and exits |

S flows from the client to the dealer, not the other way round, because S already exists at vault creation. The client adds each trustee's address and category to the returned commitments to build the `ShareCommitment[]`. Re-sharing uses the same two calls, with S taken from the keystore.

### RecoveryCoordinator local API (AF-7)

The coordinator runs as a local Node process. The UI and the scripted guardians talk to it over HTTP on 127.0.0.1:4100 only, and it never listens on a public interface. Each request gets a random bearer token when it is created, and every other call must present it.

| Call | Purpose |
| --- | --- |
| `POST /requests` with `{ patientId, mode, requestId }`, plus `clientKemPublicKey` in full mode | Starts collection. `mode` is `capsule` or `full`. In full mode the client's one-time ML-KEM public key is required, and the coordinator refuses a malformed one or one whose hash differs from the `clientKeyHash` stored on-chain. Returns the coordinator's own one-time ML-KEM public key and the request token |
| `POST /requests/:id/shares` with `{ trusteeIndex, kemCipherText, iv, ciphertext, authTag }` | A trustee's share and salt, re-wrapped to the coordinator's one-time key. Used by GuardianScript. Anything not wrapped to that key is rejected |
| `GET /requests/:id/events` | Server-Sent Events stream of progress, shaped as below |
| `GET /requests/:id/result` | Capsule mode: the capsule fields. Full mode: S wrapped to the client's one-time key, returned once, after which the coordinator wipes S. Returns 409 until released, and 410 once the request has closed |
| `POST /requests/:id/close` | Wipes the one-time key and anything released. The coordinator also does this itself at `windowExpiry` |

```ts
type CoordinatorEvent = {
  requestId: string;
  phase: 'collecting' | 'tier2-wait' | 'reconstructing' | 'released' | 'failed' | 'cancelled' | 'closed';
  needed: number;       // k
  validCount: number;   // shares that passed the commitment check
  trustees: { index: number; category: string; status: 'waiting' | 'responded' | 'invalid' | 'timeout' }[];
  windowExpiry: number; // unix seconds. Break-glass: mirrors the chain. Full mode: the off-chain FULL_MODE_TTL deadline (errata 18)
  tier2EndsAt?: number; // unix seconds, mirrors the chain
};
```

**Full mode (lost-key recovery).** Before it wraps anything, the coordinator reads `RecoveryVault.getRecovery(requestId)` and requires mode FullRecord, a requester that is the patient's account, a request that is neither cancelled nor completed, and that `sha256` of the one-time key it was given equals the stored `clientKeyHash`. It then follows the same share-collection, commitment-check, and Tier 2 steps as break-glass. It wraps S to the client's key and wipes it, and it never touches a record or a ciphertext. A full-mode request has no on-chain window, so `windowExpiry` is the coordinator's own deadline, `FULL_MODE_TTL` (300 s in the demo) after the request starts, and the coordinator wipes its keys then (errata 18). The coordinator MUST refuse to release after `windowExpiry` (see Timing parameters) and MUST drop the one-time key and any released data when the request closes. The invalid status is what the screen shows as "couldn't verify". All deadlines read chain time (D11).

### IPFS proxy API (v1.9, D12)

The browser cannot safely hold a Pinata credential, and a Node service that handled keys would break the rule that no key reaches a service. This small proxy holds only the credentials and sees only ciphertext envelopes.

| Call | Purpose |
| --- | --- |
| `POST /ipfs/pin` with a `RecordEnvelope` JSON body | Validates that the body is a well-formed envelope (schema version, ciphertext, iv, authTag, signature) and under the configured size cap, pins it through Pinata with `PINATA_JWT`, writes a copy to a local cache directory keyed by CID, and returns `{ cid }`. It rejects anything that is not an envelope, so a client mistake cannot upload plaintext |
| `GET /ipfs/:cid` | Returns the envelope from the local cache, or fetches it through `PINATA_GATEWAY` and caches it. The cache is the offline fallback the PRD risk table asks for |

POST /ipfs/pin needs the session token and the frontend origin. GET /ipfs/:cid needs no token and accepts the frontend and the Hospital B origin (port 5174), because Hospital B and the coordinator must read ciphertext (D18, D20). Credentials come from the proxy's environment only. The frontend bundle MUST NOT contain `PINATA_JWT`, and a test scans the build for it.

### Authorized retrieval path (DAT-3)

1. The grantee's client calls `ConsentRegistry.canAccess(tokenId, requester, resourceType)`, a free view call, to decide whether to continue. To leave the audit trail it then sends `checkAccess(tokenId, requester, resourceType)` as a transaction from a funded signer and reads AccessGranted or AccessDenied from the receipt. A denial stops the flow and is audit-logged (v1.9, D13).
2. It calls `RecordRegistry.recordsOf(patientId, resourceType)` to list the records. The registry is the only place that knows the CIDs.
3. For each record it fetches the envelope through the IPFS proxy using the `cid` string, and checks `sha256(ciphertext)` against `contentHash` before doing anything else.
4. It looks up the creator's public key with `PatientIdentity.getDilithiumKey(creator)` and verifies the ML-DSA signature (DAT-4).
5. It reads the `KeysWrapped` event for the token, unwraps the `typeKey` for this resource type with its own ML-KEM secret key, derives `recordKey = HKDF(typeKey, 'medichain-record-v1:' + envelope.recordId)`, and decrypts with AES-256-GCM. A failed authentication tag aborts with no partial output.
6. It returns the FHIR JSON to the caller.

### Proof request, capsule mode, and tokens (D18 to D22)

**Proof request (D19).** A verifier, such as an insurer or pharmacy, makes a request and gives it to the patient as a link or code. In the demo the seed script does this step. The patient's client reads it on SCR-05 and builds the proof for it. The verifier then submits the proof with `verifyEligibility` and reads `EligibilityVerified` from the receipt. SCR-12 shows "Verified: patient is over 18" from that event's claim type and threshold. A proof share is not a consent token, so no token or resource type is involved.

```json
{ "claimType": "AgeOver", "requiredThreshold": 18, "requestId": "0x...", "verifier": "0x..." }
```

**Capsule mode (D20).** The coordinator follows these steps for a break-glass request:

1. Authenticate. `POST /requests/challenge` with `{ requestId }` returns a challenge. `POST /requests` then carries `requesterSignature`, a wallet signature over that challenge, plus the session token. The coordinator checks the signature against `getRecovery(requestId).requester`. If that requester is a `PatientAccount`, the signer must be the account's owner.
2. Check the chain. `getRecovery` must show mode CapsuleOnly, not cancelled, not completed, and `BreakGlassCapsule.getRequest` must show WindowOpen or Tier2Waiting.
3. Fetch the capsule. Read `RecordRegistry.recordsOf(patientId, EmergencyCapsule)`, take the newest entry, and fetch its envelope with `GET /ipfs/:cid`. Check `sha256(ciphertext)` against `contentHash`, and verify the signature against the creator's published key. The creator must be the patient's own key.
4. Release only what was authorized. After reconstruction, decrypt with `capsuleKey`, require the payload `version` to equal the on-chain version, and keep only the fields whose `keccak256(name)` appears in `capsuleFields(patientId)`. A capsule that was saved but not pre-authorized has no fields, so nothing is released.
5. Scope. In this mode the coordinator decrypts the emergency capsule only. It never decrypts any other patient record.

Saving emergency info (SCR-08) registers the encrypted capsule with `addRecord` first and calls `preAuthorizeCapsule` second. A failure after the first step leaves a capsule that nobody can release.

**Tokens (D22).** Three things authorize a coordinator call:

| Token | Created | Authorizes |
| --- | --- | --- |
| Session token | At service startup, handed to the frontend and the scripts | `POST /requests/challenge`, `POST /requests`, the dealer calls, and `POST /ipfs/pin` |
| Request token | Returned by `POST /requests` | `GET /requests/:id/events`, `GET /requests/:id/result`, `POST /requests/:id/close`, and `POST /requests/:id/shares` |
| One-time key wrap | The request's coordinator key | A share must also be wrapped to it, or the coordinator rejects it. The guardian script gets the request token with the coordinator's one-time public key |

## Frontend technical requirements

The frontend is a React 18 single-page app plus a second small app for Hospital B. Keys live only in the browser, and every service call carries a token. Rows marked v1.9 are new or changed.

| Area | Specification |
| --- | --- |
| Stack | React 18, Vite, ethers.js ^6 against an injected wallet provider (`window.ethereum`) or a local Hardhat provider in tests |
| Components | PatientDashboard (consent list, guardian setup, capsule editor, audit view: UI-1, UI-2, UI-3, UI-5), RecordsList (My records: UI-8), ResponderView (role check, trigger break-glass, countdown: UI-4), ClinicianView (Shared with me, reuses the responder components, opens after an unlock or a demo identity: UI-9, AF-1, AF-19), HospitalBViewer (a second small Vite app on its own port that calls the same contracts and retrieval path with a demo identity: INT-1, INT-3, AF-2), GuardianScript (a Node CLI and test harness driving scripted responses, not a UI component: UI-7, decision D5) |
| Connect and routing (SCR-01, v1.9, D10) | Check the wallet and the chain id (31337). Resolve `PatientAccountFactory.accountOfOwner(wallet)`. If an account exists and `getDilithiumKey(account)` is non-empty, route to the patient dashboard or the unlock screen. If no account exists, offer to create one: the relayer calls `createAccount(wallet)` and pays the gas. For other wallets, look up `RoleRegistry` and `getDilithiumKey(wallet)` to route to the clinician, lab, or responder view. A wallet with both a patient account and a role gets a Continue as choice |
| State and keystore (AF-3) | The record secret S lives only in the device keystore (see Data schemas): an encrypted IndexedDB record, unlocked by passphrase, with an optional downloadable backup file and a restore-from-file option. Both key pairs are derived from S when the keystore is unlocked and are held in React state for the open tab only. Anything else persisted locally is non-secret: an address book of names for trustees and saved people, and UI preferences |
| Keystores for other parties (DAT-9, AF-19) | A clinician, lab, or second-institution endpoint unlocks its own keystore, which holds its own secret. In the demo it loads a demo identity from the fixture file the seed script wrote, and signs with Hardhat's unlocked account for that address through a JSON-RPC signer (v1.9, D13). The clinician screen (SCR-12) and the Hospital B viewer (SCR-13) show an unlock or "demo identity loaded" step before anything else. The same limit applies: keys cannot rotate without a new secret |
| Vault creation | At SCR-02 the account already exists (see routing). The client creates S with `crypto.getRandomValues`, derives the signing and ML-KEM key pairs from it, publishes both public keys through `PatientIdentity.register` from the patient's account, and writes S to the keystore. The DID hash is built from the account address (PAT-8). Nothing is rotated later, because the keys are fixed by S |
| Where crypto runs (v1.9, D12) | Encryption, signing, key wrapping, and decryption run in the client library, in the browser for the patient's and clinician's own keys and in Node for the seed script. Proof generation runs client-side with snarkjs's browser and WASM build, using the claim value and salt from the decrypted Patient record. Envelopes go to IPFS only through the IPFS proxy, which sees ciphertext. The RecoveryDealer MUST run as a short-lived, one-shot Node process, never inside the browser tab (T1-1 reworded, T1-3). The frontend starts it through the local dealer launcher. Hand-off: the dealer makes a one-time ML-KEM key pair, the client wraps S to its public key, and the dealer splits S and returns the n+1 bundles and commitments, then overwrites what it can and exits |
| Sponsored actions (AF-11) | Patient writes go through `PatientAccount.executeSigned`. The app shows a plain-language prompt, the patient signs an EIP-712 Execute message, and the relayer submits it and pays gas. If the sponsored path is dropped, the app falls back to direct wallet transactions, and the wallet's own pop-up may show gas |
| Access checks (v1.9, D13) | Screens use the free `canAccess` view to decide what to show. Opening a record sends the logged `checkAccess` transaction from a funded signer and reads the outcome from the receipt |
| Patient identification (AF-8) | The responder screen scans the emergency card payload (see Data schemas) where the camera allows, with manual entry as a fallback |
| Responder confirmation (AF-9) | The "patient cannot give consent" checkbox is UI-only. When ticked, `attestation: true` goes into the signed off-chain audit record. The contract does not enforce it |
| Guardian setup rules (AF-12) | The UI recommends k no greater than n, although the contract allows up to n+1. It warns before sending when every guardian shares a category, or when any category would hold k or more trustees (T1-5, D14), because the contract will reject both. Trustee display names live in the local address book. After a device change the UI shows group and a short address instead |
| Countdowns (v1.9, D11) | A countdown is `windowExpiry` or `tier2EndsAt` minus the latest block's timestamp, polled once a second. With interval mining that value advances without transactions. Do not subtract from `Date.now()` alone |
| Event stream (v1.9, D17) | The responder and recovery screens read the coordinator's stream with a fetch-based reader that sends the bearer token in a header, and the coordinator allows only this app's origin |
| Environment config | `.env` holding the RPC URL (default `http://localhost:8545`, chain ID 31337), deployed contract addresses, the relayer account key, and the demo timing values. Pinata credentials belong only to the IPFS proxy's environment, never to a frontend build. Never commit any of these |
| Plain-language rule | UI-6: gas amounts, raw keys, and hashes are never surfaced to the patient. The dashboard speaks in terms of "share with", "stop sharing", and "emergency info", not token IDs |

## Recovery and break-glass technical flow

Shares are checked off-chain against on-chain commitments, policy runs before any cryptography, and every request ends in a closed window. Break-glass and lost-key recovery share one guardian and quorum machinery and differ in the trigger, the requester, and what is released.

### Design decision: off-chain verification, on-chain commitment reference *(specified here)*

The PRD describes checking shares against their commitments but does not say where. This matters: if a raw share were ever submitted as a transaction or calldata to a public chain, it would become permanently public, which defeats secret sharing entirely. So commitments are the only thing that ever touches the chain. The shares and their salts are checked and combined entirely off-chain, in the RecoveryCoordinator process. `RecoveryVault.getCommitments` is a free view call precisely because a hiding commitment leaks nothing.

One further hop needs protection. During recovery, each trustee decrypts their own ML-KEM bundle locally, then must transmit the raw share to wherever reconstruction happens. That transmission MUST itself be re-wrapped to the coordinator's ephemeral ML-KEM public key for that one hop. A raw share is never sent over an unencrypted channel, even internally.

### State machine, break-glass path

| From | Event | To |
| --- | --- | --- |
| Idle | Responder calls `triggerEmergencyAccess` | PolicyCheck |
| PolicyCheck | Role or capsule check fails | Rejected: the call reverts, no window opens, no cryptography runs |
| PolicyCheck | Both checks pass | WindowOpen, then Tier1Collecting |
| Tier1Collecting | k shares verified | Reconstructed, then Released (capsule fields only) |
| Tier1Collecting | Exactly k-1 verified shares at `COLLECTION_TIMEOUT` | Tier2CoSignRequested, then DelayCountdown |
| Tier1Collecting | Fewer than k-1 verified shares at `COLLECTION_TIMEOUT` | Failed: no summary is released |
| DelayCountdown | Patient or a registered guardian calls `cancelRecovery` | Cancelled |
| DelayCountdown | Delay elapses uncancelled | CustodianShareReleased, then Reconstructed, then Released |
| Released, Failed, Cancelled, or WindowOpen | Expiry reached, then anyone calls `closeWindow` | WindowClosed (BreakGlassExpired is logged) |
| DelayCountdown | The k-th valid guardian share arrives before the delay ends | Reconstructed, then Released. The custodian's share is not used and the delay is dropped (D23) |

Every request that opens a window ends in WindowClosed, whether it was released, cancelled, or failed.

### Sequence, break-glass path (BG-8 demo)

1. The responder calls `BreakGlassCapsule.triggerEmergencyAccess(patientId)`. The role check and the capsule pre-authorization check run first. Either failing reverts the whole call, so no state changes and no cryptography runs (T1-9, BG-5, enforced by Solidity's atomic revert, not by convention).
2. On success, in one transaction, the contract makes the requestId (errata 6), calls `RecoveryVault.setReleaseMode(requestId, patientId, requester, CapsuleOnly)` (decision D1, T1-10, errata 1 and 2), sets `windowExpiry`, and emits `AuditLogged(..., BreakGlassTriggered, ...)` (AUD-2). If the audit call reverts, everything reverts.
3. Off-chain, the RecoveryCoordinator receives `POST /requests` in capsule mode and calls `RecoveryVault.getCommitments(patientId)`, a free view call.
4. Guardians respond (scripted in the demo, per decision D5). Each guardian's client decrypts its own ML-KEM bundle locally, then re-wraps the `{share, salt}` to the coordinator's ephemeral ML-KEM public key for the one hop.
5. The coordinator checks each received share: `SHA256(utf8(shareString) || salt || utf8('medichain-share-v1')) == commitment`. Mismatches are rejected before any reconstruction attempt (T1-7).
6. Once k valid shares are held, the coordinator runs Lagrange interpolation off-chain to recover S (T1-8).
7. The coordinator reads the release mode set in step 2. For a break-glass request it derives `capsuleKey`, decrypts, and returns only the pre-authorized capsule fields. A full record is never assembled on this path (BG-4, decision D1).
8. If exactly k-1 guardian shares arrive by `COLLECTION_TIMEOUT`, the coordinator reports `tier2-wait`, and the registered custodian calls `RecoveryVault.custodianCoSign(patientId, requestId)`, which starts the Tier 2 delay. In the demo the scripted custodian sends it (SCR-S2). The patient or any registered guardian can call `cancelRecovery` during the delay (BG-7). After the delay elapses uncancelled, the custodian's own bundle is released through the identical re-wrap and verify path (BG-6), and reconstruction proceeds with k-1 guardian shares plus the one custodian share. A co-sign later than `windowExpiry` minus `TIER2_DELAY` cannot finish in time: the coordinator never releases after `windowExpiry`, so that request fails.
9. When the countdown reaches the window's expiry, the responder's client or the coordinator calls `BreakGlassCapsule.closeWindow(requestId)`. It is permissionless and only succeeds after expiry. It emits BreakGlassExpired and marks the request closed (AF-5). Without this call nothing would log the closing, since a timestamp passing is not a transaction. The coordinator wipes its one-time key and any released capsule at the same moment. A request that never reaches k valid shares ends the same way: no summary is released, and `closeWindow` logs the closing.
10. If the patient or a guardian cancels during the Tier 2 delay, `cancelRecovery` emits RecoveryCancelled (AF-6), and `getRequest` reports Cancelled. The window still closes at expiry as above.

### Sequence, patient lost-key path (full mode)

1. The patient connects their wallet on the new device, and the app finds their account. The client makes a one-time ML-KEM key pair. The patient's account calls `RecoveryVault.startRecovery(patientId, clientKeyHash)` with the sha256 of that public key, which opens a FullRecord request and emits RecoveryInitiated. The patient's wallet is assumed to be intact (AF-10).
2. The client calls `POST /requests` with the mode, the request id, and the one-time public key.
3. The coordinator reads `getRecovery` and refuses unless the request is FullRecord, was opened by the patient's account, is neither cancelled nor completed, and the hash of the key it was given equals the stored `clientKeyHash`.
4. Steps 3 to 6 of the break-glass sequence run unchanged: fetch the commitments, collect the guardians' responses through the one-hop re-wrap, check each against its commitment, and rebuild S with Lagrange interpolation. The Tier 2 fallback and its cancel right apply as before.
5. The coordinator wraps S to the client's one-time key, serves it once through `GET /requests/:id/result`, and wipes S. It never decrypts a record.
6. The client unwraps S, asks the patient for a new passphrase, and writes a new keystore holding S. It re-derives both key pairs, which come out identical to the published ones, and decrypts records locally.
7. The client calls `completeRecovery(requestId)`, which requires mode FullRecord (errata 17) and emits RecoveryCompleted. The client then offers to refresh the guardians' shares (T1-6).
8. A full-mode request has no on-chain window, so the coordinator enforces `FULL_MODE_TTL` (300 s in the demo), wipes its keys then, and mirrors that deadline in the event's `windowExpiry` (errata 18).

Result: the patient's signing and encryption keys are back, so old signatures verify, old wrapped keys unwrap, and the patient can sign and share again.

### Binding the client's one-time key to the request

The coordinator checks the request on-chain, and it also has to know that the one-time key it is handed belongs to the patient who opened that request. The key is therefore bound to the request.

- **Change.** `startRecovery(bytes32 patientId, bytes32 clientKeyHash)` stores `sha256(clientKemPublicKey)`, and `getRecovery` returns it. The coordinator refuses to wrap S unless the key it was given equals the stored one. A zero hash is rejected.
- **Why it works.** The transaction is signed through the patient's account, so the key is bound to the one party that can start the request. A substituted key does not match. On loopback the risk was already low, since an attacker would already be running code on the machine, but the binding answers how the coordinator knows the key is the patient's.
- **Cost.** One extra storage slot per request and one extra argument.
- **Run, on a skeleton:** 4 tests passed on a RecoveryVault skeleton. A stranger cannot start a request, and a zero hash is rejected. `getRecovery` returns the stored hash. The coordinator-side check accepts the patient's key and refuses a substituted one. Only the requester can complete or cancel, once, and neither can follow the other.

### Proactive re-sharing (T1-6)

Re-sharing keeps the secret S unchanged but replaces every share, so a slow attacker who has collected some old shares never reaches k valid ones of one generation.

1. The patient starts it from their own client, prompted by a Refresh due reminder. A fresh one-shot RecoveryDealer run receives S from the keystore, wrapped to the dealer's one-time key as in Setup, and builds a new random polynomial, which produces new shares and new commitments with new salts.
2. Each new share bundle is encrypted to its trustee's ML-KEM key exactly as in Setup (T1-4), and the dealer overwrites what it can and exits (T1-3).
3. The patient calls `RecoveryVault.rotateShareCommitments`, which stores the new commitments, sets `updatedAt`, and increments the patient's `epoch`. A share from an earlier epoch no longer matches any current commitment, so the coordinator's check in step 5 rejects it.
4. Trustees discard their old bundles. This is a courtesy, not a security dependency, because the on-chain epoch change is what actually invalidates them.

Scope limit: this design re-deals from S, so the dealer briefly holds S again. A distributed refresh that never reconstructs S is a different protocol and is out of scope. An attacker who already holds k shares of one generation can still rebuild S offline. Re-sharing only stops shares from different generations combining. The cadence (for example every few months, or after any recovery event) is a policy value the team sets, and the contract does not enforce it. `getGuardianSet(patientId)` returns `updatedAt`, so the app can show Refresh due. There is no scheduler.

### Timing parameters

| Parameter | Where it lives | Demo value | Why this value |
| --- | --- | --- | --- |
| `WINDOW_DURATION` | BreakGlassCapsule constructor argument | 180 s | The window starts at the trigger, so it must cover the whole request including the Tier 2 safety wait, and still leave time to read the summary. Short enough to watch it expire live |
| `TIER2_DELAY` | RecoveryVault constructor argument | 60 s | Long enough for the patient to find and press Cancel during the demo, short enough to let it run out |
| `COLLECTION_TIMEOUT` | RecoveryCoordinator configuration, off-chain | 20 s | How long the coordinator waits for guardians before checking whether Tier 2 applies. Not a contract constant |
| `FULL_MODE_TTL` | RecoveryCoordinator configuration, off-chain | 300 s | Deadline for a lost-key request, which has no on-chain window. It covers the 20 s collection and the 60 s Tier 2 delay (errata 18) |

All values are in seconds. The contract values are constructor arguments and the others are configuration, so changing them means redeploying on the local network, which takes seconds. **How the three fit together:**

- The window starts at the trigger and also covers the Tier 2 delay. The coordinator MUST NOT release the capsule after `windowExpiry`.
- Tier 1 path: scripted guardians respond within seconds, so the summary is on screen with about 2.5 minutes left.
- Tier 2 path: 20 s collection timeout, then the 60 s safety wait, then a few seconds to rebuild the secret. About 90 s is used, which leaves about 90 s to read the summary.
- Rule: `WINDOW_DURATION` MUST be at least twice `TIER2_DELAY`. The deploy script asserts this (180 is at least 120). If the team lengthens the delay, lengthen the window with it.
- **One clock (v1.9, D11).** All of these deadlines, on-chain and off-chain, use chain time. The demo node mines a block every second, the coordinator reads the latest block's timestamp for `COLLECTION_TIMEOUT` and `FULL_MODE_TTL`, and the demo never advances time. v1.8 allowed jumping the clock to skip a wait, but a jump shifts chain time away from the coordinator's wall clock, so countdowns and deadlines would disagree. Tests that need to skip time run on a separate in-process network, not on the demo node.

No production values are specified. Real values would need clinical input, and clinical safety validation is a PRD non-goal, so the report should present these as demo settings only.

### Flow changes (D20, D21, D23)

- **A late k-th share.** If the k-th valid guardian share arrives while the request is in the Tier 2 delay, the coordinator rebuilds S at once and releases. The custodian's share is not used and the delay is dropped. The patient's cancel right still applies until the release.
- **Capsule mode authorization.** The sequence above runs after the checks in Proof request, capsule mode, and tokens: the requester's signature, the on-chain mode and window, and the capsule fetch and filter.
- **Tier 2 guards.** `custodianCoSign` has the guards in Contract changes (D19 to D23). A co-sign that comes too late to finish before `windowExpiry` simply fails, because the coordinator never releases after it.
- **Known limit 20.** The delay is not cryptographic. The custodian can decrypt its own bundle at any time.

## Known limits

Twenty limits go in the report. This is the same list as Limitations to state in the report in PRD v1.6, item for item. Items 6, 8, and 12 are extended in v1.9, and items 19 and 20 are new.

1. **An unmaintained secret-sharing library.** secrets.js-grempe has had no release since September 2019. It is small and passed an end-to-end test of the recovery flow, which is acceptable for a prototype.
2. **Synthetic data.** All evaluation uses Synthea-generated data, so results may not reflect real record shapes or volumes. MIMIC-III is a stretch only.
3. **Local trusted setup.** The Groth16 setup is circuit-specific, done locally, and has a single contributor, so it is not production-grade.
4. **Colluding trustees.** Any k colluding trustees can read everything and forge the patient's signatures, because the keys derive from S. Keys cannot rotate without a new S, which means re-encrypting every record and re-dealing the shares.
5. **Capsule-only release is enforced by the flow.** It is not enforced by separate keys: the emergency quorum rebuilds S. A separate capsule secret, dealt independently of S, would fix this and is a stretch.
6. **A revoked grantee keeps type keys.** Revocation cannot recall keys already delivered, so a revoked grantee can still derive keys for records of the same types added later, and the CIDs are public. An emergency summary already on a screen cannot be unseen either. Consent is therefore time-limited by the contracts, not by the keys. (Extended in v1.9.)
7. **Record metadata is public.** The record registry shows which record types a patient has, how many, when each was added, and who created it.
8. **The coordinator briefly holds S in both modes.** It needs S to derive the capsule key in break-glass mode and to wrap it for the new device in lost-key mode, and it wipes S straight after. Overwriting memory is best effort in JavaScript, because strings are immutable and the garbage collector may keep copies, so ending the process is the real mitigation. This applies to the dealer too. (Extended in v1.9.)
9. **The relayer is trusted for liveness.** It can delay or drop a signed action but cannot forge one.
10. **The attestation is UI-only.** The responder's confirmation that the patient cannot give consent is stored in the signed audit record. The contract does not enforce it.
11. **Wallet recovery is out of scope.** Lost-key recovery restores records and keys, not control of a lost wallet or account.
12. **Tier 2 needs k-1 guardians.** The custodian holds a single share, so the fallback only helps when exactly one guardian share is missing, that is, when k-1 guardians respond. With k = 2, the custodian plus one guardian is enough to rebuild S. (Extended in v1.9.)
13. **Post-quantum scope is signatures and key transport only.** Record and event signatures and share and key encapsulation use post-quantum algorithms. The proof system and Ethereum transaction signatures stay classical.
14. **Claim values are as of issuance, and the issuer is trusted.** A proof shows the value an issuer committed, not the current date. The verifier trusts the issuer, and claim values and thresholds must fit in 8 bits.
15. **Gas figures are estimates.** They come from skeleton contracts, not the final ones, and must not be quoted as results.
16. **Commitments detect tampering, not inconsistent dealing.** Hash commitments do not prove the dealer dealt consistently, unlike verifiable secret sharing. The patient's own device is the dealer, which is why this is acceptable.
17. **A lost passphrase with no backup file** goes through guardian recovery.
18. **Records authored by someone other than the patient are simulated by the seed script.** A production route would have the creator encrypt to the patient's ML-KEM key and sign a hash of the plaintext inside the payload, and the patient's client would re-encrypt under the record key and register it.
19. **No out-of-band notification.** The patient sees a break-glass or Tier 2 request on the dashboard or the Activity screen only after opening the app, and guardians have no screen, so an unconscious patient cannot cancel in practice. (New in v1.9.)
20. **The Tier 2 delay is not cryptographic.** The custodian can decrypt its own bundle at any time, so the delay holds only because the contract records when it ends and the coordinator honors that time. A compromised custodian that colludes with k-1 guardians can rebuild S without waiting. The delay gives an honest patient a visible window to cancel. (New in the second pass.)

## Non-functional technical requirements

These are planning estimates for gas, latency, storage, and browser support. The ZKP-5 benchmark is the authority once the harness runs, and none of the figures below may be quoted as measured results in the report.

### Gas budget

| Operation | Rough range | Basis |
| --- | --- | --- |
| Consent issue, with wrapped keys for one or two types | roughly 250,000 to 350,000 gas | Measured on a skeleton without token storage: about 125,000 gas for the calldata and the `KeysWrapped` event with two 1.2 KB wrapped keys. Storing the token adds more on top, so treat the total as an estimate |
| Consent revoke | 30,000 to 50,000 gas | One storage update plus an audit event |
| Record registration (`addRecord`) | about 250,000 to 285,000 gas | Measured on the skeleton: 283,091 for a first record with a 46-character CID and 249,047 for the next with a 59-character CID |
| Role assignment | 30,000 to 50,000 gas | Single mapping write |
| Groth16 verification of an issued-claim proof (one proof, with the commitment, threshold, and nonce checks, a nonce write, and an audit event) | about 270,000 gas | Measured on a skeleton: 268,683. Treat as an estimate for the final contract |
| Batch of N proofs | about 513,000 gas for two proofs, roughly 257,000 each | Measured on a skeleton: 513,275 for two. No aggregation is implemented (ZKP-4), so the saving is fewer transactions and one transaction's fixed overhead, not cheaper verification per proof |
| Break-glass trigger plus audit event | about 120,000 to 170,000 gas | A new vault request row, the window slot, the counter, three external calls, and the audit event. Estimate, not measured (D23) |
| Share-commitment registration (Setup, once per patient) | roughly (n+1) x 45,000 gas | Two storage slots per commitment after the v1.9 field reorder. The v1.8 field order needed about three slots, so roughly (n+1) x 66,000. Estimate, not measured |
| Identity registration (v1.9) | roughly 2.3 million gas | Estimate, not measured: the two public keys total about 3.1 KB, which is around 100 storage slots. It fits Hardhat's default block gas limit but is the most expensive single call in the demo |
| Account creation (v1.9, D10) | to be measured | A `CREATE2` deployment of a small contract. The relayer pays it once per patient |

Treat every figure as a planning estimate. ZKP-5 is the authority once the benchmark harness runs.

### Latency

Proof generation for the issued-claim circuit should be well under one second on ordinary developer hardware. That is a target for Punith's benchmark (ZKP-5), not a result. The circuit has 531 constraints (see circuit specification). The break-glass path itself should add no processing time beyond the steps in the sequence above. The only deliberate delay is Tier 2's, and it must be clearly distinguishable in any timing measurement from an accidental one. Automine stays on, so transactions still confirm immediately. Interval mining only adds empty blocks between them, so it does not change these timings.

### Storage and size

An AES-GCM-encrypted Observation resource is typically a few kilobytes, plus about 4.4 KB for the base64 ML-DSA signature in the envelope. Confirm the current Pinata plan's file-count and size limits before the demo rather than assuming a number, since these are account-specific and change over time. The Synthea seed script SHOULD load a small subset of resources, and the IPFS proxy's local cache covers the offline case.

### Client and browser

A current Chromium-based browser with an injected wallet (MetaMask or equivalent) is required for the frontend demo, matching the account-abstraction and consent-signing flow.

## Testing requirements

Thirty-six test groups cover the contracts, the services, and the end-to-end flows. Rows in v1.8 that targeted one component in several places are merged here, and rows marked v1.9 are new. The tests restate the PRD's acceptance tests against the exact signatures in this document.

| Component | Test type | Tool | Key cases |
| --- | --- | --- | --- |
| Contracts (all) | Unit and integration | Hardhat, Chai, Mocha | Every PRD acceptance test in the PRD's Testing and acceptance section, restated against the exact function signatures in this document |
| PatientIdentity (v1.9) | Unit | Hardhat | `register` reverts for a DID hash not derived from the caller's own address. A second registration of the same address or patientId reverts. `accountOf` and `patientIdOf` agree. `getDilithiumKey` returns empty bytes for an unregistered address |
| PatientAccountFactory (v1.9) | Unit | Hardhat | Two owners get two accounts. The deployed address equals `predictAccount`. `accountOfOwner` returns it. A second `createAccount` for the same owner returns the existing account. Anyone can call it, and the owner is the constructor parameter |
| PatientAccount | Unit | Hardhat and ethers `signTypedData` | The relayer pays gas and the owner's balance is unchanged. The target sees the account as `msg.sender`. Replay, wrong signer, and expired deadline all revert. Direct `execute` is owner-only |
| RoleRegistry | Unit | Hardhat | The emergency-responder role is assignable before ConsentRegistry's full ABAC wiring exists, so BG-1 can be built and tested independently in Core build |
| ConsentRegistry | Unit | Hardhat | Expired, revoked, and unauthorized requests revert on issue and fail on check. A valid request passes and returns the correct wrapped-key reference. `issueConsent` reverts when the wrapped-key count differs from the resource-type count, and emits `KeysWrapped` readable by tokenId. `getConsent`, `tokenIdsOf`, and `tokenIdsOfGrantee` return what was issued, and the revoked flag appears after `revokeConsent` |
| Access-check pair (v1.9) | Unit | Hardhat | `canAccess` is a view call and writes nothing. `checkAccess` emits AccessGranted or AccessDenied with the requester as actor, does not revert on a denial, and reverts when `msg.sender` differs from `requester`. The outcome is readable from the receipt |
| RecordRegistry | Unit | Hardhat | Only the patient's account can add. `recordsOf` lists by type in order and returns an empty list for an unused type. CIDv0 and CIDv1 strings round-trip. The capsule registers under its own type. An empty or over-100-byte CID reverts. RecordAdded logs the real actor |
| ProofGateway and the circuit | Unit and circuit | Hardhat, snarkjs | A valid proof of an issued claim verifies on-chain, and a tampered proof returns false without using up the nonce. A replayed nonce reverts. `setCommitment` reverts without the Doctor or Lab role, and without an active Claim token that names the caller, belongs to the patient, and covers the Claim type (an expired, revoked, other-patient, other-issuer, or other-type token is refused). Only the original issuer can replace a commitment. The patient or the issuer can revoke it and a stranger cannot. A proof against a revoked claim reverts, and a new issuer can then register one. ClaimIssued and ClaimRevoked are logged with the real actor, and the issuer is stored and emitted. A proof against another patient's commitment, another threshold, or another request's nonce is rejected. A batch checks a nonce per proof. A wrong value, a wrong salt, or a value outside 8 bits cannot produce a proof at all |
| AuditLog | Unit | Hardhat | No contract can complete a logged action if the internal log call is made to revert, which proves AUD-2's atomicity argument and not just its intent. The logged actor is the one passed in, not the calling contract. An unregistered caller reverts |
| RecoveryVault: setup rules | Unit | Hardhat | Threshold enforced at 2 <= k <= n+1 on registration and rotation. Indexes unique and contiguous, one custodian row at n+1, unique trustee addresses. A set in which every guardian shares a category reverts. A set in which any category holds k or more trustees reverts (v1.9, D14). A mixed set passes. `rotateShareCommitments` increments the epoch and sets `updatedAt`. `getGuardianSet` reports them. Commitments readable through a free view call |
| RecoveryVault: requests | Unit | Hardhat | `setReleaseMode` reverts unless called by the capsule contract, and `setBreakGlassCapsule` reverts the second time. `custodianCoSign` is callable only by the registered custodian, only for a `requestId` that belongs to the `patientId`. `cancelRecovery` blocks release during the delay, emits RecoveryCancelled, and shows as cancelled in `getRequestStatus`. A stranger cannot call `startRecovery`. The patient's call opens a FullRecord request and RecoveryInitiated names the real actor. Only the requester can `completeRecovery`, once, and not after a cancellation, and not for a CapsuleOnly request (errata 17). `getRecovery` reports patient, requester, mode, key hash, completed, and cancelled |
| BreakGlassCapsule | Unit | Hardhat | A caller without the emergency role cannot open a window. The policy check runs and can revert before any other state changes, verified by checking no other contract's state changed on a reverted call. `closeWindow` reverts before expiry, succeeds for any caller after expiry, logs once, and reverts a second time. A lost-key `requestId` reverts as unknown. `getRequest` reports WindowOpen, Tier2Waiting, Cancelled, and Closed correctly. `capsuleStatus` follows `preAuthorizeCapsule` and `revokeCapsule`. `preAuthorizeCapsule` reverts for a caller with no registered identity. Use `evm_increaseTime` for the 180-second window on the in-process test network only (v1.9, D11) |
| Key binding | Unit and integration | Hardhat, Jest | `startRecovery` rejects a zero key hash and a non-patient caller and stores the hash. The coordinator wraps S only when `sha256(clientKemPublicKey)` equals the stored hash, and refuses a substituted key |
| Client data library | Unit | Jest | Round-trip encrypt and decrypt with a known key. A tampered ciphertext fails GCM authentication. A record signed by the wrong key fails `ml_dsa65.verify` |
| FHIR shape (v1.9) | Unit | Jest | Every seeded resource has an id matching the FHIR id pattern. A party's DID appears as an `identifier`, never in an id. The issued-claim extension is a valid complex extension. If INT-2 is done, the official validator passes |
| IPFS proxy (v1.9) | Integration | Jest | Rejects a body that is not a well-formed envelope or is over the size cap. Returns the same envelope from the cache when Pinata is unreachable. A scan of the built frontend finds no Pinata credential |
| RecoveryDealer and RecoveryCoordinator | Unit and property | Jest | Every combination of at least k of the n+1 shares reconstructs the same S. Every combination of k-1 does not. A share with a mismatched salt is rejected before reconstruction is attempted. A share string run through hex decoding fails its commitment, which catches the odd-length mistake |
| PQCProvider mock | Unit, with a mock backend | Jest | Contract-level and service tests can run against a mock PQCProvider so they do not depend on the real ML-DSA and ML-KEM speed |
| Device keystore | Unit | Jest | Round trip works, a wrong passphrase fails the GCM tag, a backup file restores, the iteration count is at least 600,000, and the file holds one secret only (S for a patient, the party's own secret otherwise) |
| Keys for other parties | Unit | Jest | A clinician's or lab's keystore re-derives the public keys registered for it. A record it signs verifies against that key, and a type key wrapped to its ML-KEM key unwraps with the derived secret key |
| Key hierarchy | Unit | Jest | Re-deriving from the same S gives the same public keys. An old signature still verifies and an old wrapped key still unwraps after S is rebuilt from shares. Type, record, and capsule keys all differ from each other and from S. A wrapped type key reads later records of its own type and fails on another type |
| Seed script | Integration | Jest | Creates the demo identities on unlocked Hardhat accounts and registers their public keys. The scripted patient shares a Claim permission with Dr. Rao, and Dr. Rao issues the demo claim through `setCommitment` with that token. The fixture file is git-ignored, labelled demo-only, and holds no private keys |
| Local services (v1.9) | Integration | Jest | The dealer launcher, coordinator, and IPFS proxy reject non-loopback addresses, calls without the session token, and requests from another origin. The browser stream reader sends the token in a header |
| Local dealer launcher | Integration | Jest | S is accepted only when wrapped to the dealer's one-time key. The shares recombine to S, and the process exits afterwards |
| RecoveryCoordinator: capsule mode | Integration | Jest | Shares not wrapped to the one-time key are rejected. Events arrive in order. `/result` returns 409 before release and 410 after close. Nothing is released after `windowExpiry` |
| RecoveryCoordinator: full mode | Integration | Jest | Refuses unless `getRecovery` shows mode FullRecord, the patient's account as requester, and no cancellation or completion. Rejects a malformed one-time key, and a key whose hash differs from the stored hash. Returns S wrapped to the client's key once, then wipes it. Never reads a ciphertext. Wipes its keys at `FULL_MODE_TTL` |
| One clock (v1.9) | Integration | Hardhat node, Jest | On the demo node's settings, `block.timestamp` advances with no transactions. The on-screen countdown equals `windowExpiry` minus the latest block time. The coordinator's collection timeout fires on chain seconds. No demo script calls a time-advance method |
| Recovery on a new device | End to end | Jest | After recovery with a fresh keystore: the derived public keys equal the published ones, an old record decrypts, an old signature verifies, a new signature verifies against the original key, and the patient can wrap a type key to a grantee |
| Routing (v1.9) | Integration | Jest, browser test | A registered patient's wallet resolves to its account and the dashboard. A fresh wallet is offered account creation. A doctor's wallet routes to the clinician view |
| Build | Smoke | Hardhat | `npx hardhat compile` succeeds with `evmVersion: 'cancun'` and OpenZeppelin 5.x. A Jest test imports `@noble/post-quantum` to prove the ESM setup works |
| End to end | Scripted integration | Hardhat script and Jest | The full break-glass sequence in this document, and the two-hospital interoperability flow (INT-1, INT-3), each as one runnable script. The deploy script must refuse a configuration where `WINDOW_DURATION` is less than twice `TIER2_DELAY` |
| ProofGateway: request and result (v1.9, D19, D23) | Unit | Hardhat | A valid proof emits EligibilityVerified with the claim type, threshold, request id, and verifier. A required threshold of 256 or more reverts. A value or threshold outside 8 bits cannot produce a proof, so the range checks hold in the circuit. |
| Capsule mode (v1.9, D20) | Integration | Jest | The coordinator refuses unless getRecovery shows CapsuleOnly, getRequest shows an open window, and the requester's signature matches the on-chain requester. It reads only the capsule through GET /ipfs/:cid, verifies hash and signature, and releases only fields in capsuleFields. A saved but unauthorized capsule releases nothing. |
| Tier 2 guards (v1.9, D21, D23) | Unit and integration | Hardhat, Jest | custodianCoSign reverts for an unknown, cancelled, completed, wrong-patient, or already co-signed request, and for any caller but the custodian. A k-th valid guardian share during the delay rebuilds S at once. |
| Tokens and Hospital B (v1.9, D18, D22) | Integration | Jest | Events, result, close, and shares need the request token, and creation needs the session token. A share not wrapped to the one-time key is rejected. Hospital B's origin can GET /ipfs/:cid and cannot pin. |

## Environment, build, and deployment

Everything runs on one developer machine against Hardhat's built-in network, started by one command. Rows marked v1.9 are new or changed.

| Item | Specification |
| --- | --- |
| Developer machine | Core i5 10th generation or better, 8 GB or more RAM, 20 GB SSD (per the PRD's reproducibility target) |
| Runtime | Node.js 20.19 or later, npm or pnpm |
| Network | Hardhat's built-in local network only. Default chain ID 31337, RPC at `http://localhost:8545`. The demo node uses `mining: { auto: true, interval: 1000 }` (v1.9, D11) |
| Suggested repo layout | `/contracts`, `/circuits`, `/services` (data, proof, recovery, ipfs-proxy), `/frontend`, `/test`, `/scripts`, `/fixtures` |
| Compiler settings | In `hardhat.config.js`: `solidity: { version: '0.8.24', settings: { evmVersion: 'cancun', optimizer: { enabled: true, runs: 200 } } }`. Without `evmVersion: 'cancun'`, OpenZeppelin 5.6 fails to compile |
| Local ports | Frontend 5173, Hospital B viewer 5174, RecoveryCoordinator API and dealer launcher 4100, IPFS proxy 4101 (v1.9), Hardhat node 8545 |
| Environment variables | `RPC_URL`, `PINATA_JWT` and `PINATA_GATEWAY` (read by the IPFS proxy only, blank in `.env.example`, never committed), the post-deploy contract addresses, the relayer account key, `DEMO_WINDOW_DURATION` (default 180), `DEMO_TIER2_DELAY` (default 60), `COLLECTION_TIMEOUT` (default 20), and `FULL_MODE_TTL` (default 300), all timings in seconds (v1.9 adds `FULL_MODE_TTL` from errata 18 and the proxy's variables) |
| Demo fixture | `fixtures/demo-identities.DEMO-ONLY.json`, listed in `.gitignore`. It holds the demo identities' secrets and addresses and is never committed (DAT-9) |
| One-command demo | A single `npm run demo:breakglass` orchestrates the node start, deploy, seed, services, and the scripted trigger. The PRD's reproducibility goal is a real, testable claim rather than an aspiration. The other flows (routine sharing, lost-key recovery, cross-hospital access) are not part of this one command |

### Deploy script wiring (`scripts/deploy.js`)

The v1.8 TRD did not fix a deployment order or constructor arguments. The order below follows from the contract dependencies and is *specified here*. Steps marked (TRD) or (errata) are stated in those sources, and the rest are inferred.

1. Deploy `RoleRegistry`, `PatientIdentity`, `PatientAccountFactory` (v1.9), and `AuditLog`. They depend on nothing.
2. Deploy `ConsentRegistry` (identity, roles, audit), `RecordRegistry` (identity, audit), the generated `ProofVerifier`, and `ProofGateway` (verifier, consent registry, role registry, identity, audit).
3. Deploy `RecoveryVault` (identity, audit, `TIER2_DELAY`). Then deploy `BreakGlassCapsule` (role registry, identity, audit, vault, `WINDOW_DURATION`), after the vault, because its constructor takes the vault's address (errata 16).
4. Call `RecoveryVault.setBreakGlassCapsule` once, now that both exist (errata 16).
5. Call `AuditLog.registerEmitter` for every logging contract: ConsentRegistry, RecordRegistry, ProofGateway, RecoveryVault, and BreakGlassCapsule (TRD).
6. Call `ConsentRegistry.setProofVerifier` with the ProofGateway's address (inferred).
7. Assert `WINDOW_DURATION` is at least twice `TIER2_DELAY`, and refuse to deploy otherwise (TRD).
8. Write the deployed addresses to config.

### Seed script (`scripts/seed-synthea-data.js`)

Loads the Synthea FHIR R4 data for DAT-6, giving each resource a valid FHIR id and rewriting `urn:uuid:` references to `Patient/<id>`. It creates the demo identities as Hardhat's unlocked accounts, registers their public keys, has the scripted patient share a Claim permission with Dr. Rao, has Dr. Rao issue the demo claim with that token, and writes the demo-only fixture. It simulates records authored by doctors and labs, because the prototype's `RecordRegistry` accepts only the patient's account (limit 18). `scripts/demo-breakglass.js` runs the full break-glass sequence end to end.

## Security controls

Twenty-two controls tie each protection to the contract or service that provides it. Rows marked v1.9 are new.

| Control | Applies to | Notes |
| --- | --- | --- |
| Role- and owner-based access modifiers (`onlyRole`, `onlyPatient(tokenId)`, `onlyRegisteredCustodian`) | All state-changing functions listed in Smart contract specifications | OpenZeppelin `AccessControl` is the accepted base for RoleRegistry: a well-audited standard pattern rather than a hand-rolled one |
| Checks-effects-interactions ordering, plus a reentrancy guard on any function that calls out before finishing its own state updates | ProofGateway, RecoveryVault, BreakGlassCapsule | OpenZeppelin `ReentrancyGuard` is accepted for the same reason as above. `triggerEmergencyAccess` calls the trusted vault before storing the window, which is acceptable because the vault is wired at deploy time |
| Input validation | Consent issuance (expiry must be future-dated), Setup registration (2 <= k <= n+1, enforced on-chain per decision D2) | Validated in the contract itself, not only in the frontend, so a malformed direct call still fails safely |
| Identity binding (v1.9, D15) | `PatientIdentity.register` | The DID hash must equal the hash of the DID string built from the caller's own address, so nobody can register another party's identifier first |
| Trustee category cap (v1.9, D14) | `RecoveryVault` registration and rotation | No category may hold k or more of the n+1 trustees, so k trustees of one kind cannot rebuild S. The all-one-category check is kept |
| Access-check logging (v1.9, D13) | `ConsentRegistry.checkAccess` | `msg.sender` must equal `requester`, so one party cannot log denials in another's name. A denial returns false rather than reverting, so it is audit-logged |
| Randomness discipline | Every place a secret, salt, or key is generated | `crypto.randomBytes` or `crypto.getRandomValues` only. A lint rule or pre-commit grep for `Math.random` under `/services`, `/frontend`, and any circuit helper scripts is a cheap way to make this enforceable rather than aspirational |
| No raw share, salt, or private key ever appears in calldata or an event | RecoveryVault, AuditLog | Testable directly: the ABI of RecoveryVault should contain no function that accepts a raw share value as a parameter. See the design decision in Recovery and break-glass technical flow |
| Device keystore | A party's secret on its device: S for a patient, its own secret for a clinician, lab, or second institution (the key pairs are derived from it) | Encrypted at rest with PBKDF2 (at least 600,000 iterations) and AES-256-GCM. Unlocked keys live only in memory for the open tab. Never written in plaintext, logged, or sent to the Node services except wrapped to a one-time key |
| Relayer trust | PatientAccount and the relayer | The relayer can delay or drop a signed action but cannot forge one. Signatures are bound to an account, a nonce, and a deadline, so replay and late submission both fail |
| Account creation (v1.9, D10) | PatientAccountFactory | The owner is fixed in the constructor and the address is deterministic, so a caller who creates an account for someone else gains no control. Creating it twice returns the same account |
| Permissionless closing | `BreakGlassCapsule.closeWindow` | Safe for anyone to call, because it only succeeds after the stored expiry and only once, and all it does is log |
| Local service access (v1.9, extended) | Dealer launcher, coordinator, and IPFS proxy | Bound to 127.0.0.1, with a per-session bearer token generated at startup and CORS limited to the frontend origin (D17). The coordinator also requires a per-request token. Shares are accepted only when wrapped to the request's one-time key, and S only when wrapped to the dealer's one-time key |
| IPFS proxy confinement (v1.9, D12) | IPFS proxy | Holds the Pinata credentials and accepts only well-formed ciphertext envelopes, so a client mistake cannot upload plaintext. It never receives a key |
| Trusted emitters | `AuditLog.log` | Because the caller passes the actor, only allow-listed contracts may call it, and the allow-list is admin-only |
| Full-mode delivery of S | Lost-key recovery | S leaves the coordinator only wrapped to the client's one-time key and only once. The coordinator checks the on-chain request first, so a request that the patient's account did not open gets nothing, and it wraps only to a key whose hash matches the one stored with the request. Cancelled and completed requests are refused |
| Issued claims | ProofGateway | Only a Doctor or Lab holding an active Claim permission from the patient that names it can register a commitment, and only the original issuer can replace it. The patient or the issuer can revoke it. A proof must match the stored commitment, the required threshold, and an unused nonce derived from the chain id, the gateway address, and the request id, so it cannot be replayed, reused for another patient, or made with an invented number. Issuing and revoking are logged |
| Demo identity secrets | Seed script fixture | Written only to a git-ignored, labelled demo-only file and never used outside the demo. It holds no transaction-signing keys |
| One clock (v1.9, D11) | Chain, coordinator, and screens | Deadlines read chain time and the demo never advances it, so a countdown cannot disagree with the contract |
| Derived identity keys | Key pairs derived from S | Whoever holds k shares can rebuild S and so the signing key. Mixed trust categories (T1-5) are the mitigation. Documented in the threat model |
| Record metadata | RecordRegistry | Public by design. Record types, counts, creators, and timing are visible, and hashing the type does not hide it. Documented as a limit, not mitigated |
| PQC provider abstraction | All ML-DSA and ML-KEM call sites | Swapping the backend library never requires touching contract or protocol logic (see Cryptographic specification) |

## Traceability to the PRD

Every one of the PRD's 67 requirement IDs maps to the section that specifies it. Ranges are inclusive, and the three IDs added in PRD v1.6 (PAT-7, PAT-8, DAT-10) have their own rows.

| PRD requirements | Specified in this TRD |
| --- | --- |
| PAT-1, PAT-2, PAT-5, PAT-6 | System architecture (component catalog); Data schemas (identifiers); Smart contract specifications (PatientIdentity.sol); Cryptographic and zero-knowledge specification (keys derived from S) |
| PAT-3, PAT-4 | Smart contract specifications (PatientAccount.sol, RoleRegistry.sol); Technology stack and versions |
| PAT-7, PAT-8 (new) | Smart contract specifications (PatientAccountFactory.sol, PatientIdentity.sol); Frontend technical requirements (routing); Security controls; Testing requirements |
| CON-1 to CON-6 | Data schemas (consent token, wrapped type keys); Smart contract specifications (ConsentRegistry.sol); Module interface contracts and local services |
| ZKP-1 to ZKP-7 | Cryptographic and zero-knowledge specification (circuit: ZKP-1, ZKP-2, ZKP-6, ZKP-7); Smart contract specifications (ProofVerifier.sol and ProofGateway.sol: ZKP-3, ZKP-4, ZKP-7); Non-functional technical requirements (gas, ZKP-5) |
| AUD-1 to AUD-3 | Data schemas (audit event, signed audit record); Smart contract specifications (AuditLog.sol, and BreakGlassCapsule.sol for AUD-2) |
| DAT-1 to DAT-9 | Data schemas (FHIR resource, record envelope, record reference, issued claim, keystore, fixture); Smart contract specifications (RecordRegistry.sol: DAT-8); Module interface contracts (authorized retrieval path: DAT-3); Cryptographic and zero-knowledge specification (DAT-4, DAT-5); Frontend technical requirements (DAT-9); Environment, build, and deployment (seed script: DAT-6) |
| DAT-10 (new) | Module interface contracts and local services (IPFS proxy); Frontend technical requirements (where crypto runs); Security controls; Testing requirements |
| T1-1 to T1-10 | Recovery and break-glass technical flow; Cryptographic and zero-knowledge specification (commitment format, key hierarchy); Smart contract specifications (RecoveryVault.sol); Module interface contracts and local services (dealer launcher, coordinator); Known limits |
| BG-1 to BG-8 | Smart contract specifications (BreakGlassCapsule.sol, RoleRegistry.sol); Recovery and break-glass technical flow (state machine, sequences, timing parameters); Environment, build, and deployment (demo script) |
| UI-1 to UI-9 | Frontend technical requirements; Module interface contracts and local services; Data schemas (emergency card, keystore file) |
| INT-1 to INT-3 | Module interface contracts and local services (authorized retrieval path); System architecture (Hospital B viewer); Frontend technical requirements; Testing requirements (end to end) |
| SEC-1, SEC-2 | Known limits; Security controls; Cryptographic and zero-knowledge specification (post-quantum scope) |
| E2E-1 | Testing requirements; Environment, build, and deployment (one-command demo) |
| Non-functional rows: time handling, local-service security (new) | Technology stack and versions (chain clock); Module interface contracts and local services (shared rules); Security controls; Testing requirements |
