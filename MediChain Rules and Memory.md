# MediChain Rules and Memory

Version 1.0. For AI-assisted (vibe) coding. Based on Errata 1 to 19, Backend Schema v1.1, TRD v1.8 and the MediChain Architecture. Group 02, BCS685. Prepared 12 October 2026.

## 1. How to use this file

Load sections 2 to 4 plus your own module before every AI coding session, and build one chapter per session. Each chapter ends in a Done when checklist, and a chapter is finished only when every box ticks.

1. Give the AI sections 2 and 3, your module's header, and one chapter.
2. Ask it to restate the Done when list and write the tests first.
3. Let it implement only that chapter, inside your module's folders.
4. Run the tests. Tick the boxes here only when they pass.
5. Commit with the chapter id first, for example `M3.4: dealer CLI`. Add one line to the decision log (last section) if a decision came up.

Starter prompt:

```text
Read the MediChain project memory and rules. Implement chapter M3.4 only, in my module's folders. Write the tests first. Stop when every Done when item passes. If the spec is silent or conflicts with the errata or the schema, stop and ask me. Do not invent.
```

| Module | Owner | Owns | Chapters |
| --- | --- | --- | --- |
| M1 | Anush | Identity, accounts, roles, consent, records, relayer, threat model | M1.1 to M1.6 |
| M2 | Punith | Audit log, claim circuit, proof service and gateway, benchmarks | M2.1 to M2.6 |
| M3 | Rohith | Post-quantum crypto, keystore, DataService, dealer, guardian set, seed data, Hospital B | M3.1 to M3.7 |
| M4 | Yagna | Break-glass, Tier 2 requests, coordinator API, frontend, demo | M4.1 to M4.8 |

## 2. Project memory

MediChain is a prototype health-record platform where the patient holds the keys. Records are FHIR-style JSON, encrypted on the patient's side and stored on IPFS. A local Hardhat chain holds only hashes, consent state, public keys, share commitments, proof results and an audit trail. Break-glass emergency access and guardian-based key recovery are first-class, and every exceptional action is logged.

**Source of truth, highest first:** Errata 1 to 19, Backend Schema v1.1, TRD v1.8, the Architecture file. This file summarizes them and adds no decisions. If the code needs something none of them say, stop and ask.

**The planes rule.** State lives on-chain, computation off-chain, encrypted bytes in storage. Nothing that touches plaintext PHI, a raw share, a salt or a private key crosses into the on-chain plane.

### Vocabulary

| Term | Meaning |
| --- | --- |
| patientId | keccak256 of the UTF-8 DID string (did:medichain:0x...) |
| account | The patient's PatientAccount contract address. It is msg.sender everywhere, so it is the patient on-chain |
| owner | The wallet key that signs EIP-712 Execute messages for the account |
| party | Any registered identity (patient account, doctor, lab, Hospital B, Coordinator), looked up by address |
| resourceType | keccak256 of a FHIR type name: Patient, Observation, Consent, EmergencyCapsule, Claim |
| roleId | keccak256(bytes(roleName)) for Doctor, Lab, Insurer, IoTDevice or EmergencyResponder |
| tokenId, entryId, requestId | keccak256(abi.encode(block.chainid, address(this), msg.sender, counter)), one counter per contract |

### Fixed values

| Item | Value |
| --- | --- |
| Chain | Hardhat built-in network only, chain ID 31337, RPC http://localhost:8545 |
| Ports | Frontend 5173, Hospital B viewer 5174, coordinator and dealer launcher 4100 (127.0.0.1 only) |
| Contracts | Solidity ^0.8.24, compiled with 0.8.24, evmVersion cancun, OpenZeppelin 5.x |
| Hardhat | hardhat@^2.28, @nomicfoundation/hardhat-toolbox@^6, @nomicfoundation/hardhat-ethers@^3.1.3, ethers@^6.14. Pin Hardhat 2, never 3 |
| Runtime | Node 20.19 or later, React 18, Vite |
| Post-quantum | @noble/post-quantum at one exact pinned version (0.7.1 tested). ML-DSA-65 and ML-KEM-768 |
| Zero-knowledge | circom 2.2.3, snarkjs Groth16 on BN254, circomlib and circomlibjs, Powers of Tau power 12 |
| Timing (demo) | WINDOW\_DURATION 180 s, TIER2\_DELAY 60 s, COLLECTION\_TIMEOUT 20 s, FULL\_MODE\_TTL 300 s. WINDOW\_DURATION must be at least twice TIER2\_DELAY |
| Sizes | AES-256-GCM with a 12-byte IV and 16-byte tag. PBKDF2 at least 600,000 iterations. Claim value and threshold fit 8 bits, claim salt 31 bytes, share salt at least 16 bytes |
| Fixed strings | medichain-share-v1 for share commitments. HKDF info: medichain-dsa-v1, medichain-kem-v1, medichain-type-v1:, medichain-record-v1:, medichain-capsule-v1. EIP-712 domain MediChainAccount, version 1 |

