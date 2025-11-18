---
# Secret Network – TEE-Agnostic Node Architecture (SGX, AMD SEV, AWS Nitro)

> **Status:** Experimental R&D (non-production)
> **Scope:** Secret Network node fork / extension exploring a TEE-agnostic design with support for multiple CPU/VM TEEs (Intel SGX, AMD SEV, AWS Nitro)
> **Audience:** Protocol engineers, systems researchers, confidential computing / TEE experts

---

## 1. Overview

This repository explores a TEE-agnostic architecture for Secret Network nodes with the long-term goal of supporting multiple trusted execution backends beyond Intel SGX, while remaining within the realm of CPU/VM-style TEEs, such as:

* Intel SGX (current production TEE for Secret Network)
* AMD SEV / SEV-SNP (confidential VMs on EPYC)
* AWS Nitro Enclaves (isolated compute environments in AWS EC2)

The core idea is to decouple the Secret execution model from a single TEE vendor/implementation, without immediately changing the production network. This is strictly R&D, intended to:

1. Design a clean abstraction layer (`TEEBackend`) between the Cosmos/CometBFT node and the confidential execution environment.
2. Prototype alternative CPU/VM TEE backends (e.g., SEV, Nitro) that could execute CosmWasm within confidential VMs/enclaves.
3. Study the security, complexity and attack surface implications of a multi-TEE model for Secret Network.

GPU TEEs (e.g., NVIDIA Blackwell confidential computing) are explicitly out of scope for node execution in this project. They may be explored separately as off-chain confidential AI/oracle components, not as consensus nodes.

---

## 2. Motivation

### 2.1 Current Secret Network Model (SGX-Centric)

Secret Network currently relies on:

* Cosmos SDK + CometBFT for consensus and networking
* A CosmWasm-based execution environment running inside Intel SGX enclaves on validator nodes
* Enclave attestation and TCB validation (e.g., via `quartz-tcbinfo` and Intel DCAP) to ensure:

  * all validators execute a verified enclave binary
  * contract state remains confidential and integrity-protected

This architecture provides strong privacy and integrity guarantees, but couples Secret tightly to:

* x86-64 CPUs with SGX support
* Intel’s attestation ecosystem
* A specific enclave runtime and build pipeline

### 2.2 Why Consider Additional CPU TEEs?

There are other production-grade, CPU/VM-centric TEEs:

* AMD SEV / SEV-SNP on EPYC (confidential VMs with per-VM memory encryption and attestation)
* AWS Nitro Enclaves (isolated, attested environments inside AWS EC2 instances)

These environments share similar properties with SGX from the perspective of a blockchain node:

* Enforce confidentiality and integrity for code and data in a VM/enclave
* Provide remote attestation primitives
* Run standard CPU code (e.g., Rust, Go, CosmWasm VM) with minimal porting compared to GPU execution

This raises research questions:

* Can Secret support multiple CPU TEEs (SGX + SEV + Nitro) safely?
* What is the right abstraction boundary between node logic and TEE specifics?
* How does a multi-TEE design affect attack surface, trust assumptions, and node diversity?

### 2.3 Security Caveat: More TEEs = Larger Attack Surface

Adding more enclave types does not automatically increase security. It can:

* Increase the attack surface, since each TEE vendor has its own:

  * bugs
  * microarchitectural risks
  * attestation complexity

For this reason, the goal of this project is not to claim that “more TEEs are strictly better”, but to:

* Build a clear, auditable interface for multi-TEE execution.
* Understand, at a research level, what it would mean to support SEV or Nitro alongside SGX.
* Make it easier to evaluate trade-offs if the ecosystem ever wants to move in that direction.

---

## 3. Problem Statement

We aim to answer, at a technical design level:

**Can we design and implement a TEE abstraction layer for Secret Network such that:**

* Existing SGX-based validators continue to function unchanged.
* Experimental node types can use alternative CPU TEEs (e.g., AMD SEV, AWS Nitro) as confidential execution backends.
* Determinism and consensus safety are preserved.
* Attestation from different TEEs can be represented and validated in a uniform way.

We break this into sub-problems:

1. TEE abstraction

   * Define a vendor-agnostic `TEEBackend` interface for executing CosmWasm contracts inside a confidential environment.

