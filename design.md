# SNDL: Post-Quantum Secure Data Migration & Management Framework

> **Goal:** Eliminate the “store-now, decrypt-later” threat by migrating entire datastores and file systems to quantum-safe encryption, while providing Role-Based Access Control, OCR-driven classification, and tamper-evident auditing.

---

## 1. Introduction

Classical public-key schemes (RSA, ECC) will be broken by future quantum computers, exposing archived data at rest. SNDL (Secure Next-gen Data Layer) delivers:

- **Post-Quantum Cryptography (PQC):**  
  - **Key Encapsulation:** CRYSTALS-Kyber  
  - **Signatures:** CRYSTALS-Dilithium  
  - **AEAD Payload Cipher:** Lightweight PQ-AEAD (e.g. Ascon-128a)  
- **OCR Classification:** Tesseract-based text extraction to tag scanned documents by sensitivity  
- **RBAC Enforcement:** Fine-grained user-and-role policies at API level  
- **Tamper-Evident Audit:** HMAC-protected Merkle-tree append-only logs  

By contributing under the **OpenSSF** umbrella (Scorecard checks, Sigstore signing, CLA governance), SNDL becomes a community-reviewable, supply-chain-trusted project.

---

## 2. Use-Case Scenarios

1. **Mass Data Migration**  
   - Migrate a filesystem or database in-place.  
   - Chunk files, encrypt under TPM-derived SK, wrap SK with Kyber KEM.  
   - Log each file migrator event.

2. **On-Demand Access**  
   - Decrypt individual files by authorized users.  
   - Decapsulate SK via TPM, verify PQD signatures, deliver plaintext.

3. **OCR-Driven Ingestion**  
   - Scan images/PDFs, OCR→classify→encrypt.  
   - Attach sensitivity tags in metadata for later policy enforcement.

4. **RBAC Policy Management**  
   - CRUD on roles/actions via admin API.  
   - Every change signed, audited, tied to admin ID.

5. **Audit & Reporting**  
   - Query append-only Merkle tree by date/user.  
   - Produce root hash + inclusion proofs + signed report.

6. **Key Rotation & Revocation**  
   - Generate new Kyber keypair in TPM/HSM.  
   - Mark old keys revoked in metadata store.  
   - Publish rotation event with Dilithium signature.

---

## 3. Logical Architecture

### 3.1 API & UI Layer
- **REST/gRPC Endpoints** for migration, encrypt, decrypt, policy, audit.  
- **CLI Tool** for batch jobs.  
- **Web Dashboard** (React/Vue) for admins & auditors.

### 3.2 Core Services
- **PQC Crypto Module**  
  - Implements Kyber KEM, Dilithium sig, PQ-AEAD encryption/decryption.  
  - Key-encapsulation → SK; AEAD payload → ciphertext; post-encrypt signature.

- **RBAC Service**  
  - Central policy engine: roles → allowed actions, resource scopes.  
  - Integrated into every API call; returns user-ID and role context.

- **OCR Processor**  
  - Tesseract wrapper + language models.  
  - Outputs raw text + confidence + sensitivity classification.

- **Audit Module**  
  - Maintains Merkle tree of event leaves:  
    ```
    leaf = HMAC(SK_audit, eventType ∥ resourceID ∥ userID ∥ timestamp)
    ```
  - Signed root published periodically.

### 3.3 Integration Layer
- **TPM/HSM Driver**  
  - Secure key storage: Kyber master keys + SK derivation.  
- **Identity Connectors**  
  - LDAP / OIDC for user authentication & group sync.  
- **Storage Adapters**  
  - Filesystem (NFS/CIFS), S3-compatible object store, SQL/NoSQL metadata DB.

---

## 4. Development View

