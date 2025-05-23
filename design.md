# SNDL: Post-Quantum Secure Data Migration Framework

## Introduction

The SNDL framework addresses the critical “store‑now, decrypt‑later” threat posed by emergent quantum computers that can retroactively break classical encryption. By migrating entire datastores and file‑systems to **quantum‑safe cryptography**, SNDL ensures confidentiality, integrity, and accountability of both live and archived data.

Key features:

* **Post‑Quantum Primitives**: NIST‑standard CRYSTALS‑Kyber (KEM) for key encapsulation, CRYSTALS‑Dilithium for digital signatures, and a lightweight PQ‑AEAD cipher (e.g. Ascon‑128a) for payload encryption.
* **OCR‑Driven Classification**: Tesseract‑based pipeline to extract and classify text from scanned documents before encryption.
* **Role‑Based Access Control (RBAC)**: Fine‑grained policies at the API layer bind every operation to a user identity.
* **Tamper‑Evident Audit Trail**: HMAC‑protected, Merkle‑tree–backed logs record each encrypt/decrypt/policy event as an append‑only proof.
* **Modular, Cloud‑Native Architecture**: Containerized microservices (REST/gRPC + Web UI), orchestrated on Kubernetes with LDAP/OIDC SSO, on‑prem or cloud HSM, encrypted object storage, and managed audit databases.

This open‑source design aligns with OpenSSF best practices—Scorecard checks, Sigstore signing, and clear governance—to enable community review, contribution, and rapid adoption.

---

## 4+1 View Architecture

### 1. Scenarios (Use Case View)

| ID  | Name                              | Actors                  | Description                                                      |
| --- | --------------------------------- | ----------------------- | ---------------------------------------------------------------- |
| UC1 | Mass Data Migration               | Migration App, Admin    | In‑place, chunked encryption of entire filesystems/databases.    |
| UC2 | On‑Demand File Access             | Client App, User        | Decrypt individual files under RBAC policies, with tamper check. |
| UC3 | OCR‑Driven Classification & Store | Scanner Service, Carols | Scan, classify, and encrypt images/docs as per sensitivity.      |
| UC4 | RBAC Policy Management            | Security Admin          | Create/update roles and permissions with auditable actions.      |
| UC5 | Audit & Compliance Reporting      | Auditor UI              | Generate Merkle‑rooted proofs and signed reports.                |
| UC6 | Key Rotation & Revocation         | Scheduler, TPM, Ops     | Rotate Kyber keys, revoke old keys, and log events securely.     |

---

### 2. Logical View

```mermaid
flowchart TB
    subgraph API Layer
      REST["REST Endpoints"]
      gRPC["gRPC Services"]
      CLI["CLI Tool"]
      UI["Web Dashboard"]
    end
    subgraph Core Engine
      PQC["PQC Crypto (Kyber, Dilithium, PQ-AEAD)"]
      RBAC["RBAC Service"]
      OCR["OCR Processor"]
      Audit["Audit Module (Merkle Tree)"]
    end
    subgraph Integration
      LDAP["LDAP/OIDC"]
      HSM["HSM / TPM"]
      Store["Storage Adapters\n(S3, FS, DB)"]
    end
    API Layer --> Core Engine
    Core Engine --> Integration
```

---

### 3. Development View (Module Structure)

```
/src
 ├─ core/
 │   ├─ crypto/    # Kyber KEM, Dilithium, PQ-AEAD wrappers
 │   ├─ rbac/      # Policy engine, storage, evaluation
 │   ├─ ocr/       # Tesseract bindings, classifiers
 │   └─ audit/     # Merkle-tree, log writers, HMAC
 ├─ api/
 │   ├─ rest/      # Controllers, DTOs, routers
 │   └─ grpc/      # .proto definitions, server impl
 ├─ cli/          # Command-line tool
 ├─ web/          # React or Vue dashboard
 ├─ integration/  # LDAP/OIDC, HSM drivers, store adapters
 ├─ tests/        # Unit, integration, fuzz tests
 └─ docs/         # HLD, user guide, OpenSSF playbook
```

---

### 4. Process View

