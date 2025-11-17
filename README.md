# Secret Network – TEE-Agnostic Node Architecture with NVIDIA Blackwell Research Backend

> **Status:** Experimental R&D (non-production)  
> **Scope:** Secret Network node fork / extension exploring a TEE-agnostic design with a remote NVIDIA Blackwell–based confidential computing backend  
> **Audience:** Protocol engineers, systems researchers, confidential computing / TEE experts

---

## 1. Overview

This repository explores a **TEE-agnostic architecture** for Secret Network nodes, with the long-term goal of supporting **multiple trusted execution backends** beyond Intel SGX, including:

- existing **Intel SGX** enclaves used in the current Secret Network validator/compute model;  
- a **remote confidential computing backend** running on **NVIDIA Blackwell** hardware (e.g., DGX systems with GPU TEEs), accessed via a well-defined RPC interface.

The objective is **not** to replace the existing SGX-based network, but to:

1. **Decouple** the Secret execution model from a single TEE vendor/implementation;  
2. **Prototype** a “remote TEE sidecar” approach that can route contract execution to a separate machine/TEE (e.g., NVIDIA GPU TEE) while maintaining determinism and strong attestation;  
3. Provide a **research playground** for future evolutions where Secret could support heterogeneous TEEs (SGX, NVIDIA Confidential Computing, AMD SEV-SNP, Intel TDX, etc.).

This work is strictly experimental and **not suitable for mainnet** or for handling any sensitive production data.

---

## 2. Motivation

### 2.1 Current Secret Network Model

Secret Network today relies on:

- **Cosmos SDK + CometBFT** for consensus and networking;  
- A **CosmWasm-based execution environment** that runs inside **Intel SGX** enclaves;  
- **Enclave attestation and TCB validation** (e.g., via `quartz-tcbinfo` contracts and Intel DCAP) to ensure that all validators execute the same verified enclave binary and maintain confidentiality of contract state.

This design couples **privacy and correctness guarantees** tightly to:

- x86-64 CPUs with SGX support;  
- Intel’s attestation infrastructure;  
- a specific enclave build and runtime.

### 2.2 Why Explore NVIDIA Blackwell?

NVIDIA’s latest architectures (e.g., Blackwell) introduce **trusted execution environments in the GPU**, enabling:

- Confidential AI workloads (private model weights + private data);  
- Hardware-enforced isolation for GPU compute;  
- Remote attestation of GPU TEEs.

For Secret and other privacy-focused chains, this opens interesting research questions:

- Can we **offload contract execution or heavy compute** to a GPU-TEE backend while preserving determinism?  
- Can we support **heterogeneous TEEs** (CPU and GPU) under a single protocol?  
- What is the right **abstraction boundary** between consensus, contract execution and hardware-specific attestation?

This repository treats these questions as **research problems**, not solved engineering tasks.

---

## 3. Problem Statement

At a high level:

> **Can we design and implement a TEE abstraction layer for Secret Network such that:**
> - Existing SGX-based validators continue to function unchanged;
> - New node types can use a **remote TEE backend** (here, a Blackwell-based confidential computing service);
> - The network preserves determinism and safety, while being able to verify **attestation proofs** from multiple TEE vendors?

To make this concrete, we explore the following sub-problems:

1. **Interface design:**  
   Define a language- and vendor-agnostic interface (`TEEBackend`) for executing CosmWasm contracts inside a TEE.
2. **Remote backend:**  
   Implement a **Remote TEE Backend** that talks to a sidecar service running on NVIDIA Blackwell, which actually performs the contract execution.
3. **Attestation model:**  
   Specify a common attestation data structure that can represent:
   - Intel SGX measurements;  
   - NVIDIA GPU-TEE measurements;  
   - (potentially) future TEEs.
4. **Consensus & compatibility:**  
   Ensure that introducing a remote backend does not break:
   - determinism of contract execution;  
   - compatibility with existing SGX validators;  
   - the proof model for end-users (what does it mean to “trust” results).

---

## 4. High-Level Architecture

### 4.1 Components

The architecture introduces three main groups of components:

1. **Secret Node (TEE-Agnostic):**
   - Cosmos SDK + CometBFT node;  
   - A **TEE abstraction layer** (`TEEBackend` trait / interface);  
   - Concrete backends:
     - `SgxBackend`: current SGX enclave and call paths;
     - `RemoteBackend`: sends execution requests to a remote TEE sidecar.

2. **Remote TEE Sidecar (NVIDIA Blackwell):**
   - Runs on a DGX / Blackwell system (Grace CPU + Blackwell GPU);  
   - Implements an RPC server (e.g., gRPC) with methods:
     - `ExecuteContract`, `QueryContract`, `InitContract`, etc.;  
   - Executes CosmWasm bytecode inside a **GPU TEE** or combined CPU/GPU TEE environment;  
   - Provides **attestation reports** that can be verified by the Secret node.

