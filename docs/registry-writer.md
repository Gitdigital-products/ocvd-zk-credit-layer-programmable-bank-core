=== FILENAME: docs/registry-writer.md ===

Registry Writer: On-Chain Communication

The Registry Writer is the OCVD's dedicated module for secure, reliable, and governance-controlled communication with the blockchain. It acts as the bridge between the stateless verification pipelines and the persistent on-chain state.

Registry Adapters

The writer uses a system of adapters to interact with different smart contracts. This design allows for easy extensibility as new registry types are added.

· CreditRegistryAdapter: Formats and submits transactions to the CreditRegistry contract. Handles functions like setCreditTier.
· KYCRegistryAdapter: Interfaces with the KYCRegistry to store commitments to verified identities and the hash of the proof.
· ComplianceRegistryAdapter: Updates the status of a user (e.g., STATUS_CLEARED, STATUS_FLAGGED) on the ComplianceRegistry.

Each adapter is responsible for:

1. ABI Encoding: Correctly encoding the function calls and their arguments.
2. Gas Estimation: Estimating the gas required for the transaction.
3. Error Translation: Converting blockchain revert reasons into meaningful internal error codes.

On-Chain Write Flow

The write flow is designed for maximum reliability and auditability.

1. Request Reception: The Registry Writer listens to a specific internal queue for WriteRequest messages. A WriteRequest contains the job ID, the target registry, the method to call, the encoded arguments (including the zk-proof), and the user's on-chain address.
2. Pre-Write Validation:
   · Governance Check: It verifies that the target registry address and method are still authorized by the current on-chain governance. This prevents writing to a deprecated or malicious contract.
   · Nonce Management: It retrieves the next expected nonce for the OCVD's signer account to ensure transactions are processed in order.
3. Transaction Construction & Signing: The adapter constructs the raw transaction, estimates gas, adds a reasonable gas tip (priority fee), and signs it using a Hardware Security Module (HSM) or a secure key management service. The private key never exists in memory.
4. Submission & Monitoring: The signed transaction is broadcast to the blockchain network via the connected RPC endpoint. The writer then enters a monitoring loop, checking the transaction receipt.
5. Confirmation & Finalization: Once the transaction is confirmed (e.g., 12 block confirmations), the writer extracts the transaction hash.
6. Audit Logging: The final transaction hash, along with the job ID and a timestamp, is written to the audit log, creating a permanent, verifiable link between the off-chain job and the on-chain result.

Error Handling

The Registry Writer employs a sophisticated retry and error-handling strategy.

· Transient Failures (e.g., network timeouts, gas price too low): The transaction is placed back in the queue for a retry with exponential backoff (e.g., 1s, 2s, 4s, ... up to a max).
· Blockchain Reverts (e.g., require statement failed): The error is parsed from the revert reason. This is a critical failure. The writer logs the detailed error and moves the job to a Dead Letter Queue (DLQ) for immediate manual review. This indicates a logic mismatch between the OCVD and the contract.
· Nonce Errors (e.g., incorrect nonce): This is often a sign of a stuck transaction or a restart. The writer will resync its nonce from the chain and retry the transaction.
· Persistent Failures: After a maximum number of retries, the job is moved to the DLQ, and an alert is triggered for the operations team.

Governance-Controlled Writes

The writer is stateless regarding its permissions. Before every write, it must consult the on-chain source of truth.

```pseudocode
function isRegistryWriteAllowed(registry_address, method_signature):
    allowed_registries = governance_contract.getAllowedRegistries()
    if registry_address not in allowed_registries:
        return false
    allowed_methods = governance_contract.getAllowedMethods(registry_address)
    return method_signature in allowed_methods
```

This "check-before-write" pattern ensures that even if the writer's local cache is outdated or corrupted, it cannot perform an action that hasn't been explicitly approved by the governing body.

Audit Logging

Every action taken by the Registry Writer is logged in an immutable, append-only audit database. The log entry includes:

Field Description
job_id The internal ID of the originating verification job.
timestamp The time of the write attempt (UTC).
registry The target registry contract address.
method The smart contract method called.
tx_hash The final on-chain transaction hash (if successful).
status SUCCESS, RETRYING, FAILED.
error Detailed error message or revert reason (if any).
signed_payload A hash of the signed transaction for non-repudiation.

This audit trail is crucial for debugging, proving compliance to regulators, and providing a complete picture of the system's operation.
