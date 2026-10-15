# MediChain: Product Requirements Document (v1.6, revised draft)

Oct 13, 2026 · @Abc

## Overview

MediChain is a prototype health-record platform in which the patient, not the hospital, holds the keys. Records become FHIR-style resources, are encrypted, and are stored off-chain on IPFS. The blockchain holds only what it must: content hashes, consent state, proof-verification results, and an audit trail.

| Item | Value |
| --- | --- |
| Version | 1.6, revised draft. Decisions D1 to D23 are accepted |
| Date | 13 October 2026 (v1.5 was dated 3 October 2026) |
| Team | Group 02, Department of CSE, BGS College of Engineering and Technology, Bengaluru |
| Course | Project Phase-I (BCS685). Confirm this label is still the right one for the current semester |
| Authors | Anush G, Punith A, Rohith Gowda R, Yagna Maurya M V |

> The goal is to make exceptional access possible without making it invisible. (From the project paper, security discussion)

Doctors, labs, insurers, and devices receive role-scoped access that the contracts limit in time and let the patient revoke. A key already delivered to a grantee cannot be recalled (limitation 6). Zero-knowledge proofs let a patient prove a claim, such as age eligibility, without revealing the record behind it.

Two situations that most prototypes ignore are treated as first-class here: a patient who loses their private key, and a patient who cannot give consent because of an emergency. Guardians and an institutional custodian hold encrypted shares of one recovery secret. A responder can trigger a time-boxed break-glass access that releases only a pre-authorized emergency capsule and is always audited. To the team's knowledge from its survey of ten papers, no surveyed system implements break-glass access, which is why it is the centerpiece of the demo.

### What the project delivers

- Four integrated modules on a local Hardhat network: patient and consent contracts, the proof layer with on-chain audit, the data layer with Tier 1 recovery, and Tier 2 emergency access with the frontend.
- A React interface with a patient dashboard, a responder view, and a clinician view.
- Two scripted demos: the break-glass scenario and a two-hospital FHIR-style interoperability scenario.
- An evaluation on Synthea-generated synthetic FHIR data, a threat model, a benchmark comparison against the surveyed papers, and the final report.

### How this document was built

The four final module plans (Anush, Punith, Rohith, Yagna) are the source of truth for scope. The project deck, the IEEE-style paper, the abstract page, and the Tier 1 recovery diagram gave context and goals. Where those sources disagree with the module plans, this PRD follows the plans and lists the difference under Scope versus deck and paper claims.

Every requirement is accepted. Items marked "stretch" are optional. Requirement tags show the version that last changed them. No calendar dates were available, so the delivery plan is expressed in phases. Section Delivery plan now lists the dates the team still has to set.

## Problem and context

Health records are centralized, fragmented across hospitals and laboratories, and attractive targets for ransomware and insider misuse. The HHS Office for Civil Rights reports that breaches of unsecured health information that occurred in 2023 affected over 100 million individuals: about 113 million across 732 breaches of 500 or more records. Hacking and IT incidents caused 81% of those breaches and 96% of the individuals affected.