/src
├─ core/
│ ├─ crypto/ # PQC primitives (Kyber, Dilithium, PQ-AEAD)
│ ├─ rbac/ # Policy definitions, enforcement API
│ ├─ ocr/ # Tesseract integration, classification logic
│ ├─ audit/ # Merkle‐tree, HMAC services, storage
│ └─ tpm/ # TPM/HSM abstraction layers
├─ api/
│ ├─ grpc/ # .proto + server stubs
│ └─ rest/ # Controllers, routing, serializers
├─ cli/ # Batch & admin CLI commands
├─ web/ # Dashboard: React/Vue + REST client
├─ integration/ # LDAP/OIDC, storage adapters, KMS connectors
├─ tests/
│ ├─ unit/
│ ├─ integration/
│ └─ fuzz/ # PQC fuzz targets (ciphertext, keys)
└─ docs/ # HLD/LLD, onboarding, OpenSSF compliance guide


- **Modular packages** for testable, reusable components  
- **CI/CD pipelines** with OpenSSF Scorecard, Sigstore, fuzz testing, static analysis

---

## 5. Process & Concurrency

- **Migration Daemon**  
  1. Enumerate files → RBAC check  
  2. TPM decapsulate SK → chunk & encrypt in worker pool  
  3. PQ‐KEM wrap SK → store metadata  
  4. Audit-log leaf per file

- **On-Demand API**  
  1. Authenticate & RBAC check  
  2. Fetch ciphertext + {c_kem, sigs}  
  3. Kyber decapsulate → SK  
  4. PQ-AEAD decrypt + verify Dilithium sig per chunk  
  5. Return plaintext / error on tamper

- **OCR Pipeline**  
  1. File arrival trigger  
  2. OCR → classify sensitivity  
  3. RBAC check for “encrypt as level”  
  4. Encrypt + sign + store  

- **Scheduler Tasks**  
  - **Key Rotation:** Generate new Kyber keypair → revoke old → audit  
  - **Audit Root Publication:** Compute & sign Merkle root at fixed intervals  
  - **Health Checks:** Verify agent heartbeat, storage availability  

---

## 6. Physical Deployment

### 6.1 Hybrid On-Prem & Cloud

1. **DMZ Layer**  
   - Public LB + WAF/CDN → TLS termination, JWT auth.

2. **On-Prem Cluster**  
   - Kubernetes worker nodes host **Migration** & **OCR** agents.  
   - On-Prem HSM/TPM + Vault for key material.  
   - Encrypted NAS for source data.

3. **Cloud Layer**  
   - Managed K8s (AKS/EKS/GKE) for **Core API**, **Crypto**, **RBAC**.  
   - Cloud KMS/HSM for Kyber master keys.  
   - Object Store (S3) for ciphertext + metadata.  
   - Managed SQL/NoSQL DB for audit & metadata.  
   - SIEM ingest for logs.

### 6.2 Fully-Cloud Zero-Trust

- **API Gateway + WAF:** AuthN/Z, rate limits.  
- **Internal LB → Auto-Scaling Pods:** Crypto, OCR, RBAC microservices.  
- **Cloud HSM/KMS:** Kyber key operations, attestation.  
- **Encrypted Object Store + DB:** Ciphertext + Merkle logs.  
- **Private/Public Subnets + NAT:** Zero-trust network segments.

---

## 7. Security & Hardening

- **In-Memory Zeroization:** Automatic scrub of SK and plaintext buffers after use.  
- **Signature Verification:** Reject any tampered ciphertext immediately.  
- **RBAC Enforcement:** All endpoints require token + policy check; userID embedded in every audit leaf.  
- **Supply-Chain Protections:**  
  - OpenSSF Scorecard (SAST, dependency checks).  
  - Sigstore for signing container images & releases.  
  - Public issue tracker, CLA policy, code reviews.

---

## 8. Community & Contribution

- **OpenSSF Alignment:**  
  - Scorecard passes (SCA, SBOM, vuln scanning).  
  - Sigstore signing of binaries.  
  - Clear governance and roadmap.  
- **Documentation:**  
  - Onboarding guide, HLD/LLD, API reference.  
- **Extensibility:**  
  - Plugin-style OCR engines, custom PQ-AEAD ciphers.  
  - Additional identity backends (e.g. SCIM, SAML).  

> **Conclusion:** SNDL provides a complete, modular, and open-source framework to migrate, protect, and manage sensitive data with post-quantum resilience, fine-grained access control, and cryptographic accountability—positioned for community collaboration under OpenSSF governance.  
