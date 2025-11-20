---

# HERO CASE – MAILN: Mempool-Aware AI Liquidity Navigator

## Confidential Liquidity Intelligence for Secret + NVIDIA Spark

> **Status:** Concept / R&D
> **Role:** Flagship (“hero”) use case for the Confidential AI Plane on NVIDIA Spark
> **Scope:** Mempool-aware, MEV-aware, AI-driven liquidity management supporting Secret-based DeFi protocols

---

## 1. Concept Overview

MAILN (Mempool-Aware AI Liquidity Navigator) is the **hero use case** for the Confidential AI Plane in this repository.

Goal (one-liner):

* Use **confidential AI running on NVIDIA Spark (GPU TEEs)** to continuously analyse:

  * mempool activity (where visible),
  * cross-chain flow,
  * on-chain state and liquidity,
    and produce **private, high-quality liquidity decisions** that feed into **Secret-based DeFi protocols** running inside CPU TEEs (SGX / SEV / Nitro).

MAILN sits at the intersection of:

* Secret’s **confidential smart contracts** and CPU TEEs.
* NVIDIA Spark’s **GPU-accelerated confidential AI**.
* A TEE-aware **oracle and relayer pattern** that connects both.

The system is designed to:

* Improve LP returns and capital efficiency.
* Reduce user slippage and MEV-induced damage.
* Preserve privacy around strategy, parameters, and models.

---

## 2. Roles & Responsibilities

We structure MAILN into three main domains, aligned with the overall architecture.

### 2.1 Node Plane (Secret, CPU TEEs)

* Runs the **on-chain parts of the strategy**, in SGX or other CPU TEEs:

  * Liquidity positions (e.g., CLAMM positions, range orders, vaults).
  * Risk limits, policy constraints, whitelists / blacklists.
  * Actual execution of rebalances and routing on-chain.

Key actors:

* MAILN Controller Contract
* Liquidity Vault / DEX Contracts (Secret-based)
* Optional TEE/TCB registry (for CPU TEEs only)

### 2.2 AI Plane (NVIDIA Spark, GPU TEEs)

* Runs **heavy AI workloads**, possibly inside GPU TEEs:

  * Mempool pattern analysis for relevant chains.
  * Price impact prediction, expected flow, volatility regimes.
  * MEV risk estimation and mitigation tactics.
  * Cross-chain routing recommendations.

Key actors:

* Data Ingestion & Feature Pipeline
* AI Policy Models (e.g. mailn_policy_v1, mailn_mev_guard_v1)
* Inference Orchestrator and GPU TEE sessions

### 2.3 Oracles & Relayers

* Bridge between on-chain MAILN Controller and AI Plane:

  * Listen to oracle requests from Secret.
  * Send structured AIRequest payloads to Spark.
  * Receive AIResponse from Spark.
  * Push results back into Secret as fulfilled oracle responses.

Key actors:

* MAILN Oracle Request / Response Contracts
* MAILN Relayer Service (off-chain)

---

## 3. Hero Scenario – Continuous Liquidity Steering

High-level narrative:

1. A Secret-based DeFi protocol (e.g., a CLAMM / hybrid orderbook AMM) exposes **LP vaults** managed by a **MAILN Controller Contract**.
2. LPs deposit capital into the vault; their positions are managed algorithmically.
3. MAILN Controller periodically (or event-driven) requests **strategy updates** from the AI Plane.
4. The AI Plane (Spark) uses mempool + on-chain data to infer:

   * Upcoming large trades, flow imbalance, MEV threats.
   * Cross-chain arbitrage risk and routing opportunities.
   * Optimal parameter moves (range widen/narrow, fee tier selection, route splitting, etc.).
5. MAILN Controller receives the AI suggestions (via oracle) and:

   * Validates them against local risk policies.
   * Partially or fully applies changes in a **deterministic**, CPU-TEE-protected way.
6. Users and LPs benefit from:

   * Lower slippage.
   * Reduced MEV extraction.
   * Higher net APR for LP positions.

Confidentiality aspects:

* On-chain, only **high-level effects** are visible (actual position ranges, liquidity amounts).
* Strategy internals and model logic are hidden in:

  * GPU TEE (Spark), and
  * CPU TEE (Secret contracts).

