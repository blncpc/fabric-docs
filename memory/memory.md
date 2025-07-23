# ILOS (Immutable Loan Origination System) - Comprehensive Memory Store

## System Overview
- **Purpose**: Enterprise loan origination system built on Hyperledger Fabric
- **Architecture**: Multi-organizational blockchain network with clear separation of duties
- **Actors**: Personal Banker (PB), SPU Officer, COPS Officer, CIU Officer, RRU Officer, Disbursement Officer, Auditor

## ILOS Workflow Stages
1. **PB (Personal Banker)** - Application Submission
2. **SPU (Sector Processing Unit)** - Initial Review  
3. **COPS (Central Ops)** - Verification
4. **CIU (Central Investigation Unit)** - Compliance
5. **RRU (Review & Recommendation Unit)** - Final Approval
6. **Disbursement** - Fund Release

## Network Architecture

### Organizations
- **OrgPB**: Personal Banking Division
- **OrgSPU**: Sector Processing Unit
- **OrgCOPS**: Central Operations
- **OrgCIU**: Central Investigation Unit
- **OrgRRU**: Review & Recommendation Unit
- **OrgDisbursement**: Disbursement Division
- **OrgOrderer**: Ordering Service (separate from business orgs)

### Network Components
- **7 Organizations** (6 business + 1 orderer)
- **1-2 peers per organization** (for redundancy)
- **CAs** for each organization
- **Orderer nodes** using Raft consensus
- **Channels**: Single channel for workflow or multiple channels for privacy

## Data Architecture

### On-Chain Storage (Ledger)
```json
{
  "loanId": "LOAN-2024-001",
  "applicantId": "PB001-CUSTOMER-12345",
  "status": "REVIEWED_BY_SPU",
  "amount": 100000,
  "loanType": "BUSINESS",
  "submissionDate": "2024-01-15T10:30:00Z",
  "lastModified": "2024-01-16T14:20:00Z",
  "currentStage": "SPU_REVIEW",
  "offChainDataHash": "sha256:abc123...",
  "workflow": {
    "submitted": {
      "timestamp": "2024-01-15T10:30:00Z",
      "by": "PB001",
      "action": "SUBMITTED"
    },
    "spu_reviewed": {
      "timestamp": "2024-01-16T14:20:00Z", 
      "by": "SPU001",
      "action": "REVIEWED",
      "decision": "APPROVED"
    }
  },
  "endorsements": ["OrgPB", "OrgSPU"],
  "version": 2
}
```

### Off-Chain Storage (PostgreSQL)
- Complete customer data (PII, financial details)
- Full application forms
- Supporting documents
- Detailed review notes
- Internal communications

### API Data Structures
```go
// NTB/ETB Customer Data
type NTBCustomer struct {
    CustomerId      string `json:"customerId"`
    Cnic            string `json:"cnic"`
    Status          string `json:"status"`
    Fullname        string `json:"fullname"`
    // ... other fields
}

// CIF Data
type CustomerIdType struct {
    CustomerId   string `json:"customer_id"`
    Position     string `json:"position"`
    // ... other fields
}
```

## Technical Stack (BTS)

### Frontend
- **Authentication**: Client certificates or JWT tokens
- **Role-based UI**: Different interfaces per actor
- **Features**:
  - Application submission forms
  - Document upload
  - Status tracking
  - Review workflows
  - Approval dashboards
  - Audit trails

### Backend
- **Fabric SDK**: Go/Node.js/Java SDK for blockchain interaction
- **Database**: PostgreSQL for off-chain data
- **API Gateway**: RESTful APIs for frontend
- **Hash Generation**: SHA-256 for off-chain data integrity
- **Business Logic**: Validation, status transitions, notifications

### Blockchain Layer
- **Smart Contracts (Chaincode)**: Go/Node.js/Java
- **Consensus**: Raft ordering service
- **State Database**: LevelDB or CouchDB
- **Private Data**: For sensitive information sharing

## Complete User Story Example

### User Story 1: Loan Application Submission
**Actor**: Personal Banker (PB)
**Goal**: Submit a new loan application

#### Frontend Flow
1. PB logs into banking portal with client certificate
2. Selects "New Loan Application" 
3. Fills application form with customer data
4. Uploads required documents
5. Reviews and submits application