```mermaid
sequenceDiagram
    participant Migration as Migration Daemon
    participant MemMgr    as Memory Manager
    participant RBAC      as RBAC Service
    participant TPM       as TPM
    participant Crypto    as PQC Engine
    participant Store     as Storage
    participant Audit     as Audit Service

    Migration->>RBAC: Check migrate permission (userID)
    RBAC-->>Migration: Permit
    Migration->>MemMgr: Zeroize buffers
    MemMgr-->>Migration: Ack
    loop each file
      Migration->>TPM: Decapsulate session key SK
      TPM-->>Migration: SK
      loop chunk in file
        Migration->>Crypto: PQ-AEAD encrypt(chunk, SK)
        Crypto-->>Migration: ciphertext
        Migration->>Crypto: Sign(ciph, metadata)
        Crypto-->>Migration: sig
        Migration->>Store: write {ciphertext, sig}
      end
      Migration->>Crypto: KEM-encapsulate(SK)
      Crypto-->>Migration: {c_kem, wrappedSS}
      Migration->>Store: write session meta
      Migration->>Audit: Log migrate event
      Audit-->>Migration: Ack
      Migration->>MemMgr: Zeroize SK & buffers
      MemMgr-->>Migration: Ack
    end
```

---

### 5. Physical View

#### 5.1 Hybrid On‑Prem + Cloud

```mermaid
flowchart TB
  subgraph DMZ
    CDN["CDN / WAF"]
    LB["Load Balancer"]
  end
  subgraph OnPrem
    K8sOn["K8s Agents (Migration/OCR)"]
    HSMOn["On‑Prem HSM"]
    Vault["Vault (HSM‑backed KMS)"]
    LDAP["LDAP / OIDC"]
    NAS["Encrypted NAS"]
  end
  subgraph Cloud
    K8sCloud["K8s Core Services (API/Crypto)"]
    KMS["Cloud HSM / KMS"]
    Store["Object Storage"]
    DB["Managed Audit DB"]
  end
  Client["CLI / Web Clients"] --> CDN --> LB
  LB --> K8sOn
  LB --> K8sCloud
  K8sOn --> HSMOn & Vault & NAS
  K8sCloud --> KMS & Store & DB
  Vault --> LDAP
  KMS -.-> LDAP
  Store --> DB
```

#### 5.2 Fully‑Cloud (Zero‑Trust)

```mermaid
flowchart TB
  subgraph Internet
    Clients["CLI / Web"]
    GW["API Gateway + WAF"]
  end
  subgraph VPC
    ALB["Internal ALB"]
    Pods["Crypto & OCR Pods"]
    RBACSvc["RBAC Service"]
    KMS["Cloud HSM / KMS"]
    ObjStore["Encrypted Object Store"]
    DB["Audit + Meta DB"]
    SIEM["SIEM / Monitoring"]
  end
  Clients --> GW --> ALB --> Pods
  Pods --> RBACSvc & KMS & ObjStore & DB
  DB --> SIEM
  ObjStore --> SIEM
```

---

## Sequence Diagrams (Detailed Operations)

#### UC1: Mass Data Migration

```mermaid
sequenceDiagram
    title UC1: Mass Data Migration
    participant App    as Migration App
    participant RBAC   as RBAC Service
    participant TPM    as TPM Module
    participant Crypto as PQC Engine
    participant Store  as Storage Adapter
    participant Audit  as Audit Service
    participant Mem    as Memory Manager

    App->>RBAC: Can migrate /data/*? (userID)
    RBAC-->>App: Permit
    App->>Mem: Zeroize buffers
    Mem-->>App: Ack
    loop each file
      App->>TPM: Decapsulate → SK
      TPM-->>App: SK
      loop each chunk
        App->>Mem: Load chunk
        Mem-->>App: Buffer Ready
        App->>Crypto: EncryptPQAEAD(chunk, SK)
        Crypto-->>App: ciphertext
        App->>Crypto: SignDilithium(ciphertext ∥ metadata)
        Crypto-->>App: sig
        App->>Store: write {ciphertext, sig}
      end
      App->>Crypto: KEMEncapsulate(SK)
      Crypto-->>App: {c_kem, wrappedSS}
      App->>Store: write session metadata
      App->>Audit: Log migrate + MerkleLeaf
      Audit-->>App: Ack
      App->>Mem: Zeroize SK & buffers
      Mem-->>App: Ack
    end
    App-->>App: Migration complete
```

#### UC2: On‑Demand File Access

```mermaid
sequenceDiagram
    title UC2: On‑Demand File Access
    participant User  as Client App
    participant RBAC  as RBAC Service
    participant Store as Storage Adapter
    participant Crypto as PQC Engine
    participant TPM   as TPM Module
    participant Mem   as Memory Manager
    participant Audit as Audit Service

    User->>RBAC: Decrypt /foo/bar.pdf? (userID)
    RBAC-->>User: Permit
    User->>Store: Fetch ciphertext + {c_kem, wrappedSS}
    Store-->>User: Data
    User->>Crypto: KEMDecapsulate(c_kem) → SK
    Crypto-->>User: SK
    loop chunk
      User->>Crypto: DecryptPQAEAD(c_ph, SK)
      Crypto-->>User: plaintext
      User->>Crypto: VerifyDilithium(sig, c_ph ∥ metadata)
      Crypto-->>User: OK/TamperAlert
      alt TamperAlert
        User->>Audit: Log tamper event
        Audit-->>User: Ack
        User-->>User: Abort & alert
      else
        User-->>User: Assemble plaintext
      end
    end
    User->>Mem: Zeroize SK & buffers
    Mem-->>User: Ack
    User->>Audit: Log decrypt + MerkleLeaf
    Audit-->>User: Ack
    User-->>User: Delivery complete
```