---

## 4. Data Flows

### 4.1 Inputs to the AI Plane (Spark)

MAILN AI models consume multiple data sources:

* **On-chain data (public):**

  * Prices, liquidity depths, pool states on Secret and other chains (Cosmos / EVM).
  * Historical trade volumes and volatility patterns.

* **Mempool data (where observable):**

  * Pending swaps and large orders.
  * Gas price and priority patterns.
  * Sandwich-like transaction patterns (front-run / back-run sequences).

* **Off-chain market data (optional):**

  * Centralized exchange prices.
  * Funding rates, futures basis, implied volatility.

* **Confidential context (optional):**

  * Aggregated liquidity distribution or risk exposures not fully visible on-chain, provided encrypted by Secret contracts.

All sensitive or proprietary features can be:

* Decrypted **only** inside a GPU TEE session.
* Processed by AI models whose weights remain protected.

### 4.2 Outputs from the AI Plane

Typical MAILN AI outputs:

* Suggested **liquidity ranges** per asset pair.
* **Rebalance triggers** based on predicted flow and volatility.
* **Fee tier adjustments** or selection of the optimal fee pool.
* **Cross-chain routes**: split of volume across multiple chains / venues.
* **MEV mitigation actions**:

  * Refrain from rebalancing in certain windows.
  * Change gas / timing strategies.
  * Adjust slippage and constraints to avoid being a predictable target.

The AI output is packaged into a deterministic, machine-readable payload that the MAILN Controller can interpret.

---

## 5. MAILN Controller (Secret Contract)

The MAILN Controller Contract is the **on-chain brain** that:

* Owns the actual LP/vault positions (in a Secret-based DEX / CLAMM / hybrid AMM).
* Exposes functions for:

  * Requesting AI suggestions.
  * Receiving and validating AI responses (via oracle).
  * Executing or partially applying rebalances and configuration changes.

Responsibilities:

1. **Policy & Risk Constraints**

   * Define:

     * Max rebalance frequency.
     * Max capital shift per update (e.g., not more than X% of TVL per hour).
     * Allowed ranges, fee tiers, and venues.
   * Enforce emergency brakes (e.g., disabling MAILN if anomalies detected).

2. **Oracle Interaction**

   * Create new MAILN requests (with relevant state snapshot).
   * Store the status of outstanding requests.
   * Validate the oracle’s AI response (signature, commitment, freshness).

3. **Execution**

   * Translate validated suggestions into deterministic on-chain actions:

     * Update ranges, move liquidity, change routes.
   * Run all logic inside a CPU TEE (SGX / SEV / Nitro), preserving:

     * Key privacy.
     * Strategy-sensitive metadata (aggregation, thresholds).

Determinism is key: the Controller’s response to a given AI payload must be deterministic and consensus-friendly, even if the AI model itself is complex.

---

## 6. MAILN AI Services (Spark Side)

### 6.1 MAILN Core Models

We can think of several logical models:

* **mailn_priceflow_v1**

  * Predicts short-term flow and price impact given mempool + orderbook data.

* **mailn_mev_guard_v1**

  * Classifies windows of time as high / medium / low MEV risk.
  * Suggests “hold / small adjust / aggressive rebalance”.

* **mailn_range_policy_v1**

  * Suggests new liquidity ranges and fee tiers, given predictions from the first two.

Each model can run:

* Inside a GPU TEE for maximum confidentiality and attestation, or
* In a confidential VM environment if GPU TEE is not needed for a particular task.

### 6.2 MAILN AIRequest / AIResponse Shape

Conceptually (pseudostructs):

```
MailnAIRequest {
    id:            UUID,
    model_bundle:  String,    // e.g. "mailn_v1", can imply multiple sub-models
    task:          String,    // e.g. "update_liquidity_policy"
    chain_context: String,    // "secret", "osmosis", ...
    pool_state:    Bytes,     // serialized state snapshot
    flow_features: Bytes,     // mempool + volume + volatility features
    policy_input:  Bytes,     // risk constraints the AI must respect
}

MailnAIResponse {
    id:             UUID,
    status:         "ok" | "error",
    suggested_plan: Bytes,    // structured plan: ranges, fees, splits
    model_commit:   [u8; 32], // hash of model version/bundle
    tee_attestation: Bytes,   // GPU TEE / confidential VM attestation (optional)
}
```

