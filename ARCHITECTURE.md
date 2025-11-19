# ARCHITECTURE
## Secret Multi-TEE Node + Confidential AI on NVIDIA Spark

> **Project:** Secret Network – Multi-TEE Node (CPU TEEs) + Confidential AI / Oracle (GPU TEEs)  
> **Status:** Experimental / R&D

This document details the architecture for:

1. A **TEE-agnostic Secret node** that can, in principle, support multiple CPU TEEs (SGX, AMD SEV, AWS Nitro).  
2. A **Confidential AI Plane** running on NVIDIA Spark (Grace + Blackwell GPU TEEs), used as a high-performance confidential oracle and analytics layer for Secret.

The core separation is:

- **Node Plane (CPU TEEs)** → consensus-critical, conservative, SGX-first.  
- **AI Plane (GPU TEEs)** → off-chain, non-consensus, high-compute confidential AI.

---

## 1. Component Overview

We conceptually split the system into three domains:

1. **Secret Node Plane (CPU TEEs)**  
2. **Confidential AI Plane (NVIDIA Spark / GPU TEEs)**  
3. **On-Chain Contracts & Oracles**

### 1.1 Node Plane (CPU TEEs)

    Secret Node
     ├─ Cosmos SDK + CometBFT
     ├─ x/compute Module
     └─ TEEBackend
         ├─ SgxBackend         (local SGX enclave)
         └─ RemoteVmBackend    (remote SEV / Nitro VM)

Key properties:

- Runs consensus-critical logic.  
- Enforces determinism in contract execution.  
- Uses CPU TEEs for confidentiality (SGX today; SEV/Nitro for R&D).

### 1.2 AI Plane (NVIDIA Spark / GPU TEEs)

    Confidential AI Cluster (NVIDIA Spark)
     ├─ Inference Orchestrator (Grace CPU)
     ├─ GPU TEE Sessions (Blackwell)
     ├─ Model Registry & Policy
     └─ AI Services:
         ├─ Price / Risk Oracles
         ├─ MEV & Liquidity Analytics
         └─ Optional Copilot Services

Key properties:

- Off-chain, not part of consensus.  
- Can run expensive models (LLMs, deep nets, simulations).  
- Uses GPU TEEs and/or confidential VMs to protect data and model IP.

### 1.3 On-Chain Contracts & Oracles

    Secret Network (On-Chain)
     ├─ Application Contracts (DeFi, dApps, etc.)
     ├─ Oracle Request / Response Contracts
     └─ (Optional) TEE/TCB Registry Contract

- Application contracts can request AI / oracle results indirectly.  
- Oracle contracts receive off-chain responses and forward results to consumers.  
- A TEE/TCB registry (optional) can hold whitelisted TEE measurements for CPU TEEs and possibly GPU TEEs used in oracles.

---

## 2. Node Plane Architecture (TEE-Agnostic CPU TEEs)

### 2.1 TEEBackend Interface and Implementations

The node integrates a TEE abstraction:

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
    }
    
    pub trait TEEBackend {
        fn init(&mut self, cfg: BackendConfig) -> anyhow::Result<()>;
        fn execute_contract(&self, code: &[u8], env: &Env, msg: &[u8])
            -> anyhow::Result<Vec<u8>>;
        fn query_contract(&self, code: &[u8], env: &Env, msg: &[u8])
            -> anyhow::Result<Vec<u8>>;
        fn get_consensus_keys(&self) -> anyhow::Result<ConsensusKeys>;
        fn attest(&self) -> anyhow::Result<AttestationReport>;
    }

Backends:

- `SgxBackend`  
  - Direct ECALL/OCALL to SGX enclave.  
  - Uses Intel DCAP or similar for attestation.

- `RemoteVmBackend`  
  - Sends requests to a remote VM TEE host (SEV-SNP or Nitro Enclave).  
  - Receives execution result + VM attestation.  
  - Verifies that:
    - The measurement is expected.  
    - The attestation chain is valid.

### 2.2 Execution Flow (Node Plane)

Steps for contract execution:

1. Node receives a transaction with a contract call.  
2. `x/compute` prepares the execution context (`code`, `env`, `msg`).  
3. `x/compute` calls `TEEBackend::execute_contract(...)`.  
4. Depending on configuration:
   - `SgxBackend`:
     - Invokes SGX enclave locally.  
   - `RemoteVmBackend`:
     - Sends execution request to a remote VM TEE host.  
     - Receives result + attestation and validates them.
5. On success:
   - The deterministic result is applied to the state machine.  
   - CometBFT consensus commits the block as usual.

Determinism is enforced by:

- CosmWasm semantics.  
- Node-level restrictions against non-deterministic system calls.

---

## 3. AI Plane Architecture (NVIDIA Spark / GPU TEEs)

### 3.1 Spark Cluster Components

The Confidential AI Plane is composed of:

- **Inference Orchestrator (Grace CPU):**
  - Schedules requests across GPU TEE sessions.  
  - Manages model loading, versioning, and policy enforcement.

- **GPU TEE Sessions (Blackwell GPUs):**
  - Run AI models (LLMs, transformers, risk models, etc.).  
  - Keep weights and data confidential inside GPU TEE.  
  - Produce attestable outputs where supported.

- **Model Registry & Policy:**
  - Maps `model_id` → model weights, version, allowed inputs.  
  - Enforces which models may answer which types of requests.

- **AI Services:**
  - Price / risk oracles.  
  - MEV detection, liquidity optimisation logic.  
  - Copilot services (log analysis, advice).

### 3.2 AI Service API