#### Backend/SDK Flow
```go
// 1. Backend validates input data
func SubmitApplication(ctx context.Context, loanData LoanApplication) error {
    // Generate hash of complete application data
    dataHash := generateSHA256Hash(loanData)
    
    // Create minimal on-chain record
    onChainData := OnChainLoanData{
        LoanId: loanData.LoanId,
        ApplicantId: loanData.ApplicantId,
        Amount: loanData.Amount,
        Status: "SUBMITTED",
        OffChainDataHash: dataHash,
    }
    
    // Submit to Fabric via SDK
    response, err := fabricSDK.InvokeChaincode(
        "ilos-chaincode",
        "SubmitApplication", 
        []byte(json.Marshal(onChainData))
    )
    
    return err
}
```

#### Smart Contract Flow
```go
func (s *ILOSContract) SubmitApplication(ctx contractapi.TransactionContextInterface, loanDataJSON string) error {
    // Parse input
    var loanData OnChainLoanData
    json.Unmarshal([]byte(loanDataJSON), &loanData)
    
    // Validate business rules
    if loanData.Amount <= 0 {
        return fmt.Errorf("loan amount must be positive")
    }
    
    // Check if loan ID already exists
    existing, err := ctx.GetStub().GetState(loanData.LoanId)
    if existing != nil {
        return fmt.Errorf("loan ID already exists")
    }
    
    // Set initial status and timestamp
    loanData.Status = "SUBMITTED"
    loanData.SubmissionDate = time.Now().Format(time.RFC3339)
    
    // Store to ledger
    loanJSON, _ := json.Marshal(loanData)
    return ctx.GetStub().PutState(loanData.LoanId, loanJSON)
}
```

#### Endorsement Policy
```yaml
Endorsement Policy: "OR('OrgPB.peer')"
# Only PB's peer needs to endorse submission
```

#### What Gets Stored
- **World State**: Minimal loan record with hash reference
- **Blockchain**: Transaction showing PB submitted application
- **PostgreSQL**: Complete application data, documents, customer info

## Fabric Features Utilized

### 1. **Chaincode Lifecycle**
- Package, install, approve, commit workflow
- Version management for upgrades
- Organization-level approval process

### 2. **Private Data Collections**
- Sensitive data shared only between specific orgs
- Hash stored on public channel, data in private collection

### 3. **Access Control**
- MSP-based identity management
- Client certificates for authentication
- Attribute-based access control (ABAC)

### 4. **Event Services**
- Chaincode events for real-time notifications
- Block events for audit trails
- Filtered block events for lightweight monitoring

### 5. **Service Discovery**
- Dynamic peer discovery
- Endorsement policy resolution
- Gateway pattern for simplified client development

### 6. **Gateway SDK**
- Simplified transaction submission
- Automatic endorsement and ordering
- Connection pooling and load balancing

## Security & Compliance

### Identity Management
- **CA Hierarchy**: Root CA → Intermediate CA → End-entity certificates
- **MSP Configuration**: Each org has its own MSP
- **Certificate Attributes**: Role-based access (admin, peer, client)

### Privacy Protection
- **PII off-chain**: Personal data in PostgreSQL, not on ledger
- **Hash verification**: Integrity without exposure
- **Private data**: Sensitive workflow data in private collections

### Audit Trail
- **Immutable history**: All state changes recorded on blockchain
- **Transaction metadata**: Who, what, when for every change
- **Rich queries**: CouchDB for complex audit queries

## Data Verification Strategy

### Smart Contract Verification
1. **Input validation**: Required fields, data types, ranges
2. **Business rule enforcement**: Status transitions, approvals
3. **Hash verification**: Compare provided hash with stored hash
4. **Access control**: Verify caller identity and permissions

### Backend Verification  
1. **Data integrity**: Generate and verify hashes
2. **Business logic**: Complex validations before blockchain submission
3. **Integration**: Verify data against external systems (CIF, NTB/ETB)

## Implementation Best Practices

### On-Chain vs Off-Chain Decision Matrix
- **On-Chain**: Workflow status, approvals, timestamps, hashes, audit trail
- **Off-Chain**: PII, documents, detailed forms, internal notes, communications

### Performance Optimization
- **Batch transactions**: Multiple operations in single transaction
- **Efficient queries**: Use composite keys for range queries
- **State pruning**: Archive old loan data to maintain performance

### Error Handling
- **Transaction validation**: Comprehensive validation before submission
- **Retry mechanisms**: Handle temporary network failures
- **Rollback procedures**: Compensating transactions for complex workflows