Source: [HHS OCR, Annual Report to Congress on Breaches of Unsecured Protected Health Information, Calendar Year 2023](https://www.hhs.gov/sites/default/files/breach-report-to-congress-2023.pdf). OCR notes the counts are approximate because some entities reported uncertain numbers. Totals quoted from the live portal vary slightly by snapshot date, so this PRD uses OCR's published annual figure. The roughly 41.7 million figure is OCR's 2022 count, not 2023.

Blockchain improves tamper resistance, but it does not protect privacy on its own: data, metadata, or access logic exposed on-chain can still leak protected health information.

Existing platforms such as MediBloc and PolyMed use an all-or-nothing key model. If a patient loses their private key, the records are locked away for good. Most also skip FHIR and HL7, so they become silos that cannot be shared across hospital networks.

The team's survey of ten papers found no single system that combines cryptographic privacy, granular time-bound consent, emergency access, key recovery, real-time proofs, post-quantum resilience, interoperability, patient-friendly access, and compliance support. Section Survey gap coverage maps each of those gaps to requirements in this document.

## Goals, non-goals, and success metrics

The prototype aims at eight outcomes and rules out sixteen things, so the report's claims stay honest.

### Goals

- **Privacy by construction.** No protected health information on-chain; verification through proofs rather than disclosure.
- **Granular consent.** Time-limited, revocable, role-scoped consent tokens that the patient controls. The contracts enforce expiry and revocation. A key already delivered cannot be recalled.
- **Recoverability.** Recovery from key loss through guardian-held shares (Tier 1), with an institutional custodian share as fallback when exactly one guardian share is missing (Tier 2), all from one k-of-(n+1) scheme.
- **Accountable emergency access.** Break-glass access that is role-gated, time-boxed, limited to a pre-authorized capsule, and impossible to run without an audit event.
- **Interoperability.** A FHIR R4-inspired data model that a second institution can consume with the same consent tokens.
- **Long-term integrity.** Post-quantum signatures on records and events, and post-quantum encryption of guardian shares.
- **Measured performance.** Proof latency and gas cost benchmarked and compared with figures reported in the surveyed papers.
- **Usable by non-technical patients.** Gasless interaction and plain-language flows.

### Non-goals

These are out of scope for the prototype. Several are named in the deck or paper, so they are listed explicitly to keep the claims honest.

- Production or public-network deployment. Everything runs on Hardhat's built-in local network.
- Full rollup infrastructure. The prototype does proof batching only, with no aggregation.
- Any bridging between chains. Portability is demonstrated across institutions on a single chain.
- Proof systems other than zk-SNARKs. Only circom and snarkjs (Groth16) circuits are planned.
- Post-quantum zero-knowledge proofs, or post-quantum Ethereum transaction signatures. Post-quantum coverage is signatures and key encapsulation only.
- Legal compliance certification (HIPAA, GDPR, India's DPDP Act). The system supports audit, revocation, and minimal disclosure but makes no compliance claim.
- Bundler and paymaster infrastructure. Account abstraction is simplified and ERC-4337-style, with gas sponsorship simulated on the local network.
- A guardian-facing UI screen. Guardian responses in the demo are scripted.
- Push, email, or SMS notification. The patient sees a request on the dashboard or the Activity screen only (BG-7, limitation 19).
- Real clinical datasets in the current plan. Evaluation uses Synthea synthetic FHIR data. MIMIC-III is a stretch goal only (DAT-7).
- A native mobile app. The frontend is a responsive React web app.
- Recovering a lost wallet or account. Lost-key recovery restores access to records and the patient's keys, not control of the wallet that was lost.
- A screen for issuing claims. Issuing a claim, and the patient's permission for the issuer, are scripted in the demo.
- Onboarding real hospitals or custodians, clinical safety validation, or any real patient data.
- Feldman or Pedersen verifiable secret sharing. Hash commitments are used instead, on purpose, to avoid reintroducing a discrete-log assumption.
- Claiming FHIR conformance. The data model is R4-inspired and is not validated against the FHIR specification (DAT-1).

### Success metrics

| Metric | Target | How it is measured |
| --- | --- | --- |
| No PHI on-chain | No record content in contract storage or events after the end-to-end run | Automated scan of chain state and events |
| Consent enforcement | 100% of expired, revoked, and unauthorized requests rejected | Hardhat unit tests for CON-2 to CON-4 |
| Break-glass centerpiece | Role check, policy check, time-boxed window, audit event, capsule-only release, and expiry demonstrated end to end | Automated integration test and live demo |
| Recovery correctness | Every combination of at least k of the n+1 shares reconstructs the secret; k-1 never does; tampered shares rejected before reconstruction | Jest suite over all share combinations |
| Integrity | Any tampering with a stored record, or signing with the wrong party's key, is detected on read | Dilithium verification and AES-GCM tag tests |
| Proof performance | Proof generation in the sub-second range (deck target, not a result); gas per verification and per batch reported | Punith's benchmark harness |
| Literature comparison | A table covering all ten surveyed papers, marked "not reported" where a paper gives no figure | Benchmark writeup |
| Evaluation data | Functional tests, demos, and benchmarks run on a Synthea-generated synthetic FHIR R4 dataset, with the limitation stated in the report | DAT-6 |
| Interoperability | A second-hospital endpoint accepts the same consent token unchanged | INT-1 demo |
| Gap coverage | All ten survey gaps mapped to requirements with an honest status | Gap matrix and the threat model (SEC-1) |

## Users and stakeholders

The patient is the primary user; seven other personas act on the patient's behalf or consume what the patient shares.

| Persona | Who they are | What they need from the system |
| --- | --- | --- |
| Patient (primary) | A non-technical person who owns the records and the keys | Share selectively, stop future access at any time, recover from a lost key, pre-authorize emergency access, and see who accessed what |
| Treating clinician | A doctor holding a granted role | Request access, prove eligibility, and read only what the token allows. Signs the records they create |
| Lab, insurer, IoT device | Non-clinician roles under attribute-based access control | Narrow, time-limited access to specific resource types; verify a claim without seeing the full history. Labs sign the records they create |
| Emergency responder | An ER physician or similar role | The critical fields fast when the patient cannot consent, with access that is time-limited, audited, and limited to the capsule |
| Guardian (trustee) | A family member, hospital custodian, or neutral guardian service | Hold an encrypted share safely and release it only for a legitimate request |
| Institutional custodian | A hospital or institution acting in Tier 2 | Hold the escrowed share as trustee n+1, and release it only after co-signing and the delay window |
| Second institution | Another hospital's system | Consume the same consent tokens and FHIR-style data without a custom integration |
| Evaluators | Guide, coordinators, and viva panel | A working break-glass scenario, a threat model, and measured results |

The deck names three target applications: emergency departments getting patient summaries through time-locked triggers, pharmacies verifying prescriptions through a proof without viewing the full history, and hospital networks sharing records across FHIR-compliant systems. This prototype is FHIR R4-inspired and makes no conformance or compliance claim.

## System scope and architecture

The system is nine layers on one local Hardhat network, with state on-chain, computation off-chain, and only encrypted bytes in storage.

### Layers and ownership

| Layer | What it does | Built with | Owner |
| --- | --- | --- | --- |
| Patient | Registration, wallet linkage, simplified DID/SSI identity, each party's published Dilithium public key, ERC-4337-style account abstraction. A `PatientAccountFactory` creates one account per owner wallet and resolves wallet to account (D10) | Solidity, Hardhat, ethers.js | Anush |
| Consent and roles | Issue, revoke, and expire time-limited consent tokens. A minimal ABAC role registry ships early in Core build, and full ABAC wiring for doctor, lab, insurer, and IoT device follows in Advanced | Solidity | Anush |
| Proof and audit | circom circuits (an issued-claim circuit first), on-chain verifier and proof gateway with claim commitments, batched verification, audit events, benchmarks (updated in v1.4) | circom, snarkjs, circomlibjs, Node.js | Punith |
| Data and integrity | FHIR-style schema, AES-GCM encryption, IPFS storage, creator-signed Dilithium signatures, type-key wrapping to grantees, keys for clinicians, labs, and the second institution, Synthea evaluation data. Crypto runs in a client library; a small IPFS proxy holds the Pinata credentials and sees only ciphertext (D12) | Node.js, Pinata, @noble/post-quantum | Rohith |
| Record registry | On-chain record references listed by patient and resource type, with the CID stored as a string (DAT-8) | Solidity | Anush |
| Recovery Tier 1 | Hardened k-of-(n+1) Shamir recovery: one Setup phase deals shares to n guardians and the institutional custodian, with commitments, ML-KEM transport, and proactive re-sharing | secrets.js-grempe, @noble/post-quantum | Rohith |
| Emergency and Tier 2 | Break-glass contract, emergency consent capsule, and custodian co-sign with a delay before the custodian's share is released | Solidity | Yagna |
| Frontend | Patient dashboard, records list, responder view, demo clinician view, Hospital B viewer, scripted guardian responses, demo identities, demo orchestration, integration testing (updated in v1.4) | React, ethers.js | Yagna |
| Security analysis | Threat model: what each layer exposes and protects, mapped to the survey gaps | Documentation | Anush |

### What lives where

| Location | Contents | Never contains |
| --- | --- | --- |
| On-chain (local Hardhat network) | Patient identity and wallet link, each party's Dilithium and ML-KEM public keys, ABAC roles, consent tokens (grantee, scope, expiry, revoked flag), record references (CID string and content hash), claim commitments with the issuer's address, verifier results, break-glass windows, audit events, and the hash commitments for all n+1 shares | Record content, plaintext keys, shares, salts, claim values, or any PHI |
| Off-chain, encrypted (IPFS via Pinata) | AES-GCM ciphertext of FHIR-style resources with the creator's Dilithium signature | Plaintext records |
| Trustee custody | One share bundle (the share plus its salt) per trustee, including the custodian, encrypted to that trustee's ML-KEM public key | A plaintext share or salt in transit or at rest |
| Client and Node.js processes | Proof generation, Dilithium signing and verification, type-key wrapping, the short-lived dealer, and decryption at the point of use | Anything persisted in plaintext. The dealer overwrites its buffers where the runtime allows; JavaScript cannot guarantee it (limitation 8) |

### Tier 1 recovery architecture

This is the refined design from Rohith's module, organized by lifecycle stage and extended for the accepted decisions. The emergency path is latency-critical, so the policy check comes first and cryptography runs only after it passes.

**Setup (once, offline)**

1. **Client creates S, dealer splits it.** The patient's client creates S at vault creation with a secure random generator, never `Math.random`. The dealer receives it wrapped, in a minimal, short-lived context, ideally never on the network.
2. **Split and commit.** One Shamir split into n+1 shares, n for guardians and one for the institutional custodian, with threshold k. A commitment H(share\_i || salt\_i) per share is published on-chain.
3. **Encrypted distribution.** Each share and its salt are bundled and encrypted to the trustee's ML-KEM public key. The dealer then overwrites what it can and terminates (best effort, limitation 8).

**Custody (steady state)**

4. **Trustees hold shares.** Family, hospital custodian, and neutral guardian service, plus the institutional custodian as trustee n+1, whose share is released only after co-sign and delay. A breached channel yields ciphertext, not shares. No single trust category may hold k or more of the n+1 shares (T1-5).
5. **Proactive re-sharing.** The patient starts it, prompted by a Refresh due reminder. It regenerates shares for the same S, so a slow attacker never accumulates k valid shares of one generation.

**Recovery (emergency path, latency-critical)**

6. **Consent capsule trigger.** Validates who is asking and whether this is a recognized emergency context, before any cryptography runs.
7. **Quorum and verification.** Each submitted share is checked against its on-chain commitment, using the salt from its bundle. Bad shares are rejected here, not after they corrupt reconstruction.
8. **Reconstruct and release.** Lagrange interpolation recovers S. A break-glass request decrypts and returns only the emergency capsule. The patient's own lost-key flow uses S to restore access to the full record. A Dilithium-signed audit event is logged.

Tier 2 is part of the same scheme, not a separate process. The institutional custodian is trustee n+1 in a k-of-(n+1) scheme, dealt in the same Setup phase as the guardian shares. Its share is released only after the custodian co-signs and the delay window elapses, and it is verified against its commitment like any other share.

Because the custodian holds a single share, Tier 2 lets recovery succeed only when exactly one guardian share is missing: k-1 guardians plus the custodian. It never lets the custodian act alone, since k is at least 2. With k = 2, the custodian plus one guardian is enough, so choose k with this in mind (limitation 12).

## Functional requirements

There are 67 requirements across nine modules, 64 from v1.5 and three new. P0 is needed for the demo and viva, P1 is planned and should ship, and P2 is a stretch. Priorities are this PRD's proposal, derived from each module's phase order. Rows marked v1.6 changed in this revision.

### Patient identity and access control

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| PAT-1 | Patients can register, and registration creates an on-chain identity linked to their wallet. | P0 | Anush |
| PAT-2 | Simplified DID/SSI identity: the patient controls their own identifier and the credentials attached to it. | P0 | Anush |
| PAT-3 | Account-abstraction wallet so patients never handle gas or raw transactions. Simplified and ERC-4337-style, with sponsored gas simulated on the local network. | P1 | Anush |
| PAT-4 | ABAC role registry covering doctor, lab, insurer, and IoT device, plus an emergency-responder role for break-glass. A minimal registry (role IDs and role assignment, enough to define the emergency-responder role) ships early in Core build so Yagna can build BG-1 against it. Full ABAC wiring into consent logic follows in the Advanced phase (updated in v1.1). | P0 | Anush |
| PAT-5 | The identity contract publishes each party's Dilithium public key alongside their wallet and DID link, so any reader can verify a signature without contacting the signer. Parties include patients, doctors, and labs. | P1 | Anush |
| PAT-6 | The identity contract also publishes each party's ML-KEM public key, which type-key wrapping (DAT-5) and share encryption (T1-4) both need (accepted in v1.2). | P1 | Anush |
| PAT-7 | **New in v1.6 (D10).** A `PatientAccountFactory` deploys one `PatientAccount` per owner wallet, at an address that can be computed before deployment, and resolves wallet to account. On first connection the relayer creates the account, then the patient registers through it. The app resolves the patient's account from the wallet before it looks up the identity. Doctors, labs, and Hospital B are plain accounts and register directly. | P0 | Anush |
| PAT-8 | **New in v1.6 (D15).** `PatientIdentity.register` accepts only a DID hash equal to keccak256 of the string `did:medichain:` followed by the caller's own lowercase hex address. Nobody can register another party's identifier first. | P1 | Anush |

### Consent

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| CON-1 | A patient can issue a consent token to a grantee, scoped to specific resource types, with an expiry time. | P0 | Anush |
| CON-2 | A patient can revoke any token at any time. Revoked tokens fail every later access check. Revocation does not recall a key already delivered (DAT-5). (v1.6) | P0 | Anush |
| CON-3 | Tokens expire automatically. Expired tokens are rejected with no action from the patient. | P0 | Anush |
| CON-4 | Access checks combine token validity with role attributes. Unauthorized requests are rejected and logged. A free view call (`canAccess`) answers interface questions. The logged call (`checkAccess`) is a transaction that emits AccessGranted or AccessDenied, never reverts on a denial, and returns its outcome in the event. Clients that need a logged check use a funded signer. (v1.6, D13) | P0 | Anush |
| CON-5 | The consent contract exposes a proof-verification interface, first as a stub and then wired to the real verifier (ZKP-3). | P0 | Anush |
| CON-6 | The token format can be verified by an external institution's endpoint without modification (needed for INT-1). | P1 | Anush |

### Proof layer

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| ZKP-1 | A working circom and snarkjs toolchain, proven with a hello-world circuit such as proving x is above a threshold. | P0 | Punith |
| ZKP-2 | The first real circuit proves that a claim value issued by a Doctor or Lab meets a threshold. The prover keeps the value and a salt private. The circuit checks that Poseidon(value, salt) equals the issued commitment, that the value is at least the public threshold, and it binds a public nonce. Proving and verification keys are generated (updated in v1.4). The circuit range-checks the value and the threshold to 8 bits explicitly, and the gateway rejects a required threshold of 256 or more. (D23) | P0 | Punith |
| ZKP-3 | A Solidity verifier exported from snarkjs and connected on-chain through ProofGateway. The gateway accepts a proof only if its commitment equals the one registered for that patient and claim type, and only if its threshold is the one the verifier requires (updated in v1.4). A verifier's request names the claim type, the threshold, and the request id. On success the gateway emits an EligibilityVerified event that carries them with the verifier's address, so the verifier reads the result from the receipt and not from a return value. (D19) | P0 | Punith |
| ZKP-4 | Batched verification: several proofs verified in one transaction, each with its own commitment check and its own nonce. This is the project's substitute for full ZK-Rollup. It saves transactions, not per-proof cost, because nothing is aggregated. (v1.6) | P1 | Punith |
| ZKP-5 | Benchmarks of proof-generation latency, verification latency, and gas cost, compared against figures reported in the ten surveyed papers where available. Benchmarks that involve record data use the Synthea-based dataset (DAT-6). | P0 | Punith |
| ZKP-6 | Stretch: a diagnostic-threshold circuit, added only if the age or eligibility circuit and the core integration are complete. | P2 | Punith |
| ZKP-7 | Issued-claim commitment and replay protection. An account holding the Doctor or Lab role, and holding an active consent token from the patient that names it for the Claim resource type, registers a commitment for a patient and claim type through `ProofGateway.setCommitment`. The issuer's address is stored and emitted, and issuing is logged. Only the original issuer may replace a commitment, and the patient or the issuer can revoke it through `revokeClaim`, which is also logged. The patient receives the claim's value and salt inside the encrypted Patient record the issuer creates. A proof verifies only against the stored commitment, and each nonce, derived from the chain id, the ProofGateway address, and the request or token id, can be used once. The claim is as of issuance, and the verifier trusts the issuer. (v1.6) | P0 | Punith |

### Audit

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| AUD-1 | Every proof check, consent action, access request, break-glass trigger, and recovery event emits an on-chain audit event with an agreed schema. | P0 | Punith |
| AUD-2 | Break-glass cannot succeed without emitting its audit event. This is enforced in the contract, not by convention. | P0 | Yagna |
| AUD-3 | Recovery and access events are also Dilithium-signed and logged, recording who requested, which trustees responded, and when. | P1 | Rohith |

### Data layer and integrity

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| DAT-1 | A FHIR R4-inspired schema covering Patient and Observation resources. Consent appears only as a read-only projection of the on-chain token and is never stored. Ids match the FHIR id pattern (letters, digits, hyphen, dot, at most 64 characters). A party's DID is carried as an `identifier`, not inside an id. An issued claim is a valid complex extension. "Claim" is MediChain's permission category and has nothing to do with the FHIR billing resource of that name. The model is not validated against the FHIR specification unless INT-2 is done. (v1.6, D16) | P0 | Rohith |
| DAT-2 | Records are AES-GCM encrypted before upload, stored on IPFS through Pinata, and their hash is recorded on-chain. | P0 | Rohith |
| DAT-3 | An authorized retrieval path: fetch, verify the creator's signature, decrypt with the record key derived from the type key delivered under DAT-5, and return FHIR-style data. | P0 | Rohith |
| DAT-4 | Dilithium (ML-DSA) signature wrapping. The record's creator signs on write: the patient for self-uploaded records, a doctor or lab for records they create. Readers verify on read using the creator's public key from the identity contract (PAT-5). A tampered record, or one signed by the wrong party, must fail verification (updated in v1.4). | P1 | Rohith |
| DAT-5 | At consent time the patient's client wraps one type key per resource type in the token to the grantee's ML-KEM public key, and the wrapped keys reach the grantee with the consent token. A record's own key is derived from its type key, so the grantee can read records of the shared types, including ones added later. Revocation and expiry block later contract-mediated access, but a key already delivered cannot be recalled, and that includes later records of the same types. Versioned type keys that would limit this are a stretch (limitation 6). (updated in v1.2) | P0 | Rohith |
| DAT-6 | Synthea-generated synthetic patient data, exported as FHIR R4, is the primary dataset for functional tests, demos, and benchmarks. The seed script gives each resource a valid FHIR id, rewrites `urn:uuid:` references to `Patient/<id>` form, and adds the DID identifier. The report states plainly that the data is synthetic. (v1.6) | P0 | Rohith, Punith |
| DAT-7 | Stretch: also evaluate on MIMIC-III, only if credentialing clears with time to spare. No core requirement depends on it. | P2 | Rohith, Punith |
| DAT-8 | A record registry (`RecordRegistry`): only the patient's account can add a record, and records are listed by patient and resource type. The CID is stored as a string, and the emergency capsule registers under its own resource type. | P0 | Anush |
| DAT-9 | Keys for parties other than the patient. Each clinician, lab, and second-institution endpoint holds its own keystore, with keys derived by HKDF from its own secret, and registers its public keys in the identity contract like any party. In the demo the seed script creates the demo identities as Hardhat's unlocked accounts and stores each party's secret and its account in a git-ignored, demo-only fixture file, with no private keys. As for patients, keys cannot rotate without a new secret. (v1.6) | P0 | Rohith |
| DAT-10 | **New in v1.6 (D12).** Record encryption, signing, key wrapping, and decryption run in a client library that works in the browser and in Node scripts. A loopback IPFS proxy holds `PINATA_JWT` and `PINATA_GATEWAY`, offers pin and fetch routes, and only ever receives ciphertext envelopes. No key or plaintext reaches a service. | P0 | Rohith |

### Tier 1 recovery

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| T1-1 | The patient's client creates the secret S when the vault is created, using a cryptographically secure random generator (`crypto.getRandomValues` in the browser, `crypto.randomBytes` in Node) and never `Math.random`. At guardian setup the dealer receives S wrapped to its one-time key and runs in a minimal, short-lived context, ideally offline, using the same kind of generator for the split's random values. (updated in v1.4) | P0 | Rohith |
| T1-2 | In a single Setup run, S is split with Shamir's scheme into n+1 shares with threshold k: n shares for guardians and one for the institutional custodian. There is no separate key generation for Tier 2. k is at least 2 and at most n+1, so no single trustee can reconstruct alone. A hash commitment H(share\_i \|\| salt\_i) is computed for each share and published on-chain. Feldman VSS is deliberately not used. (updated in v1.4) | P0 | Rohith |
| T1-3 | The dealer overwrites its buffers where the runtime allows and terminates immediately after distribution. JavaScript strings are immutable and the garbage collector may keep copies, so this is best effort. Ending the process is the real mitigation (limitation 8). (v1.6) | P0 | Rohith |
| T1-4 | Each share bundle (the share plus its salt) is encrypted to its trustee's ML-KEM public key before it leaves the dealer. This includes the custodian's bundle. (updated in v1.1) | P0 | Rohith |
| T1-5 | Guardians span mixed trust categories: family, hospital custodian, and neutral guardian service. The contract rejects a set in which every guardian shares one category, and (D14) a set in which any single category holds k or more of the n+1 trustees, because k trustees of one kind could rebuild S. Guardian setup warns before the transaction is sent. (v1.6) | P1 | Rohith |
| T1-6 | Proactive re-sharing, started by the patient, regenerates shares for the same secret across all n+1 trustees. A share from an earlier generation no longer matches any current commitment, so the coordinator rejects it. The app shows Refresh due once the last refresh is older than a policy interval the team sets. The contract does not enforce a cadence, and an attacker who already holds k old shares can still rebuild S offline. (v1.6) | P1 | Rohith |
| T1-7 | Each submitted share is verified against its on-chain commitment, using the salt from its bundle, before use. Tampered shares are rejected before reconstruction. (updated in v1.4) | P0 | Rohith |
| T1-8 | Lagrange interpolation reconstructs S from at least k valid shares out of the n+1. Fewer than k reveal nothing. | P0 | Rohith |
| T1-9 | The consent-capsule policy check (who is asking, and is this a recognized emergency context) runs before any cryptographic operation. | P0 | Rohith |
| T1-10 | The recovery module has two release modes, fixed on-chain before any share moves. Break-glass: the coordinator decrypts and returns the emergency capsule only. Lost-key: the patient's account opens the request with a hash of a one-time key from the new device, the coordinator returns the rebuilt secret S wrapped to that key and never reads a record, and the device then re-derives its keys and decrypts records itself. The coordinator refuses to wrap S unless the key it was given matches the stored hash. Capsule mode has its own checks: the requester proves control of the on-chain requester address, the on-chain request is CapsuleOnly with its window open, and the coordinator reads the capsule record only through the read-only IPFS route and releases only the fields the patient pre-authorized. (D20) | P0 | Rohith |

### Break-glass and Tier 2 emergency access

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| BG-1 | An emergency-responder role defined through the ABAC scheme, built against the minimal role registry that PAT-4 ships early. (updated in v1.2) | P0 | Yagna |
| BG-2 | `triggerEmergencyAccess()` opens a time-boxed access window. Access ends when the window expires: the coordinator stops releasing and wipes its keys, the screen clears, and `closeWindow` logs the expiry on-chain. (updated in v1.6) | P0 | Yagna |
| BG-3 | Every break-glass call emits a mandatory audit event (AUD-2). | P0 | Yagna |
| BG-4 | Emergency consent capsule: a minimal-disclosure structure holding critical fields only, pre-authorized by the patient in advance. It is the only data a break-glass request releases. Full-record decryption stays with the patient's own lost-key recovery flow (T1-10). Saving emergency info registers the encrypted capsule first and pre-authorizes it second, so a failure leaves a capsule that nobody can release. (updated in v1.2, D20) | P0 | Yagna |
| BG-5 | The capsule policy check gates every break-glass request before any cryptography runs (shared with T1-9). | P0 | Yagna |
| BG-6 | Tier 2 fallback: the institutional custodian, trustee n+1 of the same k-of-(n+1) scheme, co-signs, a delay window elapses, and only then is the custodian's escrowed share released. It applies only when exactly one guardian share is missing. The share passes the same commitment check as any other and enters reconstruction only as a verified share, never raw. The custodian's co-sign reverts for an unknown, cancelled, completed, or already co-signed request, so a second call cannot reset the delay. If the k-th valid guardian share arrives during the delay, the secret is rebuilt at once and the delay is dropped. (updated in v1.6, D21 and D23) | P0 | Yagna |
| BG-7 | During the Tier 2 delay window the request is visible to the patient (dashboard alert and Activity screen) and to the scripted guardians, and the patient or any registered guardian can cancel it. There is no push, email, or SMS notification (limitation 19). (v1.6) | P1 | Yagna |
| BG-8 | A demo script that runs the break-glass scenario end to end as the main viva moment, with scripted guardian responses (UI-7). (updated in v1.2) | P0 | Yagna |

### Frontend

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| UI-1 | Patient dashboard: issue, view, and revoke consent tokens. | P0 | Yagna |
| UI-2 | Guardian setup: choose n and k and add trustees, including the institutional custodian as trustee n+1. The screen warns when one trust category would hold k or more trustees (T1-5). (v1.6) | P0 | Yagna |
| UI-3 | Emergency capsule: the patient reviews and pre-authorizes what a responder would see. | P0 | Yagna |
| UI-4 | Responder view: request access, trigger break-glass, and see the time remaining in the window. | P0 | Yagna |
| UI-5 | Audit trail on the patient dashboard: who accessed what, when, and which guardians responded. | P1 | Yagna |
| UI-6 | Plain language throughout. Gas, raw keys, and hashes are never shown to patients. | P1 | Yagna |
| UI-7 | Guardian responses in the demo are scripted, driving the real share-decryption and submission path. No guardian screen is built. (updated in v1.2) | P0 | Yagna |
| UI-8 | My records: a read-only list of the patient's records, with the creator and a Verified badge once the signature checks out. | P0 | Yagna |
| UI-9 | Shared with me: a clinician's list of permissions where they are the grantee, a readable record view with creator verification, and proof-based shares shown as verified facts with no data. It opens after the clinician's keystore is unlocked or a demo identity is loaded. (new in v1.5) | P0 | Yagna |

### Interoperability, security analysis, and integration

| ID | Requirement | Priority | Owner |
| --- | --- | --- | --- |
| INT-1 | Two-hospital demo: a second institution's endpoint consumes the same consent tokens and retrieves FHIR-style data, proving cross-system portability on one chain. | P0 | Rohith |
| INT-2 | Stretch: validate the FHIR-style resources against an official FHIR validator. This needs the identifier, extension, and id fixes in DAT-1. | P2 | Rohith |
| INT-3 | Hospital B viewer: a second small read-only app on its own port that takes a patient ID and a permission, finds the records through the registry, and shows the same record the first hospital sees. It uses a demo identity. It calls only the IPFS proxy's read-only fetch route and no other local service. (D18) | P0 | Rohith |
| SEC-1 | A threat-model document covering what each layer exposes versus protects, mapped against the survey's gap table. | P0 | Anush |
| SEC-2 | State the post-quantum scope explicitly: which components are protected (record and event signatures, share transport) and which are not (the SNARK proof system and Ethereum transaction signatures). (updated in v1.4) | P1 | Anush |
| E2E-1 | An end-to-end integration test across all four modules, with bug fixing. | P0 | Yagna |

## Key user flows

Four flows carry the product: routine sharing, break-glass access (the centerpiece), lost-key recovery, and cross-hospital access.

### A. Routine sharing

1. The patient connects a wallet. The account factory creates the patient's account (the relayer pays), and the patient registers through it. The contract links a DID-style identity and the Dilithium public key to that account (PAT-7).
2. The client converts a record to FHIR-style resources, encrypts it with AES-GCM, and signs it with the creator's Dilithium key: the patient for their own uploads, a doctor or lab for records they create. It uploads to IPFS through the proxy, and the hash goes on-chain.
3. The patient issues a consent token to a doctor with a scope, a role, and an expiry. At the same moment the client wraps a key for each shared record type to the doctor's ML-KEM public key.
4. The doctor requests access. For a proof-based share, such as "over 18", the proof is built from a claim a doctor or lab issued for this patient. It is verified on-chain, singly or in a batch, only against the commitment the issuer registered, and each request can be proved once. The logged access check verifies token, role, and expiry, and emits an audit event.
5. The record is fetched, the creator's signature is verified against the public key in the identity contract, and the doctor decrypts it with the wrapped key.
6. The patient revokes the token. The next request is rejected and logged. A wrapped key already delivered cannot be recalled, and the threat model records that limit.

### B. Break-glass emergency access (the centerpiece)

Before the emergency, the patient has pre-authorized an emergency capsule and set up n guardians plus the institutional custodian with a chosen threshold k.

1. The patient arrives unconscious. An ER physician holding the emergency-responder role opens the responder view and triggers break-glass.
2. The capsule policy check runs first: who is asking, and is this a recognized emergency context. If it fails, no cryptography runs.
3. `triggerEmergencyAccess()` opens a time-boxed window and emits the mandatory audit event.
4. Tier 1: at least k guardians respond. In the demo their responses are scripted, but they run through the real share-decryption path. Each share is checked against its on-chain commitment, bad shares are rejected, and the secret is reconstructed.
5. If exactly k-1 guardians respond, Tier 2 applies: the institutional custodian co-signs, the delay window elapses, and only then is the custodian's share released and verified. If fewer than k-1 guardians respond, recovery fails.
6. The recovery module decrypts and returns the emergency capsule only, and the responder sees its critical fields. The full record stays locked. When the window closes, the coordinator stops releasing and the summary clears from the screen. Information already seen cannot be taken back.
7. Afterward, the patient sees on the dashboard who accessed what, when, and which guardians responded.

Show the failures in the demo, too. Three short beats make the safety story concrete: a user without the emergency role is rejected, a tampered guardian share is refused before reconstruction, and a request after the window expires is denied. A fourth beat shows the patient cancelling during the Tier 2 delay.

### C. Lost-key recovery

1. The patient connects their wallet on a new device, and the app finds their account. The patient's account opens the request on-chain, which records a hash of a one-time key the new device has just made.
2. The client sends that one-time key to the coordinator. Before it does anything, the coordinator checks the on-chain request and that the key matches the stored hash.
3. At least k of the n+1 trustees release their shares. Each is verified against its on-chain commitment, and the verified shares rebuild S.
4. The coordinator returns S to the new device, wrapped to the one-time key, and wipes it. The coordinator never reads a record in this flow.
5. The patient chooses a new passphrase. The device re-derives the patient's keys, which match the published ones, and decrypts records locally. Old signatures still verify.
6. The client marks recovery complete on-chain. Shares are then refreshed through proactive re-sharing so that shares exposed during recovery go stale.
7. If exactly one guardian share is missing, the Tier 2 custodian share can make it up, as in flow B.

### D. Cross-hospital access

1. A consent token is issued at the first hospital.
2. The second hospital's endpoint receives a request carrying the same token and verifies its validity and role without modifying it.
3. The endpoint fetches the FHIR-style resources from IPFS, verifies the creator's signature, and decrypts them with the wrapped key.
4. An audit event records the access.

Flows B and C share the same guardian and quorum machinery. They differ in the trigger, the requester, and what is released: the emergency capsule only, versus the secret that unlocks the patient's own records, returned only to their new device.

## Non-functional requirements

Thirteen areas cover privacy, performance, hygiene, integrity, recovery, audit, usability, interoperability, reproducibility, and two new rows for time handling and local-service security.

| Area | Requirement | Measure |
| --- | --- | --- |
| Privacy | No PHI or record content on-chain. The chain stores hashes, CIDs, proofs, consent metadata, public keys, share commitments, and events only. | Automated scan of contract storage and events after the end-to-end run |
| Proof performance | Proof generation in the sub-second range (deck target, not a result). Verification gas and per-proof batched gas are measured and reported. | Punith's benchmark harness and the comparison table |
| Emergency latency | The break-glass path adds no unnecessary steps between a valid request and the responder seeing the capsule. The deliberate Tier 2 delay is excluded. | End-to-end timing in the demo run. A numeric target is to be set after the first baseline measurement |
| Cryptographic hygiene | Secrets and shares come from a cryptographically secure generator only (`crypto.randomBytes` in Node, `crypto.getRandomValues` in the browser). AES-GCM nonces are never reused under one key. Keys and shares are overwritten after use where the runtime allows, which is best effort in JavaScript. (v1.6) | Code-review checklist and tests |
| Post-quantum posture | ML-DSA (Dilithium) signs records and events, and ML-KEM protects share transport and type-key wrapping, through `@noble/post-quantum`. The SNARK proof system and Ethereum transaction signatures remain classical and are documented as such. | SEC-2 in the threat model |
| Integrity | Every stored record is signed by its creator on write and verified on read, and tampering is detected. | DAT-4 tests |
| Recovery availability | Any k of the n+1 shares suffice and fewer than k learn nothing. The Tier 2 custodian share lets recovery succeed when exactly one guardian share is missing, so with k-1 guardians. (v1.6) | Tier 1 suite and the BG-6 test |
| Auditability | Every consent action, proof check, access request, break-glass trigger, and recovery event is logged. Break-glass cannot run silently. | AUD tests |
| Usability | Patients never see gas, raw keys, or hashes. Consent and guardian setup are plain-language flows. | Stretch: walkthrough with a few non-technical users |
| Interoperability | Resources follow the FHIR R4-inspired schema, and a second endpoint consumes the same tokens unchanged. | INT-1 |
| Reproducibility | The whole system runs on one developer machine (Core i5 10th generation or better, 8 GB or more of RAM, 20 GB SSD, per the deck), with documented setup and a one-command demo. (updated in v1.4) | Clean-machine setup test and the one-command demo |
| Time handling | **New in v1.6 (D11).** One clock. The local chain mines a block every second, so block time follows wall-clock time. Interface countdowns and coordinator deadlines read the latest block time. The demo never advances time. Tests that do advance time run on a separate in-process network. | A countdown test and a check that no demo script calls a time-advance method |
| Local-service security | **New in v1.6 (D17).** The coordinator, dealer launcher, and IPFS proxy bind to 127.0.0.1, require a per-session bearer token, and allow only the frontend's origin through CORS. The browser reads the event stream with a fetch stream that sends the token in a header, because `EventSource` cannot send headers. Three tokens apply: a session token that starts requests and calls the dealer and the proxy, a per-request token that guards events, result, close, and shares, and shares that must also be wrapped to the request's one-time key. (D22) | Integration tests with a missing token and a wrong origin |

## Technology stack

The stack is Solidity on Hardhat, circom and snarkjs for proofs, and React with ethers.js, plus Node.js loopback services for recovery and IPFS access.

| Area | Choice | Notes |
| --- | --- | --- |
| Smart contracts | Solidity 0.8, Hardhat, ethers.js | Hardhat's built-in local network, with ethers.js as the client. The node runs with interval mining at 1 s (D11) |
| Account abstraction | A simplified, ERC-4337-style wallet contract, created per owner by a factory | Sponsored gas is simulated on the local network (PAT-7) |
| Zero-knowledge proofs | circom 2.x, snarkjs, circomlib and circomlibjs (Poseidon), Node.js | Solidity verifier exported from snarkjs. The first circuit proves that an issued claim meets a threshold |
| Storage | IPFS through the Pinata SDK, behind a loopback proxy | Data is encrypted before upload. Only the proxy holds the Pinata credentials (DAT-10) |
| Data model | HL7 FHIR R4-inspired JSON: Patient and Observation | "Inspired" is deliberate. Full conformance is a stretch goal (INT-2). Consent is a read-only projection of the on-chain token |
| Evaluation data | Synthea, exported as FHIR R4 synthetic patients | MIMIC-III is a stretch goal only (DAT-7) |
| Encryption | AES-GCM | Unique nonce per encryption |
| Secret sharing | secrets.js-grempe (Shamir) with hash commitments | The deck names an "ssss" library, but the final plan uses secrets.js-grempe |
| Post-quantum | @noble/post-quantum (pure TypeScript, no WASM build step): ML-DSA (Dilithium) and ML-KEM | Pre-1.0, so pin an exact version. All calls sit behind a swappable `PQCProvider` interface. The original plan named a different library; see the TRD |
| Local services | Node.js on 127.0.0.1: coordinator and dealer launcher (port 4100), IPFS proxy (port 4101), relayer | Bearer token per session, CORS limited to the frontend (D17) |
| Frontend | React.js with ethers.js and an injected wallet such as MetaMask | Responsive web UI. The deck says MetaMask SDK and calls it a mobile dashboard |
| Testing | Hardhat tests for contracts, Jest for Node modules | Jest is listed in Rohith's plan. @noble/post-quantum is ESM-only, so set up Jest's ESM mode in the Foundations phase |

## Survey gap coverage

Of the ten gaps in the team's paper, three are addressed, six are partly addressed, and one carries a stated limitation. The status column is deliberately conservative, so the report does not claim more than the modules deliver.

| # | Gap | What the survey found | Response in this project | Status | Requirements |
| --- | --- | --- | --- | --- | --- |
| 1 | No cross-border or cross-chain support | Mostly single-chain systems | Cross-institution portability demonstrated on a single chain | Partly addressed | INT-1, INT-3, CON-6 |
| 2 | ZKP too slow for real-time use | Most prototypes | Proof batching with no aggregation, plus measured latency and gas | Partly addressed | ZKP-4, ZKP-5 |
| 3 | No quantum resistance | Classical cryptography throughout | ML-DSA signatures, ML-KEM share transport and key wrapping. The SNARK layer and chain signatures stay classical | Partly addressed | DAT-4, DAT-5, DAT-9, PAT-5, PAT-6, T1-4, SEC-2 |
| 4 | Key loss leads to data lockout | MediBloc and PolyMed-style systems | Tier 1 guardian recovery and a Tier 2 custodian share, all from one k-of-(n+1) scheme | Addressed | T1-1 to T1-10, BG-6, BG-7 |
| 5 | No granular, time-bound consent | SmartAccess and Patientory-style systems | Time-limited, revocable, role-scoped consent tokens enforced by the contracts. A key already delivered cannot be recalled (limitation 6). **Status changed from Addressed in v1.6** | Partly addressed | CON-1 to CON-4, PAT-4, DAT-5 |
| 6 | Only simulated data tested | Most prototypes | Primary evaluation on Synthea synthetic FHIR R4 data. The data is still synthetic, so the report says so. MIMIC-III is a stretch goal only, if credentialing clears with time to spare | Limitation stated | DAT-6, DAT-7 |
| 7 | No patient-friendly UI | Technical demos | ERC-4337-style account abstraction and a plain-language dashboard (updated in v1.4) | Addressed | PAT-3, UI-1, UI-3, UI-6, UI-8, UI-9 |
| 8 | No built-in HIPAA or GDPR compliance | All surveyed systems | Supports audit, revocation, and minimal disclosure. No compliance claim (updated in v1.4) | Partly addressed | AUD-1 to AUD-3, CON-2, BG-4, ZKP-2, ZKP-3 |
| 9 | No emergency break-glass access | All surveyed prototypes | Role-gated, time-boxed, audited break-glass that releases a minimal-disclosure capsule only | Addressed | BG-1 to BG-8, T1-10 |
| 10 | No FHIR or HL7 standards | Almost all systems | FHIR R4-inspired schema. Official validation is a stretch (updated in v1.4) | Partly addressed | DAT-1, DAT-6, INT-1, INT-2 |

## Delivery plan and dependencies

The plan runs seven phases in parallel across four people, with no calendar dates yet. The matrix is taken from the four module plans, updated for the accepted decisions. Items marked v1.6 are new.

| Phase | Anush | Punith | Rohith | Yagna |
| --- | --- | --- | --- | --- |
| Foundations | Solidity and Hardhat set up; a basic contract deployed locally | circom and snarkjs learned; hello-world circuit | IPFS and Pinata; encrypt, upload, retrieve pipeline; Jest ESM mode working (v1.6) | Tiered guardian scheme specified; frontend stack learned |
| Core build | Patient identity contract: registration, wallet linkage, DID/SSI, and each party's Dilithium public key (PAT-5). A minimal ABAC role registry ships early in this phase so Yagna can build the emergency-responder role against it. `PatientAccountFactory` and the DID rule (PAT-7, PAT-8, v1.6) | Age or eligibility circuit, with proving and verification keys | FHIR schema; AES-GCM before IPFS; hash on-chain; Synthea FHIR R4 dataset (DAT-6); client library and IPFS proxy (DAT-10, v1.6) | Break-glass contract: emergency role through ABAC, `triggerEmergencyAccess()`, mandatory audit event, time-boxed access; interval-mining node config (v1.6) |
| Extension | Consent token contract (issue, revoke, expire) with a proof-layer stub; `canAccess` and `checkAccess` (CON-4, v1.6) | Solidity verifier integrated with the consent contract | Hardened Tier 1 Shamir module dealing n+1 shares in one run, with capsule-only and full-record release modes; trust-category cap (T1-5, v1.6) | Emergency consent capsule: minimal-disclosure structure, pre-authorized by the patient |
| Advanced | Full ABAC wiring into consent logic, building on the early registry | Proof batching | Dilithium signature wrapping with the creator's key; signed recovery events; grantee key wrapping (DAT-5) | Tier 2 fallback: custodian co-sign and delay window, extending Rohith's Tier 1 module and consuming commitment-verified shares |
| Build and test | Unit tests for expiry, revocation, and unauthorized access checks | On-chain audit logging live | Recovery and integrity tests: commitment checks, policy-first ordering, capsule-only release | Frontend: patient dashboard (consent and guardian setup), responder view (request access, break-glass trigger) |
| Distinctiveness | Threat model and security analysis | Benchmarks and literature comparison | Two-hospital interoperability demo | Break-glass demo scenario owned end to end as the main viva moment |
| Wrap-up | Support integration; docs | Fix bottlenecks; docs | Support integration; docs | End-to-end integration test across all four modules; bug fixing |

Stretch goals, with no fixed phase: a diagnostic-threshold circuit (ZKP-6), MIMIC-III evaluation (DAT-7), and FHIR validation (INT-2).

### Interfaces to agree early

Most integration risk sits in shared contracts between modules. Freezing them early lets each person build against a stable shape.

| Interface | Produced by | Consumed by | Needed by |
| --- | --- | --- | --- |
| ABAC role IDs from the minimal registry | Anush | Yagna, Punith | Early Core build |
| Consent token schema | Anush | Punith, Rohith, Yagna | Before the Extension phase |
| Public-key lookup: each party's Dilithium key (PAT-5) and ML-KEM key (PAT-6) | Anush | Rohith | ML-KEM keys before Extension, Dilithium keys before Advanced |
| Account resolution: `accountOfOwner` and the DID rule (PAT-7, PAT-8, v1.6) | Anush | Yagna (routing), Rohith (seed script) | Early Core build |
| Access-check pair: `canAccess` and `checkAccess` (CON-4, v1.6) | Anush | Rohith (read path), Yagna (screens) | Before the Extension phase |
| Audit event schema | Punith | Every contract and the UI | Early Core build, so contracts emit events from the start |
| Record envelope: FHIR JSON, AES-GCM ciphertext, creator's Dilithium signature and signer ID, and CID or hash | Rohith | Anush (hash storage), Yagna (UI) | Core build |
| Dealer output: n+1 share bundles (share and salt) with on-chain commitments, including the custodian's | Rohith | Yagna (Tier 2) | Extension |
| Verified-share object: a share plus its commitment-check result | Rohith | Yagna (Tier 2) | Extension |
| Capsule schema and policy-check API before crypto | Yagna | Rohith (policy before crypto), Anush (roles) | Extension |
| Chain clock: interval mining and the no-time-jump rule (D11, v1.6) | Yagna | Everyone | Early Core build |

### Dependencies to watch

**ABAC ordering, resolved.** Yagna's Core build needs the emergency-responder role, but full ABAC wiring lands in Anush's Advanced phase. The fix is in PAT-4: Anush ships a minimal role registry early in Core build, Yagna builds BG-1 against it, and the full wiring follows in Advanced. Confirm delivery of the early registry at the next team sync.

Other links: Punith's verifier integration needs Anush's consent contract, so a stub interface comes first. Yagna's Tier 2 needs Rohith's n+1 dealing from the Extension phase. Rohith's policy-first test needs Yagna's capsule. Rohith's signature verification needs Anush's published public keys. Yagna's routing screen and Rohith's seed script need Anush's account factory, so it is an early-Core item. Anush's threat model needs every module's final design, so start a draft early and update it as designs settle.

### Dates the team still has to set

- Interface freeze for the table above.
- Start of integration (E2E-1).
- First full run of `npm run demo:breakglass`.
- Viva date and the report deadline.

No date is invented here. Set them at the next sync, and put them in this section.

## Testing and acceptance

Thirty acceptance tests cover the 67 requirements; eleven are new, seven in v1.6 and four in the second pass.

| Area | Acceptance test | Covers | Owner |
| --- | --- | --- | --- |
| Consent | Expired, revoked, and unauthorized requests revert. Valid requests pass. | CON-1 to CON-4 | Anush |
| Access-check pair (v1.6) | `canAccess` is a view call and writes nothing. `checkAccess` emits AccessGranted or AccessDenied, does not revert on a denial, and the outcome is readable from the receipt. | CON-4 | Anush, Rohith |
| Roles | The emergency-responder role can be assigned from the minimal registry before full ABAC wiring exists. A caller without it cannot call `triggerEmergencyAccess()`. Each ABAC role maps only to its allowed actions. | PAT-4, BG-1 | Anush, Yagna |
| Account factory (v1.6) | Two wallets get two accounts. The deployed address equals the precomputed one. A wallet resolves to its account, and a second create for the same owner does not make a second account. The routing check classifies a registered patient correctly. | PAT-7 | Anush |
| DID binding (v1.6) | `register` reverts for a DID hash that is not derived from the caller's own address, and a second registration of the same identifier reverts. | PAT-8 | Anush |
| Signer identity | A record signed by a doctor verifies against that doctor's published public key. A record checked against the wrong party's key fails. | PAT-5, DAT-4 | Anush, Rohith |
| Proofs | A valid proof of an issued claim verifies on-chain and an altered proof fails. A proof against a commitment other than the one registered for that patient and claim type is rejected, a replayed nonce reverts, and a batch of N proofs verifies in one transaction with a nonce checked per proof. | ZKP-2 to ZKP-4, ZKP-7 | Punith |
| Audit | Every listed action emits an event with the agreed schema. No successful `triggerEmergencyAccess()` call exists without a matching audit event. | AUD-1, AUD-2 | Punith, Yagna |
| Data | Altered ciphertext fails AES-GCM authentication, an altered record fails Dilithium verification, and a round trip returns the original FHIR data. | DAT-2 to DAT-4 | Rohith |
| FHIR shape (v1.6) | Every seeded resource has a valid FHIR id. The DID appears as an `identifier`. The claim extension is structurally valid. If INT-2 is done, the official validator passes. | DAT-1, DAT-6, INT-2 | Rohith |
| IPFS proxy (v1.6) | The proxy receives only ciphertext envelopes. A scan of the built frontend finds no Pinata credential. | DAT-10 | Rohith |
| Key wrapping | A grantee with a wrapped key decrypts the record. A party without one cannot. | DAT-5 | Rohith |
| Tier 1 | All n+1 shares reconstruct the same S, with no separate secret for the custodian. Every combination of at least k shares reconstructs S, and any k-1 do not. A tampered share is rejected before reconstruction, using the on-chain commitment and the salt from the bundle. Old and new shares after re-sharing do not combine. | T1-2, T1-4, T1-6 to T1-8 | Rohith |
| Trust mix (v1.6) | A guardian set in which every guardian shares one category reverts. A set in which any category holds k or more trustees reverts. A mixed set passes. The UI warns first. | T1-5, UI-2 | Rohith, Yagna |
| Policy order | For a request that fails the policy check, no cryptographic function is called, verified with mocks or spies. | T1-9, BG-5 | Rohith, Yagna |
| Capsule-only release | A break-glass request returns only the capsule fields. Asking for full-record decryption through the break-glass path is refused. The full record decrypts only through the lost-key flow. | BG-4, T1-10 | Rohith, Yagna |
| Tier 2 | The custodian's share is not released before both the co-sign and the delay. Cancelling during the delay blocks release. The custodian's share plus k-1 guardian shares reconstructs S, and the custodian's share alone does not. | BG-6, BG-7 | Yagna |
| Interoperability | The second-hospital endpoint accepts the same token and returns the same FHIR-style data. | INT-1, INT-3 | Rohith |
| Claim issuance | A commitment set without the Doctor or Lab role, or without an active Claim token from the patient that names the caller, reverts. Only the original issuer can replace a commitment, and the patient or the issuer can revoke it. Issuing and revoking are logged, the issuer is stored and emitted, and a value below the threshold cannot produce a proof. | ZKP-7 | Punith |
| Record registry | Only the patient's account can add a record. Records are listed by patient and resource type, the CID round-trips, and the emergency capsule registers under its own type. | DAT-8 | Anush |
| Keys for other parties | A clinician's or lab's keystore re-derives the public keys registered for it. A record it signs verifies against that key, and a type key wrapped to its ML-KEM key unwraps with the derived secret key. | DAT-9 | Rohith |
| Screens | My records lists the patient's records with a Verified badge. Shared with me shows only what is shared and nothing after revocation. The Hospital B viewer returns the same record for the same permission. | UI-8, UI-9, INT-3 | Yagna, Rohith |
| Lost-key recovery | On a new device the derived public keys equal the published ones, old records decrypt, and old signatures verify. The coordinator wraps S only to a one-time key whose hash matches the hash stored with the request, and never reads a record. | T1-10 | Rohith, Yagna |
| Clock and services (v1.6) | The countdown on screen equals the window expiry minus the latest block time, and no demo script advances time. A call to the coordinator without its token, or from another origin, is refused. | NFR time handling, NFR local-service security | Yagna, Rohith |
| Privacy | After the end-to-end run, scanning chain state and events finds no PHI, no shares, and no salts. | NFR privacy | Punith |
| End to end | The full break-glass scenario runs as an automated test and as a scripted live demo. | E2E-1, BG-8 | Yagna |
| Proof request and result (D19, D23) | A valid proof emits EligibilityVerified with the claim type, threshold, request id, and verifier, and the verifier reads it from the receipt. A required threshold of 256 or more is rejected. A value or threshold outside 8 bits cannot produce a proof. | ZKP-2, ZKP-3 | Punith, Anush |
| Capsule-mode authorization (D20) | The coordinator refuses a capsule request unless the on-chain mode is CapsuleOnly, the window is open, and the requester proves control of the on-chain requester address. It returns only the pre-authorized fields. A capsule that was saved but not pre-authorized releases nothing. | T1-10, BG-4 | Rohith, Yagna |
| Tier 2 guards (D21, D23) | The co-sign reverts for an unknown, cancelled, completed, or already co-signed request, and a second call cannot reset the delay. If the k-th valid guardian share arrives during the delay, the secret is rebuilt at once. | BG-6 | Yagna, Rohith |
| Local-service tokens and Hospital B (D18, D22) | Events, result, close, and share calls need the request token, and request creation needs the session token. Hospital B's origin can read from the IPFS proxy and cannot pin. | Local-service security, INT-3 | Rohith |

Evaluation data: functional tests, demos, and benchmarks use Synthea-generated synthetic FHIR R4 data (DAT-6). MIMIC-III is a stretch goal (DAT-7), pursued only if credentialing clears with time to spare, and no core requirement depends on it.

## Risks and mitigations

Twenty-one risks are tracked, three of them High. The deck and paper wording is still the first one to close. Four rows are new in v1.6.

| Risk | Impact | Mitigation | Owner |
| --- | --- | --- | --- |
| Deck or paper wording still overstates scope, even though this PRD now uses the aligned wording. Still open in v1.6: the deck and paper files reviewed with this PRD still carry the old claims | High | Apply the aligned wording from the last section to the deck and paper before the viva | All |
| Late cross-module integration failure | High | Freeze the shared interfaces early, stub across modules, and integrate continuously | Yagna |
| Break-glass misuse, since exceptional access is the easiest path to abuse | High | Role gating, minimal-disclosure capsule, time-boxed window, mandatory audit event, patient visibility, and the Tier 2 delay and cancel window | Yagna |
| **New.** The specification outruns the code. The TRD states that the real contracts, coordinator, dealer launcher, and frontend had not been run | Medium | Build a walking skeleton first: deploy, seed, and one break-glass run, early in Core build. Set the dates listed under Delivery plan | All |
| **New.** Documents drift apart. The v1.8 TRD still carried the pre-errata design while later documents had the corrections | Medium | Fold errata 1 to 19 into the TRD, keep the errata file with the documents, and re-check the five documents against each other at every version | All |
| **New.** The chain clock and the coordinator clock disagree, so countdowns freeze or deadlines fire early | Medium | One clock (D11): interval mining, deadlines read from block time, no time jumps in the demo | Yagna, Rohith |
| **New.** No out-of-band notification. An unconscious patient cannot cancel a Tier 2 request, and guardians have no screen | Low | State it as limitation 19. The scripted guardian cancel path demonstrates the mechanism | Yagna |
| Tier 2 helps only when exactly k-1 guardians respond, because the custodian holds a single share | Medium | Choose k with this in mind, keep k at least 2 so the custodian alone can never reconstruct, and state the limit in the report and demo. With k = 2 the custodian plus one guardian can rebuild S | Yagna, Rohith |
| Yagna's BG-1 depends on Anush's early minimal role registry | Medium | PAT-4 ships the registry early in Core build. Confirm delivery at the next sync | Anush, Yagna |
| Capsule-only release is enforced by the recovery flow, not by separate keys. The same S can unlock the full record, and any k colluding trustees could reconstruct it outside the system | Medium | Record the limit in the threat model (SEC-1). Mixed trust categories (T1-5) make collusion harder. Stretch: a separate capsule secret, split and dealt independently of S, so an emergency quorum can unlock only the capsule. A key derived from S would not do this, because the quorum rebuilds S | Rohith, Anush |
| Post-quantum library maturity: @noble/post-quantum is pre-1.0, so its API can change between minor versions | Medium | Pin an exact version, test in Foundations, and keep every call behind the `PQCProvider` interface so another library, such as a WASM-based binding, can be swapped in | Rohith |
| Identity keys derive from S, so any k colluding trustees could forge the patient's signatures as well as read records, and the keys cannot rotate without a new S | Medium | Accepted trade-off of deriving keys from S, recorded in the threat model. Mixed trust categories (T1-5) make collusion harder. Rotation would mean re-encrypting every record and re-dealing the shares, which is out of scope | Rohith, Anush |
| The record registry makes metadata public: which record types a patient has, how many, when each was added, and who created it | Low | Record it in the threat model (SEC-1). Hashing the resource type does not hide it, and a per-patient salt would have to be shared with every grantee, so it is out of scope | Anush |
| Memory overwriting in JavaScript is best effort because of garbage collection and immutable strings | Medium | Use Buffers and typed arrays, overwrite in place, run the dealer as a short-lived process, and document the limit in the threat model | Rohith |
| Hash commitments detect tampering but do not prove the dealer dealt consistently, unlike verifiable secret sharing | Low | Acceptable because the patient's own device is the dealer. State this trade-off in the threat model | Rohith, Anush |
| Revocation stops future contract-mediated access but cannot recall a type key already wrapped to a grantee, which also covers records of those types added later | Medium | Document the limit in the threat model and reword consent claims (done in v1.6). Stretch: versioned type keys | Anush, Rohith |
| SNARK setup is circuit-specific and trusted (for example Groth16), and a local ceremony is not production-grade | Low | State it as a prototype limitation: the setup is local and has a single contributor | Punith |
| A proof shows a claim as of issuance, and the verifier trusts the issuer. A real system would commit to a date of birth and prove against the current date | Medium | State both limits in the report. Issuing needs the Doctor or Lab role and the patient's own Claim permission naming that issuer. Only the original issuer can replace a commitment, and the patient or the issuer can revoke it. The issuer's address is stored and emitted, so a verifier can see who vouched. Claim values and thresholds must fit in 8 bits | Punith |
| Synthetic data is not real clinical data, so results may not reflect real record shapes or volumes | Medium | State the limitation in the report. MIMIC-III only as a stretch, if credentialing clears with time to spare | Rohith, Punith |
| Demo fragility from the local chain and the network dependency on Pinata | Medium | Rehearse the script including the scripted guardian responses, record a backup run, and keep an offline fallback such as a local IPFS node or cached ciphertext | Yagna |
| Learning curve across circom, Solidity, post-quantum cryptography, and IPFS | Medium | Foundations phase with a working hello-world deliverable per module | All |

## Decisions

All 23 decisions (D1 to D23) are accepted. This table records what was decided and where it changed this document.

| ID | Decision | Outcome | Status | Owner | Changed in |
| --- | --- | --- | --- | --- | --- |
| D1 | What break-glass releases | Break-glass releases the pre-authorized emergency capsule only, not the full record. Full decryption stays with the patient's own lost-key recovery flow | Accepted | Team | BG-4, T1-10, flows B and C, recovery stage 8, testing, risks |
| D2 | How Tier 2 works | The custodian's escrowed share is generated in the same Setup phase as the Tier 1 guardian shares. The custodian is trustee n+1 in a k-of-(n+1) scheme, not a separate process | Accepted | Team | T1-2, T1-4, T1-6 to T1-8, BG-6, UI-2, recovery architecture, flows B and C, testing, risks, plan |
| D3 | Routine-access key delivery | The patient's client wraps a key to the grantee's ML-KEM public key at consent time. Refined in v1.2: one type key per resource type, derived from S (see DAT-5) | Accepted | Team | DAT-3, DAT-5, flow A, risks |
| D4 | Commitments and salts | Commitments are on-chain. Salts travel inside the ML-KEM-encrypted share bundle | Accepted | Team | T1-2, T1-4, T1-7, what-lives-where table |
| D5 | Guardians in the demo | Scripted guardian responses, not a built UI screen | Accepted | Team | UI-7, BG-8, non-goals, flow B, risks |
| D6 | Evaluation data | Synthea-generated synthetic FHIR data is the primary dataset. MIMIC-III is dropped from the current plan and is a stretch goal only, if credentialing clears with time to spare | Accepted | Team | DAT-6, DAT-7, ZKP-5, gap 6, testing, risks, claims table |
| D7 | Account abstraction depth | Simplified account abstraction, described as ERC-4337-style | Accepted | Team | PAT-3, non-goals, stack, gap 7 |
| D8 | First circuit | The first circuit proves age or eligibility. A diagnostic-threshold circuit is a stretch goal | Accepted | Team | ZKP-2, ZKP-6, plan |
| D9 | Who signs a record | The signer is whoever created the record: the patient for self-uploaded records, a doctor or lab for records they create. The identity contract publishes each party's Dilithium public key alongside their wallet and DID link | Accepted | Team | PAT-5 (new, not previously in Anush's task list), DAT-4, record envelope, flow A, testing |
| D10 | Wallet-to-account resolution | A `PatientAccountFactory` deploys one account per owner wallet at a precomputable address and resolves wallet to account. Patient routing looks the identity up by account. Without this, no document says how an account is created, and the first screen would classify every patient as unregistered | Accepted | Anush | PAT-7, PAT-3, flows A and C, delivery plan |
| D11 | One clock | The local chain runs with interval mining at 1 s. Interface countdowns and coordinator deadlines read block time. The demo never advances time, and tests that do run on a separate in-process network | Accepted | Yagna, Rohith | NFR time handling, BG-2, UI-4, risks |
| D12 | Where record crypto runs | Encryption, signing, key wrapping, and decryption live in a client library used by the browser and by Node scripts. A loopback IPFS proxy on port 4101 holds the Pinata credentials and sees only ciphertext | Accepted | Rohith | DAT-10, system scope, technology stack |
| D13 | Access-check pattern | `canAccess` is a free view call for the interface. `checkAccess` is the logged transaction and does not revert on a denial. Demo identities are Hardhat's unlocked accounts, so no private keys sit in the fixture | Accepted | Anush | CON-4, DAT-9 |
| D14 | Trust-mix rule | Besides rejecting a guardian set where every guardian shares one category, the contract rejects any set where one category holds k or more of the n+1 trustees | Accepted | Rohith, Yagna | T1-5, UI-2 |
| D15 | DID binding | `register` accepts only a DID hash derived from the caller's own address | Accepted | Anush | PAT-8 |
| D16 | FHIR validity | Ids follow the FHIR id pattern, the DID is an `identifier`, the issued claim is a valid complex extension, and Consent stays a read-only projection | Accepted | Rohith | DAT-1, DAT-6, INT-2 |
| D17 | Local-service security | Loopback services require a session bearer token and allow only the frontend origin. The browser reads the event stream with a fetch stream that sends the token in a header | Accepted | Rohith, Yagna | NFR local-service security |
| D18 | Hospital B's access to the IPFS proxy | The proxy's read-only fetch route accepts the Hospital B origin (port 5174) as well as the frontend. Pinning stays frontend-only, and Hospital B calls no other local service | Accepted | Rohith | INT-3, local-service security |
| D19 | Proof request and result | A verifier's request names the claim type, threshold, and request id. ProofGateway emits EligibilityVerified on success, so the verifier reads the result from the receipt, and the screen shows the verified fact from that event | Accepted | Punith, Anush | ZKP-3 |
| D20 | Capsule-mode authorization and fetch | The requester proves control of the on-chain requester address. The request must be CapsuleOnly with an open window. The coordinator reads the capsule through the read-only IPFS route and releases only pre-authorized fields. Saving emergency info registers the capsule before pre-authorizing it | Accepted | Rohith, Yagna | T1-10, BG-4 |
| D21 | Tier 2 guards and honesty | The custodian's co-sign reverts for an unknown, cancelled, completed, or already co-signed request. Limitation 20 states that the delay is not cryptographic | Accepted | Yagna, Rohith | BG-6, limitation 20 |
| D22 | Local-service tokens | A session token starts requests and calls the dealer and the proxy. A per-request token guards events, result, close, and shares. Shares must also be wrapped to the request's one-time key | Accepted | Rohith, Yagna | Local-service security |
| D23 | Small spec fixes | Explicit 8-bit range checks and a threshold below 256. ProofService is a benchmark and seed helper, not a demo process. A k-th valid share during the Tier 2 delay rebuilds the secret at once | Accepted | Punith, Rohith | ZKP-2, BG-6 |

### Values still to be set

The TRD sets the demo values: an access window of 180 seconds, a Tier 2 delay of 60 seconds, and a guardian collection timeout of 20 seconds, with the window at least twice the delay. A lost-key request has no on-chain window, so the coordinator enforces its own 300 second limit (errata 18). Only the numeric emergency-latency target is still to be set, after the first baseline measurement. The right of the patient and guardians to cancel (BG-7) is an accepted requirement.

## Scope versus deck and paper claims

Fourteen statements in the deck and paper are stronger than what the module plans build, and none has been edited in the deck or paper yet. A panel may probe these, so align the wording before the viva. This PRD applies the suggested wording throughout its body. The table quotes the original claims on purpose. Four rows are new in v1.6.

| Claim in the deck or paper | What the module plans deliver | Suggested wording |
| --- | --- | --- |
| ZK-Rollups for near real-time response | Proof batching: several verifications per transaction, with no aggregation | "Proof batching: several verifications per transaction" (v1.6: the v1.5 wording said "rollup-style aggregation") |
| Cross-chain interoperability | Two-hospital FHIR-style portability on one chain | "Cross-institution portability" |
| zk-SNARKs, zk-STARKs, and Bulletproofs (abstract page) | zk-SNARK circuits (Groth16) through circom and snarkjs | "zk-SNARK-based proofs" |
| **New.** Zero-knowledge consent tokens | Consent tokens are plain on-chain tokens. Zero-knowledge proofs cover only issued claims, such as age over a threshold | "Consent tokens, plus zero-knowledge proofs of issued claims" |
| Compliance-by-design (HIPAA, GDPR) | Audit trail, revocation, and minimal disclosure | "Supports compliance-oriented properties", with no certification claim |
| FHIR R4 standards, FHIR-compliant systems | A FHIR R4-inspired schema that is not validated against the specification | "FHIR R4-inspired data model" (v1.6: the v1.5 wording said "aligned") |
| Post-quantum security | ML-DSA signatures, ML-KEM share transport and key wrapping. SNARKs and Ethereum signatures stay classical | "Post-quantum integrity and key transport" |
| **New.** CRYSTALS-Dilithium "hybrid" layer | ML-DSA-65 (FIPS 204) signatures on their own, with no classical hybrid signature | "ML-DSA (FIPS 204) signatures" |
| Validation on MIMIC-III | Synthea-generated synthetic FHIR R4 data as the primary dataset. MIMIC-III is a stretch goal only (DAT-6, DAT-7) | Say the data is synthetic (Synthea), and mention MIMIC-III only if it is actually used |
| React.js mobile dashboard, ERC-4337 | Responsive React web app, simplified account abstraction | "Responsive web dashboard", "ERC-4337-style account abstraction" |
| **New.** Patients revoke access at any time | Revocation stops future contract-mediated access. Keys already delivered cannot be recalled (limitation 6) | "Revoke future access" |
| **New.** Recovery from a lost private key | Lost-key recovery restores records and the patient's keys, not the lost wallet | "Recovery of record and identity keys after device or passphrase loss" |
| 41 million records breached in 2023 (deck) | Not correct for 2023. OCR's 2023 tally is about 113 million individuals. About 41.7 million is OCR's 2022 tally | "Over 100 million individuals affected in 2023 (HHS OCR)" |
| Tool names in older plans (the secret-sharing library, local chain, JavaScript web3 client, IPFS client, and a Python secret-sharing option) | secrets.js-grempe, Hardhat's built-in network, ethers.js, and the Pinata SDK | Use the final tool names |

Also reconcile the reference numbering: the deck and the paper number the ten surveyed sources differently.

## Appendix

### Source documents

- **Module plans (source of truth):** the four final files for Yagna (Emergency Access and Recovery Tier 2, frontend, demo), Punith (proof layer, audit, benchmarking), Rohith (data layer, FHIR bridge, Tier 1 recovery, interoperability demo), and Anush (patient and consent layers, security analysis).
- **Superseded:** the earlier merged plan file, which listed an older tool set (liboqs, a Python secret-sharing library, and Ganache) and a basic Shamir module. One module-plan file still names liboqs and should be corrected.
- **Context:** the project deck, the IEEE-style paper text, and the Project Phase-I abstract page (Group 02, Department of CSE, BGS College of Engineering and Technology, Academic Year 2025-26). The Gantt chart on that page did not come through as text, so this PRD has no dates.
- **Tier 1 recovery diagram:** seven stages in v1.5. The architecture section here splits custody into two steps, so it numbers eight.
- **External source for the breach statistic:** [HHS Office for Civil Rights, Annual Report to Congress on Breaches of Unsecured Protected Health Information, Calendar Year 2023](https://www.hhs.gov/sites/default/files/breach-report-to-congress-2023.pdf), and the HHS OCR breach portal.
- **Companion documents**, revised after this PRD: the TRD, the App Flow, the Backend Schema, and the Architecture.

Team: Anush G (1MP23CS004), Punith A (1MP23CS045), Rohith Gowda R (1MP23CS049), Yagna Maurya M V (1MP23CS061).

### Limitations to state in the report

Twenty limitations go in the report. The list is used word for word as Known limits in the TRD. Items 6, 8, and 12 are extended in v1.6, and items 19 and 20 are new.

1. **An unmaintained secret-sharing library.** secrets.js-grempe has had no release since September 2019. It is small and passed an end-to-end test of the recovery flow, which is acceptable for a prototype.
2. **Synthetic data.** All evaluation uses Synthea-generated data, so results may not reflect real record shapes or volumes. MIMIC-III is a stretch only.
3. **Local trusted setup.** The Groth16 setup is circuit-specific, done locally, and has a single contributor, so it is not production-grade.
4. **Colluding trustees.** Any k colluding trustees can read everything and forge the patient's signatures, because the keys derive from S. Keys cannot rotate without a new S, which means re-encrypting every record and re-dealing the shares.
5. **Capsule-only release is enforced by the flow.** It is not enforced by separate keys: the emergency quorum rebuilds S. A separate capsule secret, dealt independently of S, would fix this and is a stretch.
6. **A revoked grantee keeps type keys.** Revocation cannot recall keys already delivered, so a revoked grantee can still derive keys for records of the same types added later, and the CIDs are public. An emergency summary already on a screen cannot be unseen either. Consent is therefore time-limited by the contracts, not by the keys. (Extended in v1.6.)
7. **Record metadata is public.** The record registry shows which record types a patient has, how many, when each was added, and who created it.
8. **The coordinator briefly holds S in both modes.** It needs S to derive the capsule key in break-glass mode and to wrap it for the new device in lost-key mode, and it wipes S straight after. Overwriting memory is best effort in JavaScript, because strings are immutable and the garbage collector may keep copies, so ending the process is the real mitigation. This applies to the dealer too. (Extended in v1.6.)
9. **The relayer is trusted for liveness.** It can delay or drop a signed action but cannot forge one.
10. **The attestation is UI-only.** The responder's confirmation that the patient cannot give consent is stored in the signed audit record. The contract does not enforce it.
11. **Wallet recovery is out of scope.** Lost-key recovery restores records and keys, not control of a lost wallet or account.
12. **Tier 2 needs k-1 guardians.** The custodian holds a single share, so the fallback only helps when exactly one guardian share is missing, that is, when k-1 guardians respond. With k = 2, the custodian plus one guardian is enough to rebuild S. (Extended in v1.6.)
13. **Post-quantum scope is signatures and key transport only.** Record and event signatures and share and key encapsulation use post-quantum algorithms. The proof system and Ethereum transaction signatures stay classical.
14. **Claim values are as of issuance, and the issuer is trusted.** A proof shows the value an issuer committed, not the current date. The verifier trusts the issuer, and claim values and thresholds must fit in 8 bits.
15. **Gas figures are estimates.** They come from skeleton contracts, not the final ones, and must not be quoted as results.
16. **Commitments detect tampering, not inconsistent dealing.** Hash commitments do not prove the dealer dealt consistently, unlike verifiable secret sharing. The patient's own device is the dealer, which is why this is acceptable.
17. **A lost passphrase with no backup file** goes through guardian recovery.
18. **Records authored by someone other than the patient are simulated by the seed script.** A production route would have the creator encrypt to the patient's ML-KEM key and sign a hash of the plaintext inside the payload, and the patient's client would re-encrypt under the record key and register it.
19. **No out-of-band notification.** The patient sees a break-glass or Tier 2 request on the dashboard or the Activity screen only after opening the app, and guardians have no screen, so an unconscious patient cannot cancel in practice. (New in v1.6.)
20. **The Tier 2 delay is not cryptographic.** The custodian can decrypt its own bundle at any time, so the delay holds only because the contract records when it ends and the coordinator honors that time. A compromised custodian that colludes with k-1 guardians can rebuild S without waiting. The delay gives an honest patient a visible window to cancel. (New in the second pass.)