2. Alternate CPU TEE backends

   * Prototype backends that delegate execution to:

     * Local SGX (current model)
     * Remote SEV/SNP VMs
     * Remote Nitro Enclaves

3. Attestation model

   * Design a common `AttestationReport` structure that can encode:

     * SGX measurements
     * SEV-SNP attestation
     * Nitro attestation docs

4. Compatibility & safety

   * Explore how these backends can coexist in devnet/testnet scenarios, without impacting mainnet’s security assumptions.

---

## 4. High-Level Architecture

### 4.1 Components

We split the system into:

1. **TEE-Agnostic Secret Node**

   * Cosmos SDK + CometBFT
   * `x/compute` (or equivalent) module
   * `TEEBackend` interface with concrete implementations:

     * `SgxBackend` (existing SGX enclave code path)
     * `RemoteVmBackend` (for SEV/Nitro remote VMs)

2. **Remote VM TEE Hosts (SEV / Nitro)**

   * Machines or cloud instances configured with:

     * AMD SEV-SNP-enabled hypervisors, or
     * AWS Nitro Enclaves
   * Run a CosmWasm VM runtime inside the confidential VM/enclave
   * Expose a secure RPC interface for executing contracts and returning outputs + attestation

3. **On-Chain TEE/TCB Registry (Optional, Future)**

   * Smart contract or native module storing:

     * Whitelisted measurements / code hashes per TEE type
     * Policy for acceptable TCB versions
     * Mapping of `TEEKind` → trust policy

### 4.2 Conceptual Diagram

```
Secret Node (TEE-Agnostic)
 ├─ Cosmos SDK + CometBFT
 ├─ x/compute / CosmWasm integration
 └─ TEEBackend
     ├─ SgxBackend         (local SGX enclave)
     └─ RemoteVmBackend    (remote SEV / Nitro VM)
           └─ RPC → VM Host (SEV/Nitro) → CosmWasm VM in TEE
```

---

## 5. TEE Backend Abstraction

### 5.1 Interface Sketch

On the Rust side (conceptual):

```
pub enum TEEKind {
    IntelSgx,
    AmdSevSnp,
    AwsNitro,
}

pub struct AttestationReport {
    pub tee_kind: TEEKind,
    pub measurement: [u8; 32],
    pub vendor_sig: Vec<u8>,
    pub tcb_version: u64,
    pub timestamp: u64,
    // Additional vendor-specific data can be included or referenced.
}

pub trait TEEBackend {
    fn init(&mut self, cfg: BackendConfig) -> anyhow::Result<()>;

    fn execute_contract(
        &self,
        contract_code: &[u8],
        env: &Env,
        msg: &[u8],
    ) -> anyhow::Result<Vec<u8>>;

    fn query_contract(
        &self,
        contract_code: &[u8],
        env: &Env,
        msg: &[u8],
    ) -> anyhow::Result<Vec<u8>>;

    fn get_consensus_keys(&self) -> anyhow::Result<ConsensusKeys>;

    fn attest(&self) -> anyhow::Result<AttestationReport>;
}
```

Concrete implementations:

* `SgxBackend`

  * Wraps existing SGX enclave code and Intel DCAP attestation path.

* `RemoteVmBackend`

  * Communicates (e.g., via gRPC/mTLS) with a remote TEE host (SEV VM or Nitro Enclave).
  * Sends contract execution requests and receives:

    * Output bytes / new state root
    * A TEE-specific attestation blob that is mapped into `AttestationReport`

### 5.2 Determinism and Consensus

For consensus safety:

* Given the same `(code, env, msg)` triple, all honest nodes—regardless of backend—must derive the same output.
* No backend may allow non-deterministic behaviour to leak into the state machine.

This implies:

* CosmWasm VM semantics must remain deterministic.
* External sources of randomness/time must be either:

  * Disabled, or
  * Funneled through deterministic, consensus-level APIs.

---

## 6. Remote VM TEE Backends (SEV / Nitro)

### 6.1 General Model

A Remote VM TEE host is:

* A physical or virtual machine with:

  * AMD EPYC + SEV-SNP, or
  * AWS EC2 + Nitro Enclave support
* Running a minimal CosmWasm runtime in a confidential VM/enclave
* Exposing a secure RPC interface:

  * `InitContract`
  * `ExecuteContract`
  * `QueryContract`
  * `GetAttestation`

