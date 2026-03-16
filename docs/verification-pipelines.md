=== FILENAME: docs/verification-pipelines.md ===

Verification Pipelines

This document details the three primary verification pipelines of the OCVD: KYC, Credit, and Compliance. Each pipeline is a deterministic sequence of operations that transforms raw input into a verifiable output.

Common Pipeline Structure

Every pipeline follows a standard interface:

1. Input Validation: Ensures the incoming data payload is well-formed and contains all required fields.
2. Rule Loading: Fetches the governance-approved ruleset for this specific pipeline.
3. Data Verification: Executes the core verification logic against the provided data.
4. Proof Generation: If verification is successful, the pipeline triggers the creation of a zk-proof.
5. Output Packaging: The final result, including the proof and public outputs, is packaged into a standardized JobResult structure.

6. KYC Pipeline

Purpose: To verify a user's identity and produce a zk-KYC proof attesting to specific attributes without revealing the source documents.

Inputs:

· user_id: A unique identifier.
· document_type: e.g., "passport", "drivers_license".
· document_front: Image data (or hash pointer to encrypted storage).
· document_back (optional): Image data.
· required_attributes: e.g., ["age>18", "nationality==US"].

Verification Steps:

1. Document Authenticity: Uses image processing and ML models to check for tampering, holograms, and MRZ (Machine Readable Zone) validity.
2. Data Extraction: Extracts key fields: full name, date of birth, nationality, document number, expiration date.
3. Liveness Check (if applicable): If a selfie is provided, it is compared to the photo on the document.
4. Attribute Check: Compares the extracted data against the required_attributes. For example, calculates age from date of birth and checks if it is over 18.

Output & Proof Generation:

· Success: The pipeline invokes the zk-kyc circuit. Private inputs are the document data. Public outputs are a hash commitment to the user's identity and a boolean vector indicating which required_attributes were satisfied. The final output includes the zk-proof.
· Failure: A structured error code is returned (e.g., ERR_DOC_EXPIRED, ERR_FACE_MISMATCH). No proof is generated.

2. Credit Pipeline

Purpose: To assess a user's financial health and produce a zk-credit proof for a specific risk tier or metric.

Inputs:

· user_id: A unique identifier.
· data_source: e.g., "bank_statement", "tax_return", "income_slip".
· financial_data: Structured data (e.g., JSON of transactions, PDF of a statement).
· request_type: e.g., "tier", "income_gt", "dti_lt".

Verification Steps:

1. Data Ingestion: Parses the financial data into a standard internal format (e.g., list of income and expense entries).
2. Metric Calculation: Calculates key metrics based on the request_type:
   · income_gt: Sums all income sources over a period.
   · dti_lt: Calculates total monthly debt payments divided by gross monthly income.
   · tier: Runs a more complex scoring model that outputs one of five pre-defined risk tiers (e.g., Tier 1: Prime, Tier 5: Subprime).
3. Threshold Comparison: Compares the calculated metric against the governance-defined thresholds for the specific request.

Output & Proof Generation:

· Success: The pipeline invokes the zk-credit circuit. Private inputs are the raw financial data. Public outputs are the user's ID commitment, the calculated tier or a boolean result (e.g., income>50000), and a timestamp.
· Failure: An error code is returned (e.g., ERR_INSUFFICIENT_DATA, ERR_NEGATIVE_INCOME).

3. Compliance Pipeline

Purpose: To screen users against global sanctions lists, watchlists, and internal risk indicators.

Inputs:

· user_id: A unique identifier.
· entity_name: Full name of the individual or business.
· date_of_birth (optional): For individuals.
· jurisdiction: Country of residence or operation.
· check_type: e.g., "sanctions", "pep", "adverse_media".

Verification Steps:

1. List Loading: Loads the latest versions of required watchlists from a governance-approved, secure source.
2. Fuzzy Matching: Performs intelligent matching against the lists, accounting for name variations, misspellings, and date proximity.
3. Risk Scoring: A match is not a binary outcome. The system assigns a risk score based on the confidence of the match and the severity of the list.
4. Rule Application: Applies governance rules to the risk score. For example: "If sanctions match score > 90%, reject. If PEP match score > 70%, flag for manual review."

Output & Proof Generation:

· Success (Clear): The pipeline generates a proof of non-matching, essentially proving that the entity is not on any of the lists.
· Success (Flagged): The pipeline may generate a proof that a manual review is required. This proof would not reveal the nature of the match, only that the automated process was inconclusive.
· Failure: A positive, high-confidence match on a sanctions list results in an immediate rejection with a ERR_SANCTIONS_HIT code.

Triggering zk-Proof Generation

Proof generation is not an afterthought; it is an integral part of each pipeline. After the verification logic successfully concludes, the pipeline automatically formats its inputs for the zk-circuit. The specific circuit used is determined by the request_type. This tight coupling ensures that a proof is only generated for a successfully verified claim, maintaining the system's integrity.
