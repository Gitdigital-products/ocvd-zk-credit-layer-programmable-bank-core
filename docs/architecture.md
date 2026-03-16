=== FILENAME: docs/architecture.md ===

Architecture: OCVD Core

This document details the high-level architecture of the Off-Chain Verification Daemon (OCVD). The system is designed for modularity, security, and deterministic operation under active governance.

High-Level Architecture

The OCVD is structured into several distinct layers, each with a specific, non-overlapping responsibility.

```
+---------------------+      +-----------------------------------------+
|   Interface Layer   |      |            Core Logic Layer              |
|---------------------|  --> |-----------------------------------------| --> [On-Chain Registries]
| - REST API          |      | +------------------+ +----------------+ |
| - Message Queue (NATS)|    | | Scheduler        | | Event Bus      | |
+---------------------+      | +------------------+ +----------------+ |
                             |                      |                   |
                             |                      v                   |
                             | +------------------+ +----------------+ |
                             | | Pipeline Workers | | Registry Writer| |
                             | | (KYC, Credit,    | | (Adapter Layer)| |
                             | |  Compliance)     | +----------------+ |
                             | +------------------+                    |
                             |                      |                   |
                             |                      v                   |
                             |            +------------------+          |
                             |            |  zk-Proof Core   |          |
                             |            +------------------+          |
                             +-----------------------------------------+
                                           |
                                           v
+---------------------+      +----------------------------+
|   Storage Layer     |      |   Governance & Audit Layer  |
|---------------------|      |-----------------------------|
| - Audit Logs (DB)   |      | - On-Chain Governance       |
| - Temp Artifacts    |      | - Signed Audit Trail        |
+---------------------+      +-----------------------------+
```

Daemon Lifecycle

1. Initialization: The daemon starts by loading its configuration and establishing connections to the blockchain node, databases, and message queue. It then fetches the latest governance rules and registry addresses from the chain, caching them locally.
2. Idle & Listening: The core enters a listening state, awaiting new jobs from the API or message queue.
3. Job Processing: A job is received and placed into the appropriate work queue.
4. Worker Execution: A worker picks up the job, loads the current rules, executes the pipeline, and generates a proof.
5. Registry Submission: The result is handed to the registry writer for on-chain submission.
6. Audit Logging: A cryptographically signed log entry for the entire process is written to the audit store.
7. Completion: The worker becomes idle, ready for the next job.

Worker Pipelines

The OCVD utilizes specialized, single-purpose worker pools.

· KYC Pipeline: Processes identity documents. It verifies document authenticity, extracts required fields, and outputs a zk-KYC proof attesting to the user's age, nationality, or status (e.g., accredited investor) without revealing the document itself.
· Credit Pipeline: Analyzes financial data. It ingests bank statements or income proofs, calculates risk metrics (e.g., DTI ratio, income stability), and produces a zk-credit proof for a specific risk tier.
· Compliance Pipeline: Monitors for sanctions and illicit activity. It checks user data against global watchlists and internal risk flags, producing a proof of compliance or a flagged status for further review.

zk-Proof Generation Flow

The proof generation is a critical, resource-intensive process.

1. Input Compilation: The worker compiles all verified data and the specific proof request (e.g., "prove income > 50k") into a structured input.
2. Circuit Invocation: The worker calls the appropriate zk-circuit (e.g., credit_check.circom) with the private inputs and public parameters.
3. Witness Generation: The circuit generates a witness—an assignment of all signals that satisfy the circuit's constraints.
4. Proof Creation: The proving key is used to generate a zero-knowledge proof from the witness.
5. Output: The proof and the public outputs (e.g., the commitment hash, the risk tier) are returned to the worker.

Registry Writer Flow

The registry writer ensures that on-chain state remains consistent with off-chain verification.

1. Transaction Construction: The writer receives the proof and public outputs. It constructs a transaction for the specific on-chain registry contract (e.g., CreditRegistry.setCreditTier(user, tier, proof)).
2. Signing & Submission: The transaction is signed using the OCVD's secure, permissioned key. It is then broadcast to the blockchain network.
3. Confirmation Monitoring: The writer monitors the transaction's status, waiting for a sufficient number of block confirmations.
4. Error Handling: If the transaction fails (e.g., due to gas issues or reorg), the writer places it in a retry queue with exponential backoff. Persistent failures trigger an alert.
5. Audit Update: The final on-chain transaction hash is appended to the audit log, creating a verifiable link between the off-chain process and the on-chain result.

Governance Enforcement Model

The OCVD does not trust its own local configuration. Before every critical operation—loading a rule, writing to a registry—it performs a lightweight check against the current on-chain governance state. This "check before act" model ensures that even if a daemon instance is compromised, it cannot execute stale or malicious rules without a corresponding on-chain update.