Secret nodes using `RemoteVmBackend` send contract execution requests to this host and receive:

* Execution results (output, new state root)
* TEE attestation data

### 6.2 Attestation Path

For each VM/enclave:

1. The host configures a confidential VM / enclave with:

   * Specific OS image
   * Pinned CosmWasm runtime binary

2. The vendor TEE stack (SEV-SNP / Nitro) produces an attestation document:

   * Contains measurement of the VM image/code
   * Is signed by the platform/TEE root of trust

3. The Secret node verifies:

   * That the measurement matches a whitelisted value in local config or an on-chain registry
   * That the attestation chain is valid for the `TEEKind` in question

The same `AttestationReport` abstraction is used for SGX and non-SGX TEEs, enabling unified logging and policy handling.

---

## 7. Compatibility and Deployment Modes

### 7.1 Coexistence with SGX Nodes

Initial R&D will not touch mainnet:

1. Devnet with SGX-only nodes

   * Baseline behaviour and tests.

2. Devnet with mixed backends

   * Some nodes configured with `SgxBackend`.
   * Some with `RemoteVmBackend` (SEV/Nitro).
   * Compare state transitions and outputs under identical workloads.

3. Isolated TEE testnets

   * Dedicated networks using only SEV/Nitro backends for targeted experiments.

Only if and when the design proves robust and audited, would any mainnet-facing proposals be discussed at governance level.

### 7.2 Governance and Trust Policy (Long-Term)

A hypothetical long-term path (not a commitment):

1. Introduce `TEEKind` metadata at the protocol level.
2. Add an on-chain TEE/TCB registry that maps:

   * `(TEEKind, measurement, tcb_version)` → “allowed / disallowed”.
3. Through governance, optionally whitelist `AmdSevSnp` or `AwsNitro` for certain roles / phases.
4. Carefully monitor security implications and revert if issues arise.

---

## 8. Threat Model and Limitations

### 8.1 Threats

* Compromise or bug in any TEE implementation (SGX, SEV, Nitro) could undermine guarantees if that TEE is trusted for execution.
* Misconfiguration or incorrect attestation verification.
* Network attackers intercepting or tampering with RPC between node and remote VM host.

### 8.2 Assumptions

* Each TEE type (SGX, SEV-SNP, Nitro) provides standard confidential computing guarantees:

  * Confidentiality and integrity for code/data
  * Cryptographically sound attestation, under the vendor’s trust model

* CosmWasm execution remains deterministic given fixed inputs.

### 8.3 Explicit Limitations

* This project does not claim that multi-TEE is strictly safer than SGX-only.
* It does not propose GPU TEEs as execution environments for Secret nodes.
* It is not production-ready, and must not be used to handle real funds or sensitive workloads.

---

## 9. Roadmap (R&D Phases)

### Phase 0 — Code Survey & Design

* Identify all SGX-specific integration points in the current Secret node.
* Design `TEEBackend` and `AttestationReport` abstractions.

### Phase 1 — TEEBackend Refactor (SGX Only)

* Implement `TEEBackend` with `SgxBackend` wrapping existing behaviour.
* Ensure that the SGX-only node runs unchanged, but now through the abstraction layer.

### Phase 2 — Dummy Remote Backend

* Implement a `RemoteVmBackend` that talks to a simple non-TEE service (pure software).
* Verify interface correctness, error handling, and determinism.

### Phase 3 — SEV / Nitro PoC

* Stand up a SEV-SNP or Nitro environment.
* Deploy a minimal CosmWasm runtime inside a confidential VM/enclave.
* Wire up `RemoteVmBackend` to execute contracts there.
* Integrate basic attestation verification.

### Phase 4 — Cross-Backend Consistency Testing

* Run the same workloads across SGX and SEV/Nitro backends.
* Identify non-determinism or discrepancies.
* Iterate until behaviour is consistent or limitations are understood.

### Phase 5 — Optional TEE/TCB Registry & Governance Hooks

* Prototype an on-chain registry for TEE measurements.
* Define example governance rules for whitelisting TEEs in a testnet context.

---

## 10. Disclaimer

This repository and its design are:

* Experimental and research-focused
* Not affiliated with any official Secret Network roadmap, unless explicitly adopted by the core team
* Provided without any warranties of correctness or security
* Not intended for mainnet use or real economic value

Use at your own risk.

---