The **relayer** is responsible for encoding/decoding these structures and mapping them into contract calls on Secret.

---

## 7. Control Loop

### 7.1 Periodic or Event-Driven Requests

The MAILN Controller may initiate AI requests:

* On a fixed schedule (e.g., every N blocks or minutes).
* When internal metrics cross thresholds:

  * Volatility spike.
  * TVL change.
  * Large liquidity move elsewhere.
* On explicit operator action (e.g., governance or admin call).

### 7.2 Validation & Execution

Steps:

1. Request created on-chain → event emitted.
2. Relayer sends MailnAIRequest to Spark.
3. Spark runs the MAILN models and returns MailnAIResponse.
4. Relayer calls back into Secret:

   * `fulfill_mailn_response(id, MailnAIResponse)`.

MAILN Controller then:

* Checks:

  * Response ID matches an outstanding request.
  * Model commitment is in the allowed list (if such a registry exists).
  * Optional TEE attestation is valid (if used).
  * Proposed moves satisfy risk and policy constraints.

* Applies:

  * All, or a clipped subset, of the suggested plan.
  * Logs decisions for off-chain monitoring and post-hoc analysis.

If something fails:

* MAILN Controller can:

  * Reject the plan entirely.
  * Fall back to a conservative default strategy.

---

## 8. Security & MEV Considerations

MAILN is designed to **reduce** MEV exposure rather than add new vectors.

Key principles:

* AI decisions are advisory; on-chain policies remain in full control.
* Timing and magnitude of rebalances are constrained by the Controller.
* Mempool analysis on Spark can:

  * Detect suspicious patterns (potential sandwiches, backruns).
  * Signal “do not move now” windows to avoid becoming a predictable target.
* Sensitive model details and thresholds are hidden inside TEEs:

  * GPU TEE (AI Plane) for model logic.
  * CPU TEE (Secret) for thresholds and private aggregation.

To prevent new oracle / AI-based attack surfaces:

* Multiple relayers can be used to avoid single-point-of-failure.
* AI results can be sanity-checked on-chain (bounds, monotonicity, etc.).
* Contracts can require conservative fallback behaviour when confidence is low.

---

## 9. PoC Plan for MAILN

A realistic step-by-step R&D progression:

1. **Simulated Environment (Off-chain Only)**

   * Use historical mempool + on-chain data.
   * Simulate MAILN strategies vs baseline strategies.
   * Evaluate slippage, IL, MEV losses, and LP APR.

2. **Testnet v0 (No TEEs Involved, Just Wire-up)**

   * Secret testnet with a simple CLAMM and MAILN Controller in non-TEE mode.
   * AI Plane runs bare-metal models (no GPU TEE yet).
   * Focus on API, relayer correctness, and determinism of contract behaviour.

3. **Testnet v1 (SGX Controller + Non-TEE AI)**

   * Run MAILN Controller inside SGX as in current Secret.
   * AI Plane still non-TEE.
   * Validate E2E behaviour with real encrypted state.

4. **Testnet v2 (SGX Controller + GPU TEE AI)**

   * Move MAILN models into GPU TEE sessions on Spark.
   * Integrate attestation support for `model_commit` and/or inference sessions.
   * Optionally record model hash and attestation metadata on-chain.

5. **Advanced Experiments**

   * Multi-chain MAILN (Secret + external DEXes via IBC/EVM bridges).
   * More complex strategies (range replication, dynamic fee curves, risk-aware rebalancing).

---

## 10. Position in the Overall Repo

Within this repository, MAILN is:

* The **flagship / hero case** demonstrating:

  * How the **TEE-agnostic node** can safely host sensitive DeFi logic in CPU TEEs.
  * How the **Confidential AI Plane** on NVIDIA Spark can add real, measurable value without joining consensus.

Other use cases (confidential copilot, generic risk engines, etc.) may be added later, but MAILN is the primary, concrete story that ties everything together.

---