### Errata that change the code

Build it this way, even where the TRD says otherwise:

- `setReleaseMode(requestId, patientId, requester, mode)` is external with an onlyBreakGlassCapsule check. The capsule address is set once through `setBreakGlassCapsule(address)`, admin-only, reverting if already set (errata 1, 2, 16).
- RecoveryVault is the source of truth for patientId, requester, release mode, cancelled and tier2EndsAt. BreakGlassCapsule keeps only windowExpiry and closed. closeWindow treats windowExpiry == 0 as an unknown request (errata 2, 19).
- `completeRecovery` requires mode FullRecord (errata 17). ReleaseMode is None, CapsuleOnly, FullRecord, and code and tests use the names, never the numbers (errata 11).
- A share is the exact string `secrets.share()` returns. Never hex-decode it. The commitment is `SHA256(utf8(shareString) || salt || utf8('medichain-share-v1'))` (errata 4).
- patientId = keccak256(utf8(didString)). The type-key HKDF info uses the resource type NAME, not its hash. A WrappedTypeKey serializes as UTF-8 JSON (errata 5). roleId follows the vocabulary above, with a name lookup in checkAccess (errata 3).
- Ids use the per-contract counter formula in the vocabulary (errata 6).
- The emergency capsule values live in a CapsulePayload, encrypted under capsuleKey and registered as type EmergencyCapsule (errata 7).
- Audit refIds: ClaimIssued uses the tokenId of the Claim token, ClaimRevoked uses claimType, and custodianCoSign emits TierEscalated (errata 8).
- The signed audit record has an optional trustees field, lives in one append-only JSONL file per patient, and is signed by a registered demo Coordinator identity (errata 9).
- `getGuardianSet(patientId)` returns threshold, epoch and updatedAt (errata 10).
- If all guardians share one category, the contract rejects it and the UI warns first (errata 12).
- Every FHIR resource has a subject reference except Patient, where id plays that role (errata 13). The record signature covers the ciphertext only, a documented limitation (errata 14).
- Pinata settings are PINATA\_JWT and PINATA\_GATEWAY, listed blank in .env.example (errata 15).
- Lost-key requests have no on-chain window, so the coordinator enforces FULL\_MODE\_TTL and mirrors it in the event's windowExpiry (errata 18).

### Shared files

These splits are proposed here. Both owners confirm them before starting.

| Shared file | Split |
| --- | --- |
| RecoveryVault.sol | Rohith: the guardian set (registerShareCommitments, rotateShareCommitments, getCommitments, getGuardianSet). Yagna: requests (startRecovery, custodianCoSign, cancelRecovery, completeRecovery, setReleaseMode, setBreakGlassCapsule, getRecovery, getRequestStatus) |
| RecoveryCoordinator | Rohith: share intake, commitment check, Lagrange, wiping (M3.6). Yagna: HTTP and SSE API, modes, windows and TTL, audit records (M4.5) |
| scripts/deploy.js | Anush creates it (M1.1). Each owner adds their own deploy and wiring lines. Yagna owns the final run order (M4.8) |
| Device keystore | Rohith builds the library (M3.2). Yagna wires it into the screens (M4.6) |

## 3. Rules

### Security (never break these)

- Never use Math.random for a secret, salt or key. Use crypto.randomBytes or crypto.getRandomValues.
- No plaintext PHI, raw share, salt or private key in calldata, events, logs, error messages or on-chain storage.
- Never log or commit S, shares, salts, keys, passphrases, .env files, keystores or the demo fixture.
- Policy before cryptography: on-chain role, capsule and request checks run first, and a failure reverts so no cryptography runs.
- Audit is atomic: a function the audit table lists calls AuditLog.log in the same transaction, and a log revert reverts the call.
- S leaves a process only wrapped to a one-time key. Dealer and coordinator zeroize buffers after use.
- Local services bind to 127.0.0.1 and require the per-session bearer token.
- Every post-quantum call goes through PQCProvider. Pin exact versions and use .js subpath imports.
- Enforce every rule in the contract, not only in the UI, so a malformed direct call still fails.

### Code

- Solidity: custom errors, an event for every state change, NatSpec on external functions, named constants instead of magic numbers, checks-effects-interactions, no tx.origin.
- Use the exact function, struct, enum and event names in Backend Schema v1.1. Never rename an interface to suit your code.
- Services and frontend default to TypeScript in strict mode, because the TRD's interfaces are TypeScript. The TRD does not mandate it, so a module may use plain JavaScript with the same shapes.
- No hard-coded addresses. Read them from the config the deploy script writes.
- No dead code. A TODO must name a chapter id.

