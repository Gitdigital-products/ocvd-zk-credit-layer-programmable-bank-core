=== FILENAME: docs/zk-credit.md ===

zk-Credit: Model, Proofs, and Risk Tiers

The zk-credit module is the core of the OCVD's privacy-preserving credit assessment capabilities. It allows users to prove their creditworthiness to a smart contract or a third party without revealing their underlying financial data.

The zk-Credit Model

The model is based on the principle of selective disclosure through risk tiers. Instead of revealing a continuous credit score, the OCVD generates proofs that a user falls into one of a set of pre-defined, governance-controlled risk tiers.

· Tier 1 (Prime): Lowest risk. Indicates high income, low debt, and excellent financial history.
· Tier 2 (Near-Prime): Good financial health with minor blemishes or slightly higher leverage.
· Tier 3 (Standard): Average risk profile.
· Tier 4 (Subprime): Higher risk, indicating significant debt or income instability.
· Tier 5 (High-Risk/Non-qualifying): Does not meet minimum criteria.

This tiered approach simplifies on-chain logic and provides clear, actionable signals for programmable banking applications (e.g., "Only Tier 1 and 2 users can borrow at this rate").

Proof Inputs

The zk-credit circuit accepts two types of inputs:

1. Private Inputs (witness):
   · Raw transaction data (categorized).
   · Income sources and amounts.
   · Recurring debt payments.
   · The user's private key, used to generate a unique commitment.
2. Public Inputs (shared):
   · The user's on-chain identity commitment (a hash of their identity).
   · The governance-approved risk tier parameters (e.g., TIER_1_MIN_INCOME, TIER_1_MAX_DTI). These are hardcoded into the circuit's verification key for the specific proving session.
   · A timestamp or nonce to prevent replay attacks.

Proof Outputs

The circuit outputs a single, compact zero-knowledge proof and a set of public outputs:

· user_commitment: A hash that uniquely and publicly identifies the user without revealing their identity.
· risk_tier: The integer (1-5) representing the user's assessed risk level.
· as_of_date: The date of the latest financial data used for the assessment, providing temporal context.
· proof: The zero-knowledge proof itself, which can be verified on-chain by a Verifier contract.

The Proof Generation Process

1. Data Aggregation: The Credit Pipeline aggregates the user's financial data, calculating the necessary inputs for the circuit.
2. Circuit Execution:
   · The circuit takes the private data and the public parameters.
   · It performs the same calculations as the off-chain pipeline (income sum, DTI ratio).
   · It compares the results against the public tier parameters.
   · It assigns the user to the correct tier.
3. Commitment Binding: The circuit hashes the user's private key or a secret with the as_of_date to create the user_commitment. This binds the proof to a specific user and time period.
4. Proof Creation: The proving key is used to generate the zk-proof, which attests that: "Given the public parameters, I know a set of private financial data that, when processed according to the rules of this circuit, results in this risk_tier and user_commitment."

Selective Disclosure

The tier-based system is a form of selective disclosure. A user can generate different proofs for different purposes:

· To borrow stablecoins, they might generate a proof showing they are Tier 1 or 2.
· To vote in a governance proposal restricted to "qualified investors," they might generate a proof that their income is above a certain threshold and they are an accredited investor (a combination of zk-credit and zk-KYC).
· They can prove they are not in a high-risk tier without revealing their exact tier.

On-Chain Submission

Once generated, the proof and its public outputs are sent to the CreditRegistry smart contract.

```solidity
// Pseudocode for CreditRegistry
function submitCreditTier(
    bytes32 _userCommitment,
    uint8 _riskTier,
    uint256 _asOfDate,
    bytes calldata _proof
) external onlyAuthorized(OCVD_ADDRESS) {
    require(verifier.verifyCreditTierProof(_userCommitment, _riskTier, _asOfDate, _proof), "Invalid Proof");
    userCreditTiers[_userCommitment] = CreditTier(_riskTier, _asOfDate, block.timestamp);
    emit CreditTierUpdated(_userCommitment, _riskTier, _asOfDate);
}
```

The contract first verifies the proof using a trusted Verifier. Only upon successful verification is the user's on-chain credit tier updated. This ensures that the on-chain state is always a direct and verified reflection of the off-chain truth.
