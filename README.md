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