3. **On-Chain Attestation / Registry:**
   - A smart contract or module that stores:
     - Vendor-specific TEE configuration / TCB info;  
     - Allowed measurements / code hashes;  
     - Policy for which TEEs are accepted at given protocol versions.

### 4.2 Execution Flow (Remote Backend)

1. Secret node receives a transaction that triggers a contract call.  
2. Node’s `x/compute` module calls into the `TEEBackend` interface.  
3. If configured to use `RemoteBackend`:
   - It serializes the execution request (code hash, state, message) into a canonical format;  
   - Sends it via RPC to the Remote TEE Sidecar.
4. The sidecar:
   - Runs the CosmWasm VM inside the GPU TEE (or equivalent protected environment);  
   - Produces:
     - The new state root / output;  
     - An **attestation report** for this execution (or for the loaded enclave / TEE context).
5. The Secret node:
   - Verifies the attestation report against allowed TEE configurations;  
   - Applies the execution results to its local state;  
   - Continues consensus as usual.

---

## 5. TEE Abstraction Layer

### 5.1 Conceptual Interface

On the Rust side (simplified example):

```rust
pub enum TEEKind {
    IntelSgx,
    RemoteNvTee,
    // Future: AmdSevSnp, IntelTdx, etc.
}

pub struct AttestationReport {
    pub tee_kind: TEEKind,
    pub measurement: [u8; 32],
    pub vendor_sig: Vec<u8>,
    pub timestamp: u64,
    // Vendor-specific fields can be embedded or referenced.
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
````

Implementations:

* `SgxBackend`:
  Wraps the existing SGX enclave, ECALL/OCALL interface and Intel attestation path.

* `RemoteBackend`:
  Sends requests to the remote NVIDIA-backed sidecar and parses its responses + attestation.

### 5.2 Determinism

For consensus safety, the following must hold:

* For a given `(contract_code, env, msg)` tuple, **all honest nodes** must obtain the same output (state transition) regardless of TEE backend;
* No backend may introduce non-determinism (e.g., time, random I/O, floating-point differences) that affects the output bytes.

This implies:

* CosmWasm execution must remain “pure” from the perspective of the chain;
* The GPU TEE path must be carefully constrained to deterministic operations, or any non-deterministic behaviour must be abstracted away before the chain sees a result.

---

## 6. Remote NVIDIA TEE Sidecar

### 6.1 Goals

The sidecar is a separate process / service running on a NVIDIA Blackwell platform that:

* Exposes a stable RPC interface to Secret nodes;
* Executes CosmWasm contracts inside an environment protected by **NVIDIA Confidential Computing** (GPU/CPU TEE);
* Provides **attestation artifacts** that Secret nodes can verify.

### 6.2 Example RPC API (Sketch)

```protobuf
service NvTeeExecutor {
  rpc InitContract (InitRequest) returns (InitResponse);
  rpc ExecuteContract (ExecuteRequest) returns (ExecuteResponse);
  rpc QueryContract (QueryRequest) returns (QueryResponse);
  rpc GetAttestation (AttestationRequest) returns (AttestationResponse);
}

message ExecuteRequest {
  bytes contract_code_hash = 1;
  bytes input_env         = 2;
  bytes input_msg         = 3;
  bytes state_root        = 4;
}

