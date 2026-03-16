=== FILENAME: docs/governance.md ===

Governance: Badge-Based Control & Deterministic Enforcement

The OCVD is not just a piece of software; it is a governed entity. Its rules, permissions, and even its core logic are subject to an on-chain governance process. This document outlines the governance model and how it is enforced deterministically by the daemon.

Badge Authority & Contributor Trust

Governance in the OCVD ecosystem is built on the concept of non-transferable achievement badges. These badges represent trust, expertise, and commitment to the network.

· Core Contributor Badge: Held by the initial development team and key architects. Grants the ability to propose changes to the core daemon codebase and the fundamental governance parameters.
· Compliance Steward Badge: Held by entities with legal and regulatory expertise. This badge is required to propose updates to sanctions lists, compliance rules, and risk thresholds.
· Security Auditor Badge: Held by individuals or firms that have successfully completed a security audit of the system. They have veto power over upgrades that could introduce critical vulnerabilities.
· Node Operator Badge: Held by entities running the OCVD instance. They can signal for non-critical updates and participate in monitoring working groups.

This badge-based system ensures that power is distributed among specialized, trusted parties, preventing any single entity from controlling the entire system.

Rule-Change Proposals

All changes to the OCVD's operational parameters are initiated through an on-chain proposal process.

1. Submission: A badge holder submits a proposal transaction to the Governance contract. This proposal contains the new rule set or a hash of the new daemon code.
2. Voting Period: A voting period begins. Only other badge holders can vote, with voting power potentially weighted by badge type (e.g., Security Auditors have more weight on security-related proposals).
3. Quorum & Approval: The proposal passes if it achieves a quorum (e.g., >50% of eligible voters) and a supermajority approval (e.g., >66% of votes cast).
4. Timelock & Execution: Once passed, the change is queued in a Timelock contract. After a mandatory delay (e.g., 48 hours), the change can be executed, updating the on-chain governance state.

Deterministic Governance Enforcement

The OCVD does not blindly trust its own configuration. It enforces governance deterministically through a "verify, then trust" pattern.

```
[Governance Contract] <--- (1. Rule Update) --- [Governance Process]
         |
         | (2. State Read)
         v
    [OCVD Daemon] <--- (3. Fetch for Every Job) --- [Local Cache]
         |
         | (4. Execute with Verified Rules)
         v
   [Verification Pipeline]
```

1. On-Chain Source of Truth: All governance parameters (registry addresses, allowed methods, risk thresholds, KYC attribute requirements) are stored on-chain in the Governance contract.
2. Local Cache: For performance, the OCVD maintains a local, in-memory cache of these parameters, refreshed on a short interval (e.g., every 60 seconds).
3. Verification on Use: Before a worker begins a pipeline, it checks the last_updated timestamp of its cached rules against the current on-chain timestamp. If the cache is stale, it forces a refresh. More critically, before a registry write, the registry-writer performs a direct on-chain check to ensure the target contract and method are still permitted. This "double-check" prevents the use of stale or maliciously altered local data.
4. Execution: Only after the rules have been cryptographically verified against the on-chain state does the pipeline proceed.

This deterministic enforcement model means that a compromised or misconfigured OCVD instance cannot cause damage. Any attempt to use an old rule or an unauthorized registry will be rejected by the worker's own pre-flight checks.

Audit-Safe Event Logs

Every governance-related action and enforcement check is logged in a manner that is both transparent and tamper-evident.

· On-Chain Events: All proposals, votes, and executions emit standard EVM events, providing a public, immutable record of governance activity.
· Off-Chain Audit Trail: The OCVD logs every instance of rule enforcement. A critical log entry includes:
  · Job ID
  · Governance Snapshot ID (e.g., the block number or timestamp of the rules used)
  · Rule Hash (a hash of the specific rule that was applied)
  · Enforcement Result (e.g., "User passed KYC check for attribute X")
  · OCVD Signature (a signature from the daemon's key over the log entry)