### Testing

- Contracts use Hardhat with Chai and Mocha. Node services use Jest. Contract tests use a mock PQCProvider.
- Each Done when box becomes a named test. Test the revert and the failure path, not only success.
- Advance chain time with Hardhat's time helpers. Never sleep in a test.
- Shamir needs property tests: every combination of k or more shares rebuilds the same S, and every combination of k-1 does not.

### Working with the AI

- One chapter per session. Stay in your module's folders. Change another module's file only through a pull request its owner approves.
- Interfaces change only through the Backend Schema and the errata, agreed by the team. Never change one inside a chapter.
- When the spec is silent, stop and ask. Mark any unavoidable guess with an `inferred` comment.
- Run all tests before a commit. Keep the README run steps current.

## 4. Contracts between modules

Until a provider ships, consume it through its Backend Schema interface with a stub, then swap in the real contract at the integration gate. Never fork an interface.

| Provider | Consumer | What the consumer relies on | Chapter |
| --- | --- | --- | --- |
| AuditLog | Every contract | log and registerEmitter. Ships first, so a stub is needed only for a day | M2.1 |
| RoleRegistry | ProofGateway, BreakGlassCapsule, ConsentRegistry | assignRole, hasRole | M1.1 |
| PatientIdentity and PatientAccount | RecordRegistry, RecoveryVault, DataService, every screen | register, accountOf, key lookups, executeSigned | M1.2 |
| Relayer | DataService, frontend | RelayRequest in, txHash out | M1.2 |
| RecordRegistry | DataService, frontend | addRecord, recordsOf, getRecord | M1.3 |
| ConsentRegistry | ProofGateway, DataService, frontend | issueConsent, checkAccess, isValid, getConsent, KeysWrapped | M1.4, M1.5 |
| PQCProvider and keystore | Contract tests, DataService, frontend | The six-method interface, the one-secret keystore | M3.1, M3.2 |
| ProofGateway and ProofService | Seed script, frontend | setCommitment, verifyEligibility, proof generation | M2.3, M2.4 |
| RecoveryVault, both halves | Coordinator, frontend, BreakGlassCapsule | Commitments, requests, Tier 2 | M3.5, M4.3 |
| Dealer launcher | Guardian screen | POST /dealer/run and the secret call | M3.4 |
| Coordinator API | Responder and recovery screens, scripts | The five endpoints and the progress event | M4.5 |
| BreakGlassCapsule | Frontend, scripts | triggerEmergencyAccess, closeWindow, getRequest | M4.2 |

Integration gates, used as build order:

1. **G1 Foundations:** RoleRegistry, PatientIdentity with PatientAccount, AuditLog, and PQCProvider with its mock are deployed and tested.
2. **G2 Records and consent:** RecordRegistry, ConsentRegistry core and the DataService write and read path work together.
3. **G3 Proofs and recovery:** ProofGateway is live, both halves of RecoveryVault exist, and the dealer and coordinator core work.
4. **G4 Demo:** BreakGlassCapsule, the screens, the seed script, Hospital B and the end-to-end tests pass.

## Module 1: Anush, identity, consent and records

- **Scope:** PatientIdentity, PatientAccount, RoleRegistry, ConsentRegistry, RecordRegistry, the Relayer and the threat model.
- **Folders:** /contracts (these five), /services/relayer, /test, /docs/threat-model.md.
- **Consumes:** AuditLog (stub until M2.1), the proof hook (stub until M2.4).
- **Provides:** identity and key lookup, roles, consent tokens with KeysWrapped, record references, sponsored transactions.
- **Stack:** Solidity, OpenZeppelin 5.x, Hardhat 2.x, ethers 6, Chai and Mocha, Jest for the relayer.

### M1.1 Environment and RoleRegistry

**Goal.** A working Hardhat project and the first contract the others depend on.

**Build.** A Hardhat 2.x project with the pinned versions in section 2, local network only. RoleRegistry as in Backend Schema 3.3: the Role enum, admin-only assignRole through OpenZeppelin AccessControl, and hasRole. Create scripts/deploy.js with a step that writes deployed addresses to config.

**Done when.**

- [ ] npx hardhat test runs on a clean checkout.
- [ ] assignRole reverts for a non-admin.
- [ ] hasRole is true after assignment and false for an unassigned account.
- [ ] deploy.js deploys RoleRegistry and writes its address to config.

### M1.2 PatientIdentity, PatientAccount and the relayer

**Goal.** Every party has an identity with two public keys, and a patient can act without handling gas.

**Build.** PatientIdentity (Backend Schema 3.1): register(didHash, dilithiumPubKey, mlkemPubKey), accountOf, patientIdOf, getDilithiumKey, getMLKEMKey. One identity per address and per patientId, and no updateKeys. PatientAccount (3.2): owner, nonce, execute for the owner wallet, and executeSigned(target, data, deadline, sig) over the EIP-712 Execute(target, data, nonce, deadline) message, with the nonce incremented before the call. A Node relayer that takes a RelayRequest, submits executeSigned, pays gas and returns the txHash.

