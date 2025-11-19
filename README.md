# Secret Network – Multi-TEE & Confidential AI R&D
## (SGX / SEV / Nitro + NVIDIA Spark GPU TEEs)

> **Status:** Experimental R&D (non-production)  
> **Scope:**  
> - TEE-agnostic Secret node design (SGX, AMD SEV, AWS Nitro)  
> - Confidential AI / Oracle layer built on NVIDIA Spark (GPU TEE) for Secret  
> **Audience:** Protocol engineers, systems researchers, confidential computing / AI experts

---

## 1. Overview

This repository explores a **two-track R&D effort** around Secret Network:

1. **TEE-Agnostic Node Plane (CPU TEEs)**  
   - Make the Secret node execution layer *logically* independent from a single TEE vendor.  
   - Keep Intel SGX as the production baseline, but design an abstraction that can, in principle, support:
     - Intel SGX (current Secret production TEE)  
     - AMD SEV / SEV-SNP (confidential VMs on EPYC)  
     - AWS Nitro Enclaves (isolated compute in AWS EC2)

2. **Confidential AI Plane on NVIDIA Spark (GPU TEEs)**  
   - Use **NVIDIA Spark-class hardware (Grace + Blackwell GPUs with confidential computing)** as a high-performance **confidential AI / Oracle layer** for Secret:
     - Private AI oracles (risk, pricing, MEV detection, mempool analysis, etc.)  
     - Confidential analytics engines for Secret-based DeFi and apps  
     - An optional, privacy-preserving “Secret Copilot” for operators and developers

The key design decision is:

- **CPU TEEs (SGX, SEV, Nitro)** are reserved for the **consensus-critical node path**.  
- **GPU TEEs (NVIDIA Spark / Blackwell)** are used for **off-chain confidential AI**, feeding results into Secret via oracles/contracts, not running the node itself.

---

## 2. Goals & Non-Goals

### 2.1 Goals

- Design a **TEE abstraction layer** (`TEEBackend`) for Secret nodes that:
  - Preserves current SGX behaviour.
  - Allows experimentation with SEV / Nitro in devnets/testnets.

- Define and prototype a **Confidential AI Oracle stack** on NVIDIA Spark that:
  - Runs heavy AI/ML workloads inside GPU TEEs when possible.
  - Exposes a clean, deterministic interface to Secret contracts.
  - Can provide privacy-preserving signals: prices, risks, MEV alerts, liquidity decisions, etc.

- Explore a **Confidential Copilot** concept:
  - Off-chain assistant for validators / developers.
  - Can analyse logs, metrics, chain data in a privacy-preserving way.
  - Optional and strictly non-consensus-critical.

### 2.2 Non-Goals

- This project does **not**:
  - Propose replacing SGX in mainnet in the short term.  
  - Run the Secret node itself inside a GPU TEE.  
  - Claim that “more TEEs automatically increase security”.  
  - Provide production-ready code for mainnet.

---

## 3. Motivation

### 3.1 Secret Network & CPU TEEs

Secret Network today:

- Uses Cosmos SDK + CometBFT for consensus.  
- Runs CosmWasm-based smart contracts inside Intel SGX enclaves.  
- Relies on SGX attestation (e.g., DCAP + `quartz-tcbinfo`) to ensure:
  - The correct enclave binary is running.  
  - Contract state and keys remain confidential.

This works well but implies:

- A **strong dependency on SGX** (hardware + attestation stack).  
- Limited flexibility in hosting environments and future TEE alternatives.

A TEE-agnostic design with a **carefully scoped set of CPU TEEs** (SGX, SEV, Nitro) would:

- Allow structured experimentation in devnets/testnets.  
- Provide a cleaner boundary between:
  - “What the chain expects from a TEE” vs  
  - “How any specific vendor implements it”.

### 3.2 NVIDIA Spark & GPU TEEs for Confidential AI

NVIDIA Spark-class systems (Grace CPU + Blackwell GPUs) with confidential computing provide:

- Hardware-enforced **GPU TEEs** for AI workloads.  
- The ability to run:
  - Private models (weights never exposed in plaintext to the host).  
  - Private data (user / protocol data stays encrypted outside the GPU TEE).  
  - Attestable inference/training sessions.

This is extremely relevant for Secret because:

- Secret contracts can store and manage **encrypted state and logic**, but:
  - Heavy AI (LLMs, deep nets, complex analytics) is not practical *inside* SGX.  
  - Running large models off-chain is natural, but we want **attestable privacy** and **strong guarantees**.

By combining:

- Secret (private smart contracts, encrypted state) and  
- Spark (GPU-accelerated confidential AI),

we can build:

- **Confidential AI Oracles** that:
  - Read encrypted or partially protected data.  
  - Run AI analytics/inference in GPU TEE.  
  - Return compact, attestable results to Secret.

- **Confidential Copilot** patterns that assist:
  - Validators (alerting, tuning, monitoring)  
  - dApp devs (design, debugging, risk analysis)
  without exposing raw logs or sensitive data.

---

## 4. Architecture at 10,000 ft

Very high-level, we have two planes:

1. **Node Plane (CPU TEEs)** – consensus critical  

    Secret Node (TEE-Agnostic)
     ├─ Cosmos SDK + CometBFT
     ├─ x/compute / CosmWasm integration
     └─ TEEBackend
         ├─ SgxBackend         (local SGX enclave – current)
         └─ RemoteVmBackend    (remote SEV / Nitro VM – R&D)

2. **AI Plane (GPU TEEs) – off-chain, non-consensus**  

    Confidential AI Cluster (NVIDIA Spark / GPU TEE)
     ├─ AI Oracles (price, risk, MEV, liquidity)
     ├─ Confidential Analytics Services
     └─ Optional Copilot services (operator / dev assistant)

Secret contracts interact with the AI Plane via **oracle contracts** and **off-chain relayers**, but the core node stays CPU-TEE-based.

---

## 5. Subsystem A: TEE-Agnostic Node Plane (CPU TEEs)

### 5.1 TEEBackend Interface (Conceptual)

In Rust-like pseudocode:

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
        // Vendor-specific fields can be added or referenced.
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

Concrete backends:

- `SgxBackend`  
  - Wraps the existing SGX enclave and Intel DCAP attestation path.  

- `RemoteVmBackend`  
  - Talks to a remote VM TEE host (SEV-SNP or Nitro Enclave) over a secure RPC channel.  
  - Sends contract execution requests and receives:
    - Deterministic execution results.  
    - Vendor-specific attestation data mapped into `AttestationReport`.

### 5.2 Determinism & Safety

Consensus-critical constraints:

- For a given `(code, env, msg)` triple, **all honest nodes**, regardless of backend, must compute the **same result**.  
- No backend may leak:
  - host time,  
  - non-deterministic randomness,  
  - external side effects  
  into the state machine.

Thus:

- CosmWasm stays deterministic by design.  
- TEEs are used only to protect code/data, not to change the semantics of execution.

---

## 6. Subsystem B: Confidential AI Plane on NVIDIA Spark

### 6.1 Hardware & Trust Model

The Confidential AI layer assumes:

- A NVIDIA Spark-class system with:
  - Grace CPU for orchestration and secure VM/container hosting.  
  - Blackwell GPUs with **GPU-TEE / confidential computing** for AI workloads.

We treat the Spark cluster as:

- A high-performance **confidential AI “coprocessor”** for Secret.  
- Separate from consensus; it **does not run `secret-node`**.  
- A source of **attestable AI results** that can be consumed by Secret contracts.

### 6.2 Roles of the AI Plane

1. **Confidential AI Oracles**  
   - Tasks:
     - Price feeds, volatility estimates.  
     - Risk scoring for positions / addresses.  
     - MEV detection / mempool pattern analysis (for chains where mempool is visible).  
     - Liquidity optimisation signals (e.g., MAILN-like strategies).
   - Data:
     - Public chain data (from Secret + other chains).  
     - Off-chain market data.  
     - Possibly encrypted or aggregated private data sent from Secret.

2. **Confidential Analytics Engines**  
   - Batch or streaming analytics over:
     - Secret DeFi protocols.  
     - Cross-chain flows and risk metrics.  
   - Results fed back into Secret via governance or dedicated contracts.

3. **Confidential Copilot (Optional)**  
   - Helps node operators and developers:
     - Understand logs, performance, anomalies.  
     - Propose parameter changes or upgrades.  
   - Can run:
     - Partially inside GPU TEE (for sensitive data).  
     - Partially on top of public / non-sensitive context.

### 6.3 AI Oracle API (Conceptual)