This is a critical and nuanced question—especially for regulated industries like banking. Let’s rigorously analyze whether you should store all this data on the Hyperledger Fabric ledger, referencing best practices, security, privacy, and operational realities.

---

## 1. **What Does the Banking Industry (and Fabric Best Practice) Recommend?**

### **A. Principle: Store Only What Needs to be Shared, Audited, or Proven**

- **Fabric’s ledger is for data that must be:**
  - **Immutable** (cannot be changed or deleted)
  - **Shared** (across multiple, potentially competing organizations)
  - **Auditable** (regulators, auditors, or other orgs may need to verify)
  - **Proven** (you need to prove to a third party that a fact existed at a certain time)

- **Do NOT store:**
  - Large volumes of PII (personally identifiable information) unless absolutely necessary
  - Data that is not required for cross-org workflow, audit, or compliance
  - Data that is only relevant to your own org’s internal processes

**References:**
- [Hyperledger Fabric Private Data Best Practices](https://hyperledger-fabric.readthedocs.io/en/latest/private-data/private-data.html)
- [IBM: Best Practices for Storing Data on Blockchain](https://community.ibm.com/community/user/integration/blogs/mark-parzygnat1/2020/01/14/best-practices-for-storing-data-on-blockchain)
- [Enterprise Blockchain Patterns](https://www.hyperledger.org/learn/publications/enterprise-blockchain-patterns)

---

### **B. What Do Banks Actually Do?**

- **On-chain:** Only store the minimum set of data required for regulatory, audit, and workflow purposes (e.g., loan status, key approvals, hashes of documents, references to off-chain data).
- **Off-chain:** Store all detailed customer data, KYC, and supporting documents in secure, access-controlled databases (like PostgreSQL), with references/hashes on-chain for integrity and audit.

**Why?**
- **Regulatory compliance:** Storing PII on a shared ledger can violate data protection laws (GDPR, local banking regulations).
- **Performance:** Ledgers are not optimized for large, complex, or frequently changing data.
- **Privacy:** Even with private data collections, there’s risk of data leakage or misconfiguration.

---

## 2. **Your Data Structures: What Should Go On-Chain?**

### **A. What to Store On-Chain (Ledger)**
- **Loan ID** (unique key)
- **Minimal customer reference** (e.g., customer ID, not full details)
- **Loan status and workflow state** (e.g., SUBMITTED, APPROVED, DISBURSED)
- **Timestamps and actor IDs for each workflow step**
- **Hash of the full application data** (for integrity/audit)
- **References to off-chain data** (e.g., document hashes, URLs, or IDs)
- **Any data required for cross-org workflow or regulatory audit**

### **B. What to Store Off-Chain (PostgreSQL)**
- All detailed customer information (full KYC, addresses, phone, etc.)
- All supporting documents (PDFs, images, etc.)
- All data not required to be shared or audited by other orgs

### **C. If You Need to Prove Data Integrity**
- Store a **cryptographic hash** of the full application/customer data on-chain.
- This allows you to prove, later, that the off-chain data has not been tampered with.

---

## 3. **Example: On-Chain Data Structure**

```json
{
  "loanId": "LN12345",
  "customerId": "CUST001",
  "status": "APPROVED_BY_RRU",
  "workflow": [
    { "step": "SUBMITTED", "actor": "pbUser01", "timestamp": "..." },
    { "step": "REVIEWED_BY_SPU", "actor": "spuUser01", "timestamp": "..." }
    // ...
  ],
  "applicationHash": "0xabc123...", // hash of full application data
  "documentHashes": [
    { "type": "ID Proof", "hash": "0xdef456..." }
  ],
  "offChainRef": "postgres://loans/LN12345"
}
```

---

## 4. **Why Not Store Everything On-Chain?**

- **Privacy:** Even with private data collections, there’s risk of misconfiguration or future access by unintended parties.
- **Regulatory Risk:** You may be forced to delete or modify PII (right to be forgotten), which is impossible on an immutable ledger.
- **Performance:** Large, complex records slow down endorsement, validation, and block propagation.
- **Cost:** More data on-chain means more storage, backup, and operational overhead for every org in the network.

---

## 5. **Counter-Arguments (If You Disagree)**

- **“But I want full transparency!”**  
  You can achieve this with hashes and references, without exposing sensitive data.
- **“But I want to use private data collections!”**  
  Even then, only store what is absolutely necessary, and be aware of the operational complexity and risk.
- **“But I want to use blockchain as my main database!”**  
  This is not the intended use of Fabric or any enterprise blockchain. Use it as a trust and audit layer, not a full data store.

---

## 6. **Best Practice Recommendation**

**Store on the ledger:**
- Loan ID, status, workflow steps, actor IDs, timestamps
- Hashes of full application/customer data and documents
- References to off-chain data

**Store in PostgreSQL:**
- All detailed customer and loan data
- All supporting documents

**If you need to prove data integrity:**  
- Always store a hash of the full off-chain record on-chain.

---

## 7. **References in Fabric Docs**

- [Private Data: When to use a collection within a channel vs. a separate channel](private-data/private-data.md)
- [Key Concepts: Ledger](key_concepts.rst)
- [Chaincode Data Structures](smartcontract/smartcontract.md)
- [Security Model](security_model.md)

---

## 8. **Summary Table**

| Data Field         | On-Chain (Ledger) | Off-Chain (DB) | Private Data Collection? | Hash on Ledger? |
|--------------------|-------------------|----------------|-------------------------|-----------------|
| Loan ID            | Yes               | Yes            | -                       | -               |
| Customer ID        | Yes               | Yes            | Maybe                   | -               |
| Full Customer Data | No                | Yes            | Maybe                   | Yes             |
| Loan Details       | Minimal           | Yes            | Maybe                   | Yes             |
| Status/Workflow    | Yes               | Yes            | -                       | -               |
| Document Hashes    | Yes               | Yes            | -                       | -               |
| Document Files     | No                | Yes            | Maybe                   | Yes             |

---

## 9. **Conclusion**

**Do NOT store all your detailed customer and loan data on the ledger.**  
Store only what is required for audit, workflow, and cross-org trust, plus hashes/references for integrity.  
This is the best practice in banking and enterprise blockchain, and is strongly recommended by the Hyperledger Fabric community and major industry players.

If you want a sample chaincode struct or a reference implementation, let me know!



Absolutely! Here’s a comprehensive, step-by-step scenario for a full ILOS (Immutable Loan Origination System) application, including user stories, technical backend tech stack (BTS), frontend features, Fabric SDK interactions, smart contract logic, endorsement policies, and what is stored in the ledger at each step.

---

# **ILOS Full Process: End-to-End Scenario**

## **Actors**
- **Personal Banker (PB)**: Submits and manages applications.
- **SPU Officer**: Reviews applications.
- **COPS Officer**: Performs central ops verification.
- **CIU Officer**: Conducts compliance checks.
- **RRU Officer**: Gives final approval.
- **Disbursement Officer**: Releases funds.
- **Auditor**: Views application history.

---

## **User Story 1: Loan Application Submission (PB)**

### **Frontend Features**
- Secure login (PB identity)
- Loan application form (fields: amount, term, applicant info, document upload)
- “Submit Application” button
- Dashboard: View submitted applications and their statuses

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications`
- **Validates** input, stores metadata in PostgreSQL
- **Uses Fabric SDK** to invoke `SubmitApplication` on chaincode

### **Fabric SDK Call**
- **Function:** `SubmitApplication`
- **Args:** Minimal loan data (loanId, applicantId, amount, term, hash of full application data)
- **Identity:** PB’s client certificate

### **Smart Contract Logic**
- Verifies PB’s role (`OrgPB.client`)
- Validates input (loanId uniqueness, required fields)
- Sets status to `SUBMITTED`
- Stores minimal data and hash in ledger

### **Endorsement Policy**
- `"OR('OrgPB.peer')"` (PB’s org peer must endorse)

### **Ledger Storage**
```json
{
  "loanId": "LN12345",
  "applicantId": "PB001",
  "amount": 10000,
  "term": 60,
  "status": "SUBMITTED",
  "applicationHash": "0xabc123...",
  "history": [
    { "status": "SUBMITTED", "actor": "pbUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 2: Initial Review (SPU Officer)**

### **Frontend Features**
- Secure login (SPU identity)
- Dashboard: List of applications with status `SUBMITTED`
- “Review” button, notes field

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications/:loanId/review`
- **Uses Fabric SDK** to invoke `ReviewBySPU`

### **Fabric SDK Call**
- **Function:** `ReviewBySPU`
- **Args:** loanId, review notes
- **Identity:** SPU officer’s client certificate

### **Smart Contract Logic**
- Verifies SPU’s role (`OrgSPU.client`)
- Checks current status is `SUBMITTED`
- Updates status to `REVIEWED_BY_SPU`, records notes

### **Endorsement Policy**
- `"OR('OrgSPU.peer')"`

### **Ledger Storage**
```json
{
  ...,
  "status": "REVIEWED_BY_SPU",
  "review": {
    "reviewedBy": "spuUser01",
    "reviewDate": "...",
    "notes": "All docs present"
  },
  "history": [
    ...,
    { "status": "REVIEWED_BY_SPU", "actor": "spuUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 3: Central Ops Verification (COPS Officer)**

### **Frontend Features**
- Secure login (COPS identity)
- Dashboard: List of applications with status `REVIEWED_BY_SPU`
- “Approve” button, notes field

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications/:loanId/cops-approve`
- **Uses Fabric SDK** to invoke `ApproveByCOPS`

### **Fabric SDK Call**
- **Function:** `ApproveByCOPS`
- **Args:** loanId, notes
- **Identity:** COPS officer’s client certificate

### **Smart Contract Logic**
- Verifies COPS role
- Checks status is `REVIEWED_BY_SPU`
- Updates status to `APPROVED_BY_COPS`, records notes

### **Endorsement Policy**
- `"OR('OrgCOPS.peer')"`

### **Ledger Storage**
```json
{
  ...,
  "status": "APPROVED_BY_COPS",
  "cops": {
    "approvedBy": "copsUser01",
    "approvalDate": "...",
    "notes": "KYC/AML passed"
  },
  "history": [
    ...,
    { "status": "APPROVED_BY_COPS", "actor": "copsUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 4: Compliance Check (CIU Officer)**

### **Frontend Features**
- Secure login (CIU identity)
- Dashboard: List of applications with status `APPROVED_BY_COPS`
- “Clear” button, notes field

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications/:loanId/ciu-clear`
- **Uses Fabric SDK** to invoke `ClearByCIU`

### **Fabric SDK Call**
- **Function:** `ClearByCIU`
- **Args:** loanId, notes
- **Identity:** CIU officer’s client certificate

### **Smart Contract Logic**
- Verifies CIU role
- Checks status is `APPROVED_BY_COPS`
- Updates status to `CLEARED_BY_CIU`, records notes

### **Endorsement Policy**
- `"OR('OrgCIU.peer')"`

### **Ledger Storage**
```json
{
  ...,
  "status": "CLEARED_BY_CIU",
  "ciu": {
    "clearedBy": "ciuUser01",
    "clearanceDate": "...",
    "notes": "No compliance issues"
  },
  "history": [
    ...,
    { "status": "CLEARED_BY_CIU", "actor": "ciuUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 5: Final Approval (RRU Officer)**

### **Frontend Features**
- Secure login (RRU identity)
- Dashboard: List of applications with status `CLEARED_BY_CIU`
- “Approve” button, notes field

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications/:loanId/rru-approve`
- **Uses Fabric SDK** to invoke `ApproveByRRU`

### **Fabric SDK Call**
- **Function:** `ApproveByRRU`
- **Args:** loanId, notes
- **Identity:** RRU officer’s client certificate

### **Smart Contract Logic**
- Verifies RRU role
- Checks status is `CLEARED_BY_CIU`
- Updates status to `APPROVED_BY_RRU`, records notes

### **Endorsement Policy**
- `"OR('OrgRRU.peer')"`

### **Ledger Storage**
```json
{
  ...,
  "status": "APPROVED_BY_RRU",
  "rru": {
    "approvedBy": "rruUser01",
    "approvalDate": "...",
    "notes": "Final approval granted"
  },
  "history": [
    ...,
    { "status": "APPROVED_BY_RRU", "actor": "rruUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 6: Disbursement (Disbursement Officer)**

### **Frontend Features**
- Secure login (Disbursement identity)
- Dashboard: List of applications with status `APPROVED_BY_RRU`
- “Disburse” button, amount field, notes field

### **Backend (BTS)**
- **API Endpoint:** `POST /api/loan-applications/:loanId/disburse`
- **Uses Fabric SDK** to invoke `DisburseFunds`

### **Fabric SDK Call**
- **Function:** `DisburseFunds`
- **Args:** loanId, amount, notes
- **Identity:** Disbursement officer’s client certificate

### **Smart Contract Logic**
- Verifies Disbursement role
- Checks status is `APPROVED_BY_RRU`
- Updates status to `DISBURSED`, records disbursement details

### **Endorsement Policy**
- `"OR('OrgDISB.peer')"`

### **Ledger Storage**
```json
{
  ...,
  "status": "DISBURSED",
  "disbursement": {
    "disbursedBy": "disbUser01",
    "disbursementDate": "...",
    "amountDisbursed": 10000,
    "notes": "Funds released"
  },
  "history": [
    ...,
    { "status": "DISBURSED", "actor": "disbUser01", "timestamp": "..." }
  ]
}
```

---

## **User Story 7: Audit (Auditor)**

### **Frontend Features**
- Secure login (Auditor identity)
- Search applications by loanId, applicantId, status
- View full application history (all status transitions, actors, timestamps)

### **Backend (BTS)**
- **API Endpoint:** `GET /api/loan-applications/:loanId/history`
- **Uses Fabric SDK** to invoke `GetHistoryForKey`

### **Fabric SDK Call**
- **Function:** `GetHistoryForKey`
- **Args:** loanId
- **Identity:** Auditor’s client certificate

### **Smart Contract Logic**
- Returns full immutable history of the application

### **Ledger Storage**
- All state transitions and actors are available for audit

---

# **Summary Table: End-to-End Process**

| Step         | Frontend Feature         | API Endpoint         | SDK Function         | Contract Logic         | Endorsement Policy      | Ledger Storage         |
|--------------|-------------------------|----------------------|----------------------|------------------------|-------------------------|-----------------------|
| Submission   | Form, dashboard         | POST /loan-apps      | SubmitApplication    | PB role, validate      | OrgPB.peer              | Minimal + hash        |
| Review (SPU) | Review, notes           | POST /loan-apps/:id/review | ReviewBySPU     | SPU role, status check | OrgSPU.peer             | Review details        |
| COPS         | Approve, notes          | POST /loan-apps/:id/cops-approve | ApproveByCOPS | COPS role, status check| OrgCOPS.peer            | COPS details          |
| CIU          | Clear, notes            | POST /loan-apps/:id/ciu-clear | ClearByCIU     | CIU role, status check | OrgCIU.peer             | CIU details           |
| RRU          | Approve, notes          | POST /loan-apps/:id/rru-approve | ApproveByRRU  | RRU role, status check | OrgRRU.peer             | RRU details           |
| Disbursement | Disburse, amount, notes | POST /loan-apps/:id/disburse | DisburseFunds | DISB role, status check| OrgDISB.peer            | Disbursement details  |
| Audit        | Search, view history    | GET /loan-apps/:id/history | GetHistoryForKey | N/A (read-only)        | N/A                     | Full history          |

---

# **Fabric Features to Use**

- **Client Identity Library**: Enforce role-based access in chaincode ([docs](chaincode4ade.rst))
- **Endorsement Policies**: Per-stage, per-org ([docs](endorsement-policies.rst))
- **Private Data Collections**: For sensitive fields ([docs](private-data/private-data.md))
- **Chaincode Events**: For real-time UI updates ([docs](peer_event_services.rst))
- **GetHistoryForKey**: For audit ([docs](https://hyperledger.github.io/fabric-chaincode-go/chaincode.html#ChaincodeStubInterface.GetHistoryForKey))
- **Rich Queries**: If using CouchDB ([docs](couchdb_as_state_database.rst))

---

# **Frontend: Fabric Perspective**

- **Role-based dashboards**: Each user sees only what’s relevant to their role.
- **Status tracking**: Real-time updates via chaincode events.
- **Audit trail**: Full history view for auditors.
- **Minimal data entry**: Only required fields, with off-chain storage for large/sensitive data.
- **Notifications**: On status change, approval, or disbursement.

---

# **What’s Stored in the Ledger at Each Step?**

- **Minimal, workflow-critical data**: loanId, status, actor, timestamp, hash of full application.
- **No full PII or large documents**: Only references/hashes.
- **Full audit trail**: All status transitions, actors, and timestamps.

---

**If you want a sample chaincode function, SDK call, or UI wireframe for any step, let me know!**  
This scenario ensures a robust, auditable, and privacy-preserving ILOS process, leveraging the best of Hyperledger Fabric and enterprise application design.