**Done when.**

- [ ] register stores both keys and rejects a duplicate address or patientId.
- [ ] executeSigned works with a valid signature and reverts on replay, wrong signer and expired deadline.
- [ ] The target contract sees the account address as msg.sender.
- [ ] The relayer submits a signed action, and the owner-wallet execute fallback works.

### M1.3 RecordRegistry

**Goal.** Grantees can find a patient's encrypted records by type.

**Build.** Backend Schema 3.5. addRecord(patientId, resourceType, cid, contentHash, creator) is callable only when accountOf(patientId) is msg.sender. The cid is a string of 1 to 100 bytes. Add recordsOf and getRecord, a per-contract entryCounter for entryId, no delete, and a RecordAdded audit log.

**Done when.**

- [ ] Only the patient's account can add a record.
- [ ] An empty cid and a 101-byte cid both revert.
- [ ] recordsOf returns records in the order added.
- [ ] RecordAdded is logged with the entryId as refId.

### M1.4 ConsentRegistry: tokens and key delivery

**Goal.** A patient can grant, and later revoke, time-limited access by resource type.

**Build.** Backend Schema 3.4. issueConsent(grantee, roleId, resourceTypes, expiresAt, wrappedTypeKeys) checks that the caller is a registered patient account, the expiry is in the future, and there is one wrapped key per type. It rejects the EmergencyCapsule type, accepts an empty entry for a Claim type, stores the token and emits KeysWrapped. Wrapped keys are emitted, never stored. Add revokeConsent for the issuing patient only, plus isValid, getConsent, tokenIdsOf and tokenIdsOfGrantee, and the audit logs.

**Done when.**

- [ ] Issuing stores a token and emits KeysWrapped with matching arrays.
- [ ] A past expiry and a length mismatch both revert.
- [ ] The EmergencyCapsule type reverts.
- [ ] Only the issuing patient can revoke, and revoke costs one storage update plus an audit event (TRD target 30,000 to 50,000 gas).

### M1.5 Access check, roles and the proof hook

**Goal.** Every use of a token is checked against the token and the grantee's role, and logged.

**Build.** checkAccess(tokenId, requester, resourceType) returns true only if the token is not revoked, not expired, includes the type, and the requester's role matches. Resolve roleId to the Role enum by name (errata 3). Emit AccessGranted or AccessDenied. Add the admin-only setProofVerifier hook, stubbed until M2.4.

**Done when.**

- [ ] A valid token with a matching role returns true and logs AccessGranted.
- [ ] A revoked token, an expired token, a wrong type and a wrong role each return false and log AccessDenied.
- [ ] A non-admin cannot set the proof hook.
- [ ] A Claim token can be issued with an empty wrapped-key entry.

### M1.6 Tests and the threat model

**Goal.** The consent layer is proven, and the security analysis is written.

**Build.** A test suite for consent flows: expiry, revocation and unauthorized access, using time helpers. Then docs/threat-model.md: for each layer (chain, services, storage and custody, client) what it exposes and what it protects, mapped against the survey's gap table, including every known limit in the TRD and Architecture section 9.

**Done when.**

- [ ] The suite covers expiry, revocation and unauthorized access.
- [ ] The threat model has a row for each layer and each known limit.
- [ ] Every limit names its mitigation or says there is none.

## Module 2: Punith, proofs, audit and benchmarks

- **Scope:** AuditLog, the claim circuit, ProofService, ProofVerifier with ProofGateway, audit coverage and the benchmarks.
- **Folders:** /contracts (AuditLog, ProofGateway, the generated ProofVerifier), /circuits, /services/proof, /bench, /test.
- **Consumes:** RoleRegistry (M1.1), consent token reads (M1.4).
- **Provides:** AuditLog to everyone, which ships first, and claim proofs to the seed script and the frontend.
- **Stack:** circom 2.2.3, snarkjs, circomlib and circomlibjs, Solidity, Hardhat, Jest.

### M2.1 AuditLog (ship first)

**Goal.** One canonical append-only feed that every other contract can call.

**Build.** Backend Schema 3.7. The ActionType enum, which is append-only, registerEmitter for admins, and log(patientId, actor, action, refId) emitting AuditLogged(patientId, actor, action, refId, timestamp) with patientId, actor and action indexed. Only allow-listed emitters may log, and there is no other storage. Publish the interface so others can stub it.

**Done when.**

- [ ] An allow-listed emitter can log and any other caller reverts.
- [ ] The event arguments match the schema.
- [ ] The enum order is documented and new values are only appended.