Common API surface:

    AIRequest {
        id:            UUID,
        model_id:      String,       // e.g. "risk_v1", "mev_detector_v2"
        task:          String,       // e.g. "score", "forecast", "classify"
        input_payload: Bytes,        // e.g. JSON/CBOR with market/chain data
        constraints:   PolicyConfig, // limits: max_time, determinism, etc.
    }
    
    AIResponse {
        id:              UUID,
        status:          "ok" | "error",
        output_payload:  Bytes,      // e.g. JSON with risk score / decision
        model_commit:    [u8; 32],   // hash of model weights/config
        tee_attestation: Bytes,      // optional GPU TEE attestation blob
    }

The AI Plane ensures:

- For a given model and input, the output is as deterministic as model design allows.  
- Where available, `tee_attestation` proves that the model ran inside an approved GPU TEE session.

---

## 4. Integration: Secret ⇄ AI Plane via Oracles

### 4.1 On-Chain Oracle Contracts

We use a pattern with two types of contracts:

1. **Oracle Request Contract**
   - Receives requests from application contracts.  
   - Stores pending requests and emits events for off-chain relayers.

2. **Oracle Consumer (Application) Contract**
   - Calls the request contract to create a new oracle request.  
   - Later receives fulfilled responses and updates its state.

### 4.2 Off-Chain Relayer

The relayer:

- Watches events or state changes in the Oracle Request contract.  
- Translates them into `AIRequest` calls to the AI Plane (Spark).  
- Converts `AIResponse` back into a transaction to Secret that:
  - Updates the Request contract.  
  - Notifies the Consumer contract.

### 4.3 Example Flow: Risk Oracle

Scenario: a Secret DeFi protocol wants a confidential risk score for a portfolio.

1. Application contract calls `OracleRequest::new_request(...)`.  
2. Request contract logs an event with `request_id` and input data (possibly encrypted).  
3. Relayer picks up the event and forms an `AIRequest` for model `risk_v1`.  
4. AI Plane on Spark:
   - Runs the risk model in GPU TEE.  
   - Produces `AIResponse` with risk score + model commitment + (optional) TEE attestation.  
5. Relayer sends `fulfill_request(request_id, AIResponse)` to the Request contract.  
6. Request contract validates the response and forwards data to the Consumer contract.  
7. Consumer contract:
   - Updates its internal state (e.g., collateral requirements, alerts).  
   - May choose to verify a commitment or signature based on policy.

---

## 5. Example: Liquidity AI + Multi-TEE Node

This example ties both planes together.

- Node Plane: runs the DeFi contract logic and state updates inside SGX or SEV/Nitro.  
- AI Plane: computes strategy parameters for liquidity management in a confidential way.

High-level steps:

1. A Secret liquidity management contract (running in SGX or SEV/Nitro) needs updated strategy parameters.  
2. It sends an oracle request describing:
   - Current positions, volumes, volatility (possibly in encrypted form).  
3. The AI Plane (Spark) runs a MEMPOOL-/MEV-/DeFi-aware model to suggest:
   - New ranges, fees, or rebalancing actions.  
4. The AI output is fed back via oracle into the contract.  
5. The contract, still running in a CPU TEE, decides what to execute on-chain given:
   - AI suggestions,  
   - on-chain constraints,  
   - safety thresholds.

The key point: the **decision** and **state transition** remain in the CPU TEE node path; the AI Plane provides confidential, high-quality *signals*.

---

## 6. Security & Trust Considerations

### 6.1 CPU TEEs (Node Plane)

- SGX remains the primary, battle-tested TEE for Secret.  
- SEV/Nitro backends are strictly experimental and must:
  - Prove determinism.  
  - Prove attestation correctness.  
  - Be gated behind configuration / governance in testnets.

### 6.2 GPU TEEs (AI Plane)

- GPU TEEs for AI oracles:
  - Do not participate in consensus.  
  - Provide **advisory** outputs to contracts.  
- Contracts should:
  - Treat AI results as inputs, not as absolute truth.  
  - Implement limits, sanity checks, and fallback behaviour.  
- Attestation from GPU TEEs is beneficial but:
  - May have different trust properties than SGX/SEV/TDX;  
  - Should be modelled as *additional assurance*, not a replacement for economic/game-theoretic safeguards.

### 6.3 Oracle & Relayer Risks

- Relayers can be:
  - Byzantine, offline, or censored.  
- Mitigations:
  - Multiple relayers, possibly with different operators.  
  - On-chain validation of signatures/commitments.  
  - Timeouts and fallback logic on contracts.

---

## 7. Roadmap Mapping (Architecture Perspective)

- Node Plane:
  - N0–N5 as described in README (TEE abstraction → SEV/Nitro PoC → testnet registry).  
- AI Plane:
  - A0–A5 (API → non-TEE AI → GPU TEE integration → oracles → copilot experiments).

Architecturally, the two tracks are **loosely coupled**:

- Node Plane can evolve independently (even if AI Plane does not exist).  
- AI Plane can serve other ecosystems too, but here it is designed to integrate tightly with Secret’s privacy guarantees.

---

## 8. Summary

This architecture:

- Keeps Secret’s **core node** conservative and TEE-focused on CPU enclaves (SGX now, SEV/Nitro as R&D).  
- Exploits NVIDIA Spark’s **GPU TEEs** where they shine: confidential, heavy AI workloads.  
- Connects both worlds using a clear, auditable oracle pattern, so:
  - Consensus stays simple and deterministic.  
  - AI remains powerful, flexible, and privacy-preserving.

It is intentionally modular so that individual parts can be replaced, upgraded, or dropped without forcing the entire system to change.

---