The AI Plane exposes a stable, deterministic API, e.g.:

    AIRequest {
        id:            UUID,
        model_id:      String,
        task:          String,        // e.g. "risk_score", "price_forecast"
        input_payload: Bytes,        // structured data (JSON/CBOR/Protobuf)
        constraints:   PolicyConfig, // e.g. max_time, determinism constraints
    }
    
    AIResponse {
        id:              UUID,
        status:          "ok" | "error",
        output_payload:  Bytes,      // model output (JSON/CBOR/Protobuf)
        model_commit:    [u8; 32],   // hash of model weights/config
        tee_attestation: Bytes,      // GPU TEE attestation blob (optional)
    }

Relayers and oracle workers translate `AIResponse` into Secret contract calls.

---

## 7. Example Flows

### 7.1 CPU TEE Execution (TEE-Agnostic Node)

1. User sends a transaction to execute a Secret contract.  
2. Node’s `x/compute` module calls into `TEEBackend::execute_contract`.  
3. Depending on configuration:
   - `SgxBackend` runs the contract inside the local SGX enclave, OR  
   - `RemoteVmBackend` sends the request to a SEV/Nitro host and verifies attestation.  
4. The node applies the resulting state transition and continues consensus.

The rest of the network only sees deterministic state changes; which specific TEE did the work is abstracted away.

### 7.2 AI Oracle for a Secret Contract (Spark / GPU TEE)

1. A Secret contract wants, for example, a **risk score** or **liquidity adjustment signal**.  
2. It emits an event or updates a dedicated “oracle request” contract.  
3. An off-chain **Oracle Relayer** observes this event and sends an `AIRequest` to the Spark AI Plane.  
4. Inside Spark:
   - A model runs inside the GPU TEE on the relevant data.  
   - The AI Plane produces an `AIResponse` with:
     - The risk score / decision.  
     - Optional GPU-TEE attestation of the inference session / model hash.
5. The Oracle Relayer submits the `AIResponse` back into Secret, invoking an **Oracle Consumer contract**.  
6. The contract:
   - Verifies the response (optionally checks a commitment/attestation scheme).  
   - Updates its own logic (e.g., adjusts parameters, enforces limits, triggers actions).

Secret stays in control; Spark is a “confidential AI oracle provider”.

---

## 8. Repository Layout (Proposed)

A possible directory structure for this repo:

    /node/
      /tee/
        backend_trait.rs
        sgx_backend.rs
        remote_vm_backend.rs
      /docs/
        tee_design.md
    
    /ai-plane/
      /spark-oracle/
        api/
        inference-engine/
        attestation-integration/
      /docs/
        ai_oracle_design.md
        copilot_concepts.md
    
    /integration/
      secret_oracle_contracts/
      relayer/
    
    README.md
    ARCHITECTURE.md
    SECURITY_NOTES.md

This is indicative; actual code layout may evolve.

---

## 9. R&D Roadmap (Dual Track)

### 9.1 Node Plane (TEE-Agnostic CPU TEEs)

- Phase N0 – Code survey & TEEBackend design  
- Phase N1 – SGX-only refactor behind `TEEBackend`  
- Phase N2 – Dummy `RemoteVmBackend` (non-TEE local service)  
- Phase N3 – SEV / Nitro PoC with basic attestation  
- Phase N4 – Cross-backend determinism tests (SGX vs SEV/Nitro)  
- Phase N5 – Optional TEE/TCB registry and governance hooks for testnets

### 9.2 AI Plane (Confidential AI on Spark)

- Phase A0 – Define AI Oracle API and data models  
- Phase A1 – Implement baseline AI Oracle (no TEE, deterministic paths)  
- Phase A2 – Integrate with NVIDIA Spark and GPU-TEE attestation for inference  
- Phase A3 – Build Secret-side oracle contracts + relayers  
- Phase A4 – Pilot use cases:
  - Price oracle  
  - Risk / credit scoring  
  - MEV / volume pattern alerts  
  - Liquidity optimisation
- Phase A5 – Explore Confidential Copilot patterns (logs, metrics, tuning)

---

## 10. Disclaimer

This repository and design are:

- **Experimental and research-focused**  
- **Not** part of any official Secret Network roadmap unless explicitly adopted by the core team  
- Provided without any warranties of correctness or security  
- **Not** intended for mainnet use or handling real economic value

Use at your own risk.

---