### M2.2 ZK toolchain and the claim circuit

**Goal.** A proof that a hidden value meets a threshold and matches a commitment.

**Build.** Start with a hello-world circuit. Then the claim circuit: private value and salt, public threshold, commitment and nonce. Poseidon(2) of value and salt equals the commitment, GreaterEqThan(8) of value and threshold equals 1, and the nonce is squared so the compiler keeps it. No outputs. Expect 531 constraints (253 non-linear, 278 linear). Powers of Tau power 12, then a local phase 2. The public signal order is threshold, commitment, nonce.

**Done when.**

- [ ] Compiling reports 531 constraints, 3 public and 2 private inputs.
- [ ] A valid witness proves and verifies off-chain.
- [ ] A value below the threshold fails at witness generation.
- [ ] A tampered proof fails verification.

### M2.3 ProofService and claim helpers

**Goal.** A service that turns a claim into a proof the contract accepts.

**Build.** A Node module with the circomlibjs Poseidon commitment (8-bit value, 31 random bytes of salt), the nonce keccak256(abi.encode(chainId, gatewayAddress, requestId)) reduced modulo the BN254 scalar field, witness building, Groth16 proving with snarkjs, exportSolidityCallData into 8 proof elements and 3 public signals, and a batching helper.

**Done when.**

- [ ] The JavaScript commitment equals the circuit's.
- [ ] The JavaScript nonce equals the contract's expectedNonce.
- [ ] A proof for value 25 and threshold 18 verifies.
- [ ] A value or threshold above 255 is rejected before proving.

### M2.4 ProofVerifier and ProofGateway

**Goal.** On-chain verification with issuer gating and replay protection.

**Build.** Export the Solidity verifier from snarkjs. ProofGateway (Backend Schema 3.6): setCommitment(patientId, claimType, commitment, tokenId) requires the Doctor or Lab role and an active Claim token that names the caller, only the original issuer may replace a claim, and a zero or out-of-field commitment reverts. Add revokeClaim, getClaim and expectedNonce. verifyEligibility(patientId, claimType, requestId, requiredThreshold, proof, publicSignals) checks, in order: the claim exists, publicSignals\[0\] equals the threshold, \[1\] equals the stored commitment, \[2\] equals expectedNonce, the nonce is unused, then the verifier. Success marks the nonce used and logs ProofVerified. An invalid proof returns false without using up the nonce. Add verifyBatch, the ClaimCommitted and ClaimRevoked events, and the audit refIds from errata 8.

**Done when.**

- [ ] A valid proof verifies and uses up its nonce, and a replay reverts.
- [ ] A tampered proof returns false and the nonce is still usable.
- [ ] setCommitment reverts without the role or without a matching token.
- [ ] A proof against a revoked claim reverts with no claim.
- [ ] A batch of two proofs verifies in one transaction.

### M2.5 Audit coverage

**Goal.** Nothing that matters succeeds without a trace.

**Build.** Check that every logging contract is registered as an emitter and logs the right action and refId (Architecture section 16). Write an integration test that runs a full flow and asserts the event sequence. Add a helper that reads AuditLogged events by patientId in order, for the Activity screen.

**Done when.**

- [ ] A test asserts each ActionType appears with the right actor and refId.
- [ ] A reverting AuditLog reverts the calling function.
- [ ] The event reader returns a patient's events in order.

### M2.6 Benchmarks and comparison

**Goal.** Real latency and gas numbers, compared with the survey.

**Build.** A bench script for proof generation latency and for gas, single and batched, with enough repeats for a median and a 95th percentile. Compare with the numbers reported, where available, in the 10 surveyed papers. Do not quote the skeleton estimates of about 270,000 gas for one proof and about 513,000 for two as results.

**Done when.**

- [ ] The script outputs median and 95th percentile latency and gas for one proof and a batch.
- [ ] The comparison table cites each paper or says not reported.
- [ ] The bottlenecks are written down.

## Module 3: Rohith, data layer, post-quantum crypto and Tier 1 recovery

- **Scope:** PQCProvider and the key hierarchy, the device keystore library, DataService, the RecoveryDealer and its launcher, the guardian-set half of RecoveryVault, the share-intake core of the coordinator, the seed script and the Hospital B viewer.
- **Folders:** /services/data, /services/recovery (dealer and share logic), /contracts/RecoveryVault.sol (guardian half), /scripts/seed-synthea-data.js, /frontend/hospital-b (a second Vite app), /fixtures (git-ignored).
- **Consumes:** M1 (identity, records, consent), M2 (AuditLog, and ProofGateway for the demo claim), M4 (the request half of the vault and the coordinator API shell).
- **Provides:** PQCProvider, the keystore, DataService, the dealer, share commitments and the seed data.
- **Stack:** Node 20.19 or later, @noble/post-quantum at an exact pin, secrets.js-grempe, the Pinata SDK, node:crypto, Jest.