#### UC3: OCR‑Driven Classification & Storage

```mermaid
sequenceDiagram
    title UC3: OCR‑Driven Classification & Storage
    participant Scan  as Scanner Service
    participant OCR   as OCR Engine
    participant Class as Classifier
    participant RBAC  as RBAC Service
    participant TPM   as TPM Module
    participant Crypto as PQC Engine
    participant Store as Storage Adapter
    participant Audit as Audit Service
    participant Mem   as Memory Manager

    Scan->>OCR: New scan.png
    OCR-->>Scan: raw_text
    Scan->>Class: classify(raw_text)
    Class-->>Scan: level=Secret
    Scan->>RBAC: Encrypt as Secret? (userID)
    RBAC-->>Scan: Permit
    Scan->>Mem: Zeroize buffers
    Mem-->>Scan: Ack
    Scan->>TPM: Decapsulate → SK
    TPM-->>Scan: SK
    Scan->>Mem: Load image
    Mem-->>Scan: Buffer ready
    Scan->>Crypto: EncryptPQAEAD(image, SK, AD=level ∥ userID)
    Crypto-->>Scan: ciphertext
    Scan->>Crypto: SignDilithium(ciphertext ∥ metadata)
    Crypto-->>Scan: sig
    Scan->>Store: write {ciphertext, sig, level, userID}
    Store-->>Scan: Ack
    Scan->>Crypto: KEMEncapsulate(SK)
    Crypto-->>Scan: {c_kem, wrappedSS}
    Scan->>Store: write session metadata
    Scan->>Audit: Log OCR-store + MerkleLeaf
    Audit-->>Scan: Ack
    Scan->>Mem: Zeroize SK & buffers
    Mem-->>Scan: Ack
```

#### UC4: RBAC Policy Management

```mermaid
sequenceDiagram
    title UC4: RBAC Policy Management
    participant Admin as Admin Console
    participant API   as RBAC API
    participant DB    as Policy DB
    participant Crypto as PQC Engine
    participant Audit as Audit Service

    Admin->>API: POST /roles {role, actions} (adminID)
    API->>DB: Insert policy
    DB-->>API: OK
    API->>Crypto: SignDilithium(request ∥ adminID)
    Crypto-->>API: sig
    API->>Audit: Log policy-create + sig + MerkleLeaf
    Audit-->>API: Ack
    API-->>Admin: 201 Created {policyID, sig}

    Admin->>API: PATCH /roles/X {actions} (adminID)
    API->>DB: Update policy
    DB-->>API: OK
    API->>Crypto: SignDilithium(update ∥ adminID)
    Crypto-->>API: sig
    API->>Audit: Log policy-update + sig + MerkleLeaf
    Audit-->>API: Ack
    API-->>Admin: 200 OK
```

#### UC5: Audit & Compliance Reporting

```mermaid
sequenceDiagram
    title UC5: Audit & Compliance Reporting
    participant Auditor as UI
    participant API      as Audit API
    participant DB       as Audit Store
    participant Merkle   as Merkle Service
    participant Crypto   as PQC Engine

    Auditor->>API: GET /reports?from=&to= (auditorID)
    API->>DB: Fetch leaves
    DB-->>API: [leaf1..leafN]
    API->>Merkle: buildRoot([leaf1..leafN])
    Merkle-->>API: rootHash + proofs
    API->>Crypto: SignDilithium(rootHash ∥ auditorID)
    Crypto-->>API: reportSig
    API-->>Auditor: {rootHash, proofs, reportSig}
```

#### UC6: Key Rotation & Revocation

```mermaid
sequenceDiagram
    title UC6: Key Rotation & Revocation
    participant Job   as Scheduler
    participant TPM   as TPM Module
    participant Store as Key Metadata Store
    participant Crypto as PQC Engine
    participant Audit as Audit Service

    Job->>TPM: Gen new Kyber kp_new
    TPM-->>Job: pub_new
    Job->>Store: Revoke old, add pub_new
    Store-->>Job: OK
    Job->>Crypto: SignDilithium(pub_new ∥ jobID)
    Crypto-->>Job: sig
    Job->>Audit: Log key-rotate + sig + MerkleLeaf
    Audit-->>Job: Ack
    Job-->>Ops: Notify rotation complete + sig
```