message ExecuteResponse {
  bytes output_data       = 1;
  bytes new_state_root    = 2;
  AttestationReport att   = 3;
}
```

The attestation format here is **vendor-specific** but must be mappable to the generic `AttestationReport` structure inside the Secret node.

### 6.3 Attestation Path

High-level steps:

1. Sidecar boots a **TEE-protected environment** (GPU confidential computing session) with:

   * predefined VM / container image;
   * pinned CosmWasm VM build.

2. The TEE environment exposes a **measurement** (hash of code + config).

3. NVIDIA’s confidential computing stack provides a **remote attestation** mechanism that:

   * signs the measurement with a vendor key;
   * allows verifiers (Secret nodes) to validate it via known CA / root of trust.

4. Secret nodes validate:

   * that the measurement corresponds to an expected / whitelisted build;
   * that the attestation issuer is an accepted vendor root;
   * that timestamp / TCB version are within allowed ranges.

Initially, this repo may use **mock attestation** for development, then gradually replace it with real NVIDIA CC attestation once the stack is integrated.

---

## 7. Consensus & Compatibility Considerations

### 7.1 Coexistence with SGX Nodes

In early phases, we envision:

* **SGX nodes** (current mainnet behaviour) and
* **TEE-agnostic experimental nodes** (with `RemoteBackend`)

running side by side on a testnet / devnet.

To avoid breaking consensus:

* The experimental network may initially be **separate** from mainnet;
* Later, a carefully managed testnet with both backends can be used to:

  * compare outputs;
  * validate that both SGX and Remote NV TEE generate identical state transitions.

### 7.2 Protocol Upgrade Path (Long-Term)

If research is successful and the community wishes to adopt a multi-TEE model, a possible sequence is:

1. **Introduce TEEKind in protocol metadata**, but keep SGX as the only allowed type.
2. **Add vendor-agnostic attestation registry** contract/module that can store TCB info for multiple TEEs.
3. **Whitelisting**: governance votes to allow additional TEEKind (e.g., RemoteNvTee) for specific roles (e.g., non-validator nodes, or limited-capacity validators).
4. Gradual migration where some portion of validators can run with RemoteNvTee, provided they prove deterministic behaviour and correct attestation.

This README does not prescribe policy; it only outlines the technical design that might enable such paths.

---

## 8. Threat Model and Assumptions

### 8.1 Adversaries

* Malicious node operators attempting to:

  * forge attestation reports;
  * run modified code inside a TEE;
  * bypass or disable TEE guarantees.

* Compromised TEE vendors or catastrophic TEE vulnerabilities.

* Network adversaries who can:

  * man-in-the-middle RPC calls between Secret nodes and Remote TEE sidecars;
  * replay or tamper with messages.

### 8.2 Assumptions

* The underlying TEE (Intel SGX, NVIDIA CC, etc.) provides:

  * confidentiality and integrity for code and data inside the enclave / TEE;
  * unforgeable attestation reports, under standard trust assumptions for the vendor.

* CosmWasm execution is deterministic given the same inputs.

* RPC channels between Secret nodes and Remote TEE sidecars are:

  * authenticated and integrity-protected (e.g., mutual TLS);
  * monitored for failures / delays.

### 8.3 Limitations

* This R&D does **not** aim to formally prove that SGX and NVIDIA TEEs provide equivalent security guarantees;
* Confidential computing on GPUs is relatively new and may evolve rapidly;
* The attestation formats, vendor roots of trust and APIs might change under us.

---

## 9. Roadmap (R&D Phases)

### Phase 0 — Codebase Survey & Design

* Map SGX dependencies in current Secret code (enclave, ECALL/OCALL, TCB validation);
* Design the `TEEBackend` trait/interface and identify all call sites;
* Draft vendor-agnostic `AttestationReport` structures.

### Phase 1 — TEEBackend Abstraction (No NVIDIA yet)

* Implement `TEEBackend` with:

  * `SgxBackend` (wrap existing behaviour);
  * `DummyRemoteBackend` (non-TEE, local process or simple RPC echo).
* Ensure the node compiles and runs with the abstraction layer in place.

### Phase 2 — Remote TEE Sidecar (Non-TEE PoC)

* Implement a standalone “Remote Sidecar” that:

  * exposes the RPC API;
  * executes CosmWasm in a standard environment (no TEE) for now;
  * returns mock attestation reports.
* Wire up `RemoteBackend` to talk to the sidecar.

### Phase 3 — NVIDIA Confidential Computing Integration

* Port the sidecar to a NVIDIA Blackwell system (DGX/Spark);
* Integrate NVIDIA confidential computing APIs to:

  * launch a GPU TEE enclave/session;
  * obtain real attestation quotes;
  * bind CosmWasm VM execution to that enclave.

### Phase 4 — Determinism & Cross-Backend Consistency

* Compare SGX vs Remote NV TEE execution results on identical workloads;
* Identify and eliminate sources of non-determinism;
* Draft test suites to verify backend consistency.

### Phase 5 — Attestation Registry & Policy (Optional)

* Implement a simple on-chain registry for TEE measurements;
* Prototype governance / configuration for whitelisting TEE kinds and TCB versions.

---

## 10. Disclaimer

This repository and design are:

* **Experimental and research-oriented.** No guarantees of correctness, security, or economic safety.
* **Not affiliated with nor endorsed as an official Secret Network upgrade** unless explicitly stated by the core team.
* Intended for:

  * exploration of TEE-agnostic architectures;
  * academic or prototype work on GPU-based TEEs in blockchain contexts.

Use at your own risk. Do **not** deploy to mainnet or rely on this code for real value.

---

## 11. References and Related Work

* Secret Network documentation and whitepapers (SGX-based confidential smart contracts).
* CosmWasm VM and integrations with SGX.
* Intel SGX, DCAP remote attestation.
* NVIDIA Confidential Computing and GPU TEEs (Blackwell architecture).
* TEE-agnostic frameworks (e.g., Enarx, Fortanix, etc.) exploring multi-TEE WebAssembly runtimes.

Further references will be added as the R&D progresses.