### M3.1 PQCProvider and the key hierarchy

**Goal.** One interface for every post-quantum call, and the key tree that grows from S.

**Build.** The interface: dsaKeygen, sign, verify, kemKeygen, encapsulate, decapsulate. NoblePQCProvider over @noble/post-quantum at one exact version, with .js subpath imports. A MockPQCProvider for contract tests. The HKDF-SHA256 hierarchy: dsaSeed (32 bytes), kemSeed (64 bytes), typeKey from the type name, recordKey from the typeKey and recordId, and capsuleKey.

**Done when.**

- [ ] Re-deriving from the same S gives the same public keys.
- [ ] Type, record and capsule keys all differ from each other.
- [ ] Sign and verify round-trip, and a wrong key fails verification.
- [ ] Encapsulate and decapsulate round-trip.
- [ ] The mock and the real provider pass the same interface tests.

### M3.2 Device keystore and other parties' keystores

**Goal.** A passphrase-protected home for one secret per party.

**Build.** The Backend Schema 5.7 shape: PBKDF2-HMAC-SHA256 with at least 600,000 iterations, then AES-256-GCM, holding one secret only (S for a patient, the party's own secret otherwise). Backup export and restore. Keystores for the clinician, lab, Hospital B and Coordinator that re-derive their registered public keys. A writer for the git-ignored fixtures/demo-identities.DEMO-ONLY.json.

**Done when.**

- [ ] A keystore round-trips, and a wrong passphrase fails the GCM tag.
- [ ] A backup file restores the secret.
- [ ] A test asserts the iteration count is at least 600,000.
- [ ] A clinician's re-derived public key equals the one registered for it.
- [ ] The fixture file is git-ignored.

### M3.3 DataService: write and read path

**Goal.** Records are encrypted, signed, pinned and registered, and read back safely.

**Build.** FHIR R4-inspired builders for Patient, Observation and the Consent projection, with a subject on every resource except Patient (errata 13). AES-256-GCM per record under recordKey, an ML-DSA-65 signature over the ciphertext, the RecordEnvelope, a Pinata upload (PINATA\_JWT, PINATA\_GATEWAY), and addRecord through the patient's account. The read path from Architecture section 11: hash check, signature check, unwrap the typeKey, decrypt. WrappedTypeKey serializes as UTF-8 JSON. The CapsulePayload is encrypted under capsuleKey and registered as EmergencyCapsule (errata 7).

**Done when.**

- [ ] A record round-trips with a known key.
- [ ] A tampered ciphertext fails GCM, and a record signed by the wrong key fails verification.
- [ ] A hash mismatch stops the read before decryption.
- [ ] A granted type reads, and an ungranted type fails.

### M3.4 RecoveryDealer and launcher

**Goal.** Split S once, safely, and hand out encrypted shares.

**Build.** A one-shot dealer CLI and the loopback launcher on 127.0.0.1:4100: POST /dealer/run and POST /dealer/:sessionId/secret, with a per-session bearer token. S is accepted only wrapped to the dealer's one-time ML-KEM key. Split with secrets.js-grempe using a CSPRNG. Keep each share as its exact string and commit with `SHA256(utf8(shareString) || salt || utf8('medichain-share-v1'))`, with a salt of at least 16 bytes. Encrypt each share and salt to its trustee's ML-KEM key, return n+1 bundles and commitments, then zeroize and exit.

**Done when.**

- [ ] Every combination of k or more shares rebuilds S, and every combination of k-1 does not.
- [ ] A 99-character share string commits correctly, with no hex-decoding.
- [ ] The launcher rejects non-loopback callers and calls without the token.
- [ ] S is rejected unless wrapped to the dealer's key, and the process exits afterwards.

### M3.5 Guardian half of RecoveryVault

**Goal.** Share commitments on-chain, with the rules that stop a weak guardian set.

**Build.** Backend Schema 3.8, guardian half only. registerShareCommitments(commitments, threshold) once per patient, rotateShareCommitments (epoch plus one, updatedAt set), getCommitments and getGuardianSet. Enforce 2 <= threshold <= commitments.length, reject a set where all guardians share one category (the custodian row is excluded), indexes unique and contiguous from 1 to n+1, exactly one InstitutionalCustodian at index n+1, and unique trustee addresses. Agree the file layout with Yagna, who owns the request half (M4.3).

**Done when.**

- [ ] A valid set registers.
- [ ] k of 1, and k above n+1, revert.
- [ ] A single-category set and a duplicate trustee address revert.
- [ ] Rotation bumps the epoch, and old commitments stop matching.
- [ ] Only the patient can register, and only once.

### M3.6 Coordinator core: verify and rebuild

**Goal.** The part of the coordinator that handles shares, as a pure module Yagna's API calls.

**Build.** In /services/recovery, a module `collectAndRebuild` that takes shares re-wrapped to the coordinator's one-time key, checks each against getCommitments before use, runs Lagrange interpolation, and zeroizes everything afterwards. The HTTP, SSE and mode logic is M4.5.

**Done when.**

- [ ] A tampered share and a mismatched salt are rejected before reconstruction.
- [ ] k valid shares rebuild S.
- [ ] A share from an earlier epoch is rejected.
- [ ] Buffers are zeroized after use.

### M3.7 Seed script, Hospital B and interoperability

**Goal.** A repeatable demo world, and proof that a second institution can use the same tokens.

**Build.** seed-synthea-data.js loads the Synthea FHIR data and creates the demo identities (Dr. Rao, a lab, Hospital B, plus the responder, guardians, custodian and Coordinator the demo needs). It registers their public keys, has the scripted patient share a Claim permission with Dr. Rao, has Dr. Rao issue the demo claim, and writes the fixture. The Hospital B viewer on port 5174 uses its own identity and the same token: tokenIdsOfGrantee, KeysWrapped, fetch, decrypt (INT-1, INT-3).

**Done when.**

- [ ] The seed runs from a clean chain.
- [ ] The demo claim verifies on-chain.
- [ ] Hospital B reads a granted type with the same token Dr. Rao uses.
- [ ] Hospital B cannot read an ungranted type.

## Module 4: Yagna, break-glass, Tier 2, frontend and demo

- **Scope:** BreakGlassCapsule, the request half of RecoveryVault, the coordinator API and lifecycle, the frontend, the scripts and the final integration.
- **Folders:** /contracts (BreakGlassCapsule, RecoveryVault requests), /services/recovery (API), /frontend, /scripts, /docs/demo-script.md.
- **Consumes:** every other module.
- **Provides:** the emergency flow, the screens, `npm run demo:breakglass` and the integration test.
- **Stack:** Solidity, Node (HTTP and SSE), React 18, Vite, ethers 6, Jest.

### M4.1 Tier spec and frontend foundations

**Goal.** A written spec for the tiered scheme and a frontend that connects to the chain.

**Build.** A short design doc: Tier 1 is k-of-(n+1), Tier 2 is the institutional custodian plus a delay plus a cancel right. A React 18 and Vite app on port 5173 with ethers 6. SCR-01: wallet connect, network check for chain 31337, and routing by registration and role. Use mock providers until the contracts are live.

**Done when.**

- [ ] Rohith and Anush have reviewed the design doc.
- [ ] The app runs on 5173.
- [ ] SCR-01 shows no wallet, wrong network and connected states, and routes by registration and role.

### M4.2 BreakGlassCapsule

**Goal.** A time-boxed emergency window that cannot start without a trace.

**Build.** Backend Schema 3.9. The Capsule struct, preAuthorizeCapsule(fields), revokeCapsule and capsuleStatus. triggerEmergencyAccess(patientId) runs the role check, then the capsule check, makes the requestId, calls recoveryVault.setReleaseMode(requestId, patientId, requester, CapsuleOnly), sets windowExpiry and logs BreakGlassTriggered, all in one transaction. closeWindow is permissionless, works only after expiry and only once. getRequest reads the vault and combines it with windowExpiry and closed. Add the WINDOW\_DURATION constructor argument, and make deploy.js assert it is at least twice TIER2\_DELAY.

**Done when.**

- [ ] A responder without the role reverts and nothing changes.
- [ ] A patient who has not pre-authorized the capsule reverts.
- [ ] A success creates the request, the window and the event in one transaction.
- [ ] A reverting AuditLog reverts the trigger.
- [ ] closeWindow reverts before expiry, logs once after it, and reverts the second time.
- [ ] A lost-key requestId reverts as unknown in closeWindow.

### M4.3 RecoveryVault: request half

**Goal.** Requests, Tier 2 and cancellation, with the vault as source of truth.

**Build.** setReleaseMode is external with an onlyBreakGlassCapsule check, and setBreakGlassCapsule(address) is admin-only and callable once. startRecovery(patientId, clientKeyHash) is for the patient's account, rejects a zero hash and opens a FullRecord request. custodianCoSign(patientId, requestId) is for the InstitutionalCustodian only, starts TIER2\_DELAY and logs TierEscalated. cancelRecovery is for the patient or a registered guardian. completeRecovery is for the requester, once, not after a cancellation, and only for FullRecord. Add getRecovery and getRequestStatus.

**Done when.**

- [ ] setReleaseMode from any other address reverts, and a second setBreakGlassCapsule reverts.
- [ ] A zero key hash and a non-patient caller both revert in startRecovery.
- [ ] Cancel works for the patient and a guardian, and reverts for a stranger.
- [ ] completeRecovery on a CapsuleOnly request reverts.
- [ ] The Tier 2 delay is respected when chain time advances.

### M4.4 Emergency capsule module

**Goal.** The patient chooses, in advance, what a responder may see.

**Build.** The CapsulePayload fields: bloodType, allergies, currentMedications, emergencyContacts. The SCR-08 flow: tick fields, call preAuthorizeCapsule, have DataService encrypt under capsuleKey and register as EmergencyCapsule, and show the On status on the patient home. Revoke turns it off.

**Done when.**

- [ ] Only the ticked fields are in the payload.
- [ ] The on-chain field list equals the keccak256 of the field names.
- [ ] The version bumps on each pre-authorization.
- [ ] Revoking makes triggerEmergencyAccess revert.

### M4.5 Coordinator API and lifecycle

**Goal.** The HTTP and SSE service that runs a recovery or an emergency request end to end.

**Build.** The five endpoints and the progress event from Architecture section 13. Capsule and full modes. Full mode requires getRecovery to show FullRecord, the patient's account as requester, no cancellation or completion, and sha256(clientKemPublicKey) equal to the stored clientKeyHash. Never release after windowExpiry, which in full mode is the FULL\_MODE\_TTL deadline. Wait COLLECTION\_TIMEOUT before checking Tier 2. Require the bearer token. Wipe keys at close or expiry. Return 409 before release and 410 after close. Call collectAndRebuild from M3.6. Append signed audit records, with the trustees field, to the patient's JSONL file, signed by the Coordinator identity.

**Done when.**

- [ ] Shares not wrapped to the one-time key are rejected.
- [ ] Events arrive in order.
- [ ] /result returns 409 before release and 410 after close.
- [ ] Nothing is released after windowExpiry, and full mode refuses a wrong key hash.
- [ ] A signed audit record is appended and its signature verifies.

### M4.6 Patient screens

**Goal.** A patient can set up, share, revoke and review without touching crypto or gas.

**Build.** SCR-02 create or unlock the vault, SCR-03 home, SCR-04 My records, SCR-05 Share, SCR-06 Who has access, SCR-07 Guardians (through the dealer launcher) and SCR-09 Activity (events plus signed records). Wire in Rohith's keystore. Keys exist in memory after unlock, never in storage.

**Done when.**

- [ ] A new patient creates a vault, adds a record and shares it.
- [ ] Revoking removes access in the UI.
- [ ] Guardian setup warns on a single category, and shows the contract's rejection in plain words.
- [ ] Activity shows plain sentences, such as which guardians responded.

### M4.7 Responder, recovery and clinician screens

**Goal.** The emergency and recovery flows work from the screens.

**Build.** SCR-11 Emergency access: scan or enter the patient id from `medichain://patient/<patientId as 0x hex>`, the attestation tick, the trigger, the countdown and the capsule. SCR-10 Recover my access in full mode. SCR-12 Shared with me. The alert on the patient home with a Cancel request button that calls cancelRecovery.

**Done when.**

- [ ] A user without the responder role is refused in plain words.
- [ ] The countdown follows chain time.
- [ ] The capsule shows only the pre-authorized fields.
- [ ] Recovery on a fresh keystore restores the keys, and an old record decrypts.
- [ ] Cancel in the alert stops Tier 2.

### M4.8 Demo orchestration and integration

**Goal.** One command reproduces the demo, and the four modules work together.

**Build.** The scripts SCR-S1 (guardian responses), SCR-S2 (custodian co-sign) and SCR-S3 (demo runner). `npm run demo:breakglass` orchestrates deploy, seed and the scripted break-glass trigger. deploy.js refuses a configuration where WINDOW\_DURATION is under twice TIER2\_DELAY. Write docs/demo-script.md for the viva, with break-glass as the main moment. Then an end-to-end test across all four modules and bug fixing.

**Done when.**

- [ ] One command runs deploy, seed and break-glass from a clean chain.
- [ ] The end-to-end test passes for Tier 1, Tier 2 and a cancel.
- [ ] The recovery-on-a-new-device test passes.
- [ ] The two-hospital flow passes.
- [ ] The demo script has been rehearsed.

## Decision log

Add a row whenever a chapter forces a choice the spec does not make.

| Date | Module | Decision or open item | Status |
| --- | --- | --- | --- |
| 12 Oct 2026 | M3, M4 | Split of RecoveryVault and the RecoveryCoordinator between Rohith and Yagna (section 2, Shared files) | Proposed, both owners to confirm |
| 12 Oct 2026 | M1, M4 | scripts/deploy.js: Anush creates it, every owner adds lines, Yagna owns the run order | Proposed, to confirm |
| 12 Oct 2026 | M3, M4 | Device keystore: Rohith builds the library, Yagna wires it into the screens | Proposed, to confirm |
