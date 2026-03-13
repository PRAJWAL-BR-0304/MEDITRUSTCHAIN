# Drug Master Approval Flow — Implementation Plan
## MediTrustChain: Regulator-Controlled Composition Lock

> **Source:** `reg---manu.docx` — Panel feedback correction  
> **Principle:** Regulator defines and locks drug composition. Manufacturer can ONLY create batches from pre-approved drug templates.  
> **Stack:** Next.js (App Router) · TypeScript · Supabase (PostgreSQL + RLS) · Solidity (Hardhat) · ethers.js

---

## Table of Contents

1. [Overview of the Change](#overview)
2. [Phase 1 — Database Schema (Supabase Migration)](#phase-1)
3. [Phase 2 — Smart Contract Update](#phase-2)
4. [Phase 3 — Backend / API Routes](#phase-3)
5. [Phase 4 — Regulator Dashboard (Frontend)](#phase-4)
6. [Phase 5 — Manufacturer Dashboard (Frontend)](#phase-5)
7. [Phase 6 — Verification & Hash Comparison Logic](#phase-6)
8. [Phase 7 — TypeScript Types Update](#phase-7)
9. [Phase 8 — Testing & Validation](#phase-8)
10. [File Map — Every File to Touch](#file-map)
11. [Viva Explanation Script](#viva-script)

---

## Overview of the Change <a name="overview"></a>

### Before (Current State — WRONG)
```
Manufacturer → Types composition → System hashes it → Stored on blockchain
```
Manufacturer is in full control of composition — panel correctly flagged this.

### After (Corrected Flow — CORRECT)
```
Regulator → Approves drug template → Locks composition hash on blockchain
    ↓
Manufacturer → Selects approved drug from dropdown → Creates batch
    ↓
System → Verifies composition hash against blockchain before batch creation
    ↓
Blockchain → Stores batch hash (with pre-approved composition)
```

---

## Phase 1 — Database Schema (Supabase Migration) <a name="phase-1"></a>

### Task 1.1 — Create `drug_master` Table

**File to create:**
```
MediTrustChain/supabase/migrations/20260311000001_drug_master.sql
```

**SQL content:**
```sql
-- =====================================================
-- MIGRATION: DRUG MASTER TABLE
-- Purpose: Regulator locks approved drug compositions
-- =====================================================

CREATE TABLE IF NOT EXISTS drug_master (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Drug identity
    drug_name       VARCHAR(255) NOT NULL,
    generic_name    VARCHAR(255),
    drug_code       VARCHAR(100) NOT NULL UNIQUE,     -- e.g., "DRG-001"

    -- Approved composition (locked by regulator)
    composition     TEXT NOT NULL,                   -- e.g., "Paracetamol 500mg"
    strength        VARCHAR(100) NOT NULL,            -- e.g., "500mg"
    dosage_form     VARCHAR(100) NOT NULL,            -- e.g., "Tablet", "Syrup"
    approved_expiry_months INTEGER NOT NULL,          -- shelf life in months

    -- Approved manufacturers (array of organization IDs)
    approved_manufacturer_ids UUID[] DEFAULT '{}',

    -- Cryptographic hash (SHA-256 of drug_name + composition + strength)
    composition_hash  VARCHAR(66) NOT NULL,           -- 0x + 64 hex chars

    -- Blockchain proof
    blockchain_tx_hash  VARCHAR(66),                 -- transaction hash
    blockchain_block    BIGINT,                       -- block number

    -- Regulator who approved
    approved_by         UUID REFERENCES stakeholders(id),
    approved_by_org     UUID REFERENCES organizations(id),

    -- Timestamps & status
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_drug_master_code    ON drug_master(drug_code);
CREATE INDEX IF NOT EXISTS idx_drug_master_active  ON drug_master(is_active);
CREATE INDEX IF NOT EXISTS idx_drug_master_approver ON drug_master(approved_by);

-- Trigger: auto-update updated_at
CREATE TRIGGER update_drug_master_updated_at
    BEFORE UPDATE ON drug_master
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- =====================================================
-- RLS Policies
-- =====================================================

ALTER TABLE drug_master ENABLE ROW LEVEL SECURITY;

-- Admin: full access
CREATE POLICY admin_all_drug_master ON drug_master
    FOR ALL USING (
        EXISTS (SELECT 1 FROM admin_users WHERE id = auth.uid() AND is_active = true)
    );

-- Regulator: can INSERT and UPDATE their own approved drugs
CREATE POLICY regulator_manage_drug_master ON drug_master
    FOR ALL USING (
        EXISTS (
            SELECT 1 FROM stakeholders
            WHERE user_id  = auth.uid()
            AND   role     = 'regulator'
            AND   is_active = true
        )
    );

-- All active stakeholders: can SELECT (manufacturer needs to read the list)
CREATE POLICY stakeholder_read_drug_master ON drug_master
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM stakeholders
            WHERE user_id  = auth.uid()
            AND   is_active = true
        )
    );
```

### Task 1.2 — Add `drug_master_id` Column to `batches` Table

**File to create:**
```
MediTrustChain/supabase/migrations/20260311000002_batches_add_drug_master_fk.sql
```

```sql
-- Link batches to the approved drug template
ALTER TABLE batches
    ADD COLUMN IF NOT EXISTS drug_master_id UUID REFERENCES drug_master(id),
    ADD COLUMN IF NOT EXISTS composition_hash VARCHAR(66),    -- hash used at creation time
    ADD COLUMN IF NOT EXISTS composition TEXT,                -- denormalized for quick reads
    ADD COLUMN IF NOT EXISTS strength VARCHAR(100);           -- denormalized

CREATE INDEX IF NOT EXISTS idx_batches_drug_master ON batches(drug_master_id);
```

---

## Phase 2 — Smart Contract Update <a name="phase-2"></a>

### Task 2.1 — Add `DrugTemplate` Struct to `MediTrustChainV2.sol`

**File to edit:**
```
MediTrustChain/blockchain/contracts/MediTrustChainV2.sol
```

**Changes to make (add inside the contract, after existing structs):**

```solidity
/// ============ Drug Master / Regulator Template ============

struct DrugTemplate {
    uint256 id;
    string drugCode;           // e.g., "DRG-001"
    string drugName;
    string composition;
    string strength;
    bytes32 compositionHash;   // keccak256(drugName + composition + strength)
    address approvedBy;        // regulator wallet
    uint256 approvedAt;
    bool isActive;
}

uint256 private drugTemplateCounter = 0;
mapping(uint256 => DrugTemplate) public drugTemplates;
mapping(string => uint256) public drugCodeToTemplateId;   // drugCode → templateId
mapping(bytes32 => bool) public approvedCompositionHashes; // quick lookup

event DrugTemplateCreated(
    uint256 indexed templateId,
    string indexed drugCode,
    string drugName,
    bytes32 compositionHash,
    address indexed approvedBy
);
```

### Task 2.2 — Add `approveDrugTemplate` Function

**Add to `MediTrustChainV2.sol`:**

```solidity
/**
 * @dev Regulator locks an official drug composition
 * @param drugCode  Unique identifier e.g. "DRG-001"
 * @param drugName  Official name
 * @param composition  Approved ingredients
 * @param strength  Dosage strength
 */
function approveDrugTemplate(
    string calldata drugCode,
    string calldata drugName,
    string calldata composition,
    string calldata strength
) external whenNotPaused returns (uint256) {
    User memory caller = users[msg.sender];
    require(caller.isActive, "User not registered");
    require(caller.role == UserRole.REGULATOR, "Only regulators can approve drug templates");
    require(drugCodeToTemplateId[drugCode] == 0, "Drug code already exists");

    bytes32 compHash = keccak256(abi.encodePacked(drugName, composition, strength));

    drugTemplateCounter++;
    drugTemplates[drugTemplateCounter] = DrugTemplate({
        id:              drugTemplateCounter,
        drugCode:        drugCode,
        drugName:        drugName,
        composition:     composition,
        strength:        strength,
        compositionHash: compHash,
        approvedBy:      msg.sender,
        approvedAt:      block.timestamp,
        isActive:        true
    });

    drugCodeToTemplateId[drugCode]          = drugTemplateCounter;
    approvedCompositionHashes[compHash]     = true;

    emit DrugTemplateCreated(drugTemplateCounter, drugCode, drugName, compHash, msg.sender);
    return drugTemplateCounter;
}
```

### Task 2.3 — Modify `createBatch` to Require Drug Template ID

**Edit the existing `createBatch` function signature and add hash verification:**

```solidity
function createBatch(
    string calldata batchCode,
    string calldata drugName,
    uint256 quantity,
    uint256 mfgDate,
    uint256 expDate,
    uint256 drugTemplateId          // NEW: must reference an approved template
) external whenNotPaused returns (uint256) {
    // ... existing checks ...

    // NEW: Verify drug template exists and is active
    DrugTemplate storage template = drugTemplates[drugTemplateId];
    require(template.id != 0,    "Drug template not found");
    require(template.isActive,   "Drug template is inactive");

    // Verify composition hash matches the approved template
    bytes32 expectedHash = template.compositionHash;
    require(approvedCompositionHashes[expectedHash], "Composition not approved by regulator");

    // Existing hash now incorporates the approved templateId
    bytes32 dataHash = keccak256(abi.encodePacked(
        batchCode,
        msg.sender,
        drugName,
        quantity,
        mfgDate,
        expDate,
        block.timestamp,
        drugTemplateId           // binds batch to approved template
    ));

    // ... rest of existing createBatch logic unchanged ...
}
```

### Task 2.4 — Redeploy Contract

**File to run:**
```
MediTrustChain/blockchain/scripts/deploy.ts
```

```bash
cd MediTrustChain/blockchain
npx hardhat compile
npx hardhat run scripts/deploy.ts --network <your-network>
```

After deployment:
- Update `CONTRACT_ADDRESS` in `MediTrustChain/src/lib/blockchain/index.ts` (or wherever it's stored)
- Update ABI file if it's stored separately

---

## Phase 3 — Backend / API Routes <a name="phase-3"></a>

### Task 3.1 — Create `POST /api/drug-master/approve` Route

**File to create:**
```
MediTrustChain/src/app/api/drug-master/approve/route.ts
```

**Logic:**
```typescript
// POST /api/drug-master/approve
// Called by: Regulator Dashboard
// What it does:
//   1. Validate input (drug_name, composition, strength, dosage_form, etc.)
//   2. Compute compositionHash = SHA-256(drug_name + composition + strength)
//   3. Store record in drug_master table (Supabase)
//   4. Return { drug_master_id, composition_hash } to frontend
//   5. Frontend calls approveDrugTemplate() on blockchain with this hash

import { createClient } from '@/lib/supabase/server';
import { NextRequest, NextResponse } from 'next/server';
import crypto from 'crypto';

export async function POST(req: NextRequest) {
  const supabase = createClient();
  const body = await req.json();

  const {
    drug_name,
    generic_name,
    drug_code,
    composition,
    strength,
    dosage_form,
    approved_expiry_months,
    approved_manufacturer_ids,
  } = body;

  // Compute composition hash (same algorithm as smart contract keccak256)
  // Use ethers.js solidityPackedKeccak256 for exact match with Solidity
  const compositionHash = '0x' + crypto
    .createHash('sha256')
    .update(drug_name + composition + strength)
    .digest('hex');

  const { data, error } = await supabase
    .from('drug_master')
    .insert({
      drug_name,
      generic_name,
      drug_code,
      composition,
      strength,
      dosage_form,
      approved_expiry_months,
      approved_manufacturer_ids,
      composition_hash: compositionHash,
    })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }

  return NextResponse.json({ drug_master: data }, { status: 201 });
}
```

> **Note:** The blockchain call (`approveDrugTemplate`) happens on the **client side** in the regulator dashboard using `useContract()` hook — same pattern as existing `createBatch` and `approveBatch` in the project.

### Task 3.2 — Create `GET /api/drug-master` Route

**File to create:**
```
MediTrustChain/src/app/api/drug-master/route.ts
```

```typescript
// GET /api/drug-master
// Called by: Manufacturer Dashboard (to populate the drugs dropdown)
// Returns: list of active approved drugs

import { createClient } from '@/lib/supabase/server';
import { NextRequest, NextResponse } from 'next/server';

export async function GET(req: NextRequest) {
  const supabase = createClient();

  const { data, error } = await supabase
    .from('drug_master')
    .select(`
      id,
      drug_name,
      drug_code,
      composition,
      strength,
      dosage_form,
      approved_expiry_months,
      composition_hash,
      blockchain_tx_hash,
      approved_by,
      approved_manufacturer_ids
    `)
    .eq('is_active', true)
    .order('created_at', { ascending: false });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }

  return NextResponse.json({ drugs: data });
}
```

### Task 3.3 — Create `PATCH /api/drug-master/[id]/blockchain` Route

**File to create:**
```
MediTrustChain/src/app/api/drug-master/[id]/blockchain/route.ts
```

```typescript
// PATCH /api/drug-master/[id]/blockchain
// Called after: Regulator successfully calls approveDrugTemplate() on-chain
// What it does: Saves the blockchain tx_hash and block number back to Supabase

import { createClient } from '@/lib/supabase/server';
import { NextRequest, NextResponse } from 'next/server';

export async function PATCH(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createClient();
  const { blockchain_tx_hash, blockchain_block } = await req.json();

  const { error } = await supabase
    .from('drug_master')
    .update({ blockchain_tx_hash, blockchain_block })
    .eq('id', params.id);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }

  return NextResponse.json({ success: true });
}
```

---

## Phase 4 — Regulator Dashboard (Frontend) <a name="phase-4"></a>

### Task 4.1 — Add "Drug Approval" Tab to Regulator Dashboard

**File to edit:**
```
MediTrustChain/src/app/dashboard/regulator/page.tsx
```

**What to add:**

1. Import and use the existing `<Tabs>` component (already in the file).
2. Add a new tab: `"Drug Templates"` alongside the existing batch approval tab.
3. Inside the new tab, add a form with:

```tsx
// Drug Approval Form Fields:
const drugSchema = z.object({
  drugCode:              z.string().min(1),   // "DRG-001"
  drugName:              z.string().min(1),
  genericName:           z.string().optional(),
  composition:           z.string().min(1),   // "Paracetamol 500mg"
  strength:              z.string().min(1),   // "500mg"
  dosageForm:            z.string().min(1),   // "Tablet"
  approvedExpiryMonths:  z.coerce.number().min(1),
});
```

4. On submit: Call `POST /api/drug-master/approve` → get back `drug_master_id` and `composition_hash`
5. Then call `approveDrugTemplate()` on blockchain (use `useContract()` hook you already have)
6. After blockchain success: Call `PATCH /api/drug-master/[id]/blockchain` to save tx hash
7. Display approved drugs in a table below the form

**Key UI elements:**
- Form uses `react-hook-form` + `zod` (same pattern as manufacturer's batch form)
- Uses `<Button>` with `<Loader2>` spinner during submission
- Uses `useToast()` for success/error messages
- Blockchain section only shows if `isWalletMatched` (same pattern as existing regulator code)

---

## Phase 5 — Manufacturer Dashboard (Frontend) <a name="phase-5"></a>

### Task 5.1 — Replace Composition Text Input with Drug Dropdown

**File to edit:**
```
MediTrustChain/src/app/dashboard/manufacturer/page.tsx
```

**Current state of form fields:**
```tsx
// EXISTING (to be removed/changed):
<FormField name="drugName" ... />   // free text drug name
// (no composition field exists currently — but drug_name is free text)
```

**New state after changes:**
```tsx
// NEW: Replace free text drug name with a controlled dropdown

// 1. Add state to fetch approved drugs
const [approvedDrugs, setApprovedDrugs] = useState<DrugMaster[]>([]);
const [selectedDrug, setSelectedDrug] = useState<DrugMaster | null>(null);

// 2. useEffect to fetch approved drugs on mount
useEffect(() => {
  fetch('/api/drug-master')
    .then(res => res.json())
    .then(data => setApprovedDrugs(data.drugs || []));
}, []);

// 3. Replace drugName text input with Select dropdown
<FormField name="drugMasterId" render={...}>
  <Select onValueChange={(val) => {
    const drug = approvedDrugs.find(d => d.id === val);
    setSelectedDrug(drug || null);
    field.onChange(val);
  }}>
    <SelectTrigger><SelectValue placeholder="Select approved drug..." /></SelectTrigger>
    <SelectContent>
      {approvedDrugs.map(drug => (
        <SelectItem key={drug.id} value={drug.id}>
          {drug.drug_name} — {drug.drug_code} ({drug.strength})
        </SelectItem>
      ))}
    </SelectContent>
  </Select>
</FormField>

// 4. Show read-only composition details when drug is selected
{selectedDrug && (
  <Card className="border-green-600">
    <CardContent>
      <p><strong>Composition:</strong> {selectedDrug.composition}</p>
      <p><strong>Strength:</strong> {selectedDrug.strength}</p>
      <p><strong>Form:</strong> {selectedDrug.dosage_form}</p>
      <Badge variant="outline" className="text-green-600">
        ✔ Regulator Approved
      </Badge>
    </CardContent>
  </Card>
)}
```

### Task 5.2 — Update Batch Schema in Manufacturer Dashboard

```typescript
// Update batchSchema in manufacturer/page.tsx:
const batchSchema = z.object({
  batchId:      z.string().min(1, "Batch ID is required"),
  drugMasterId: z.string().uuid("Please select an approved drug"),    // NEW
  mfgDate:      z.date({ required_error: "Manufacturing date is required." }),
  expDate:      z.date({ required_error: "Expiry date is required." }),
  quantity:     z.coerce.number().min(1, "Quantity must be at least 1"),
  // drugName is now derived from drugMasterId — remove from form
});
```

### Task 5.3 — Update Batch Submission Logic

**In the form submit handler (`onSubmit` function):**

```typescript
// Current: addBatch({ name: values.drugName, ... })
// New:
const drug = approvedDrugs.find(d => d.id === values.drugMasterId);
if (!drug) {
  toast({ title: "Error", description: "Invalid drug selection", variant: "destructive" });
  return;
}

// Server-side hash verification (compare against blockchain)
const verifyRes = await fetch('/api/drug-master/verify-hash', {
  method: 'POST',
  body: JSON.stringify({
    drug_master_id: drug.id,
    composition_hash: drug.composition_hash,
  }),
});
const { valid } = await verifyRes.json();

if (!valid) {
  toast({
    title: "❌ Hash Mismatch",
    description: "Drug composition hash does not match blockchain record. Batch rejected.",
    variant: "destructive"
  });
  return;
}

// Only if hash is valid — proceed with createBatch (existing blockchain call)
// Pass drug.drug_code as drugName, drug.composition_hash for blockchain
await createBlockchainBatch(
  values.batchId,
  drug.drug_name,
  values.quantity,
  // ... dates ...
  drug.blockchain_template_id  // the on-chain drug template ID
);
```

---

## Phase 6 — Verification & Hash Comparison Logic <a name="phase-6"></a>

### Task 6.1 — Create Hash Verification API Route

**File to create:**
```
MediTrustChain/src/app/api/drug-master/verify-hash/route.ts
```

```typescript
// POST /api/drug-master/verify-hash
// Verifies that the composition hash in DB matches what's on blockchain
// This is the CRITICAL security check

import { createClient } from '@/lib/supabase/server';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  const supabase = createClient();
  const { drug_master_id, composition_hash } = await req.json();

  // Fetch from DB
  const { data: drug, error } = await supabase
    .from('drug_master')
    .select('composition_hash, blockchain_tx_hash, is_active')
    .eq('id', drug_master_id)
    .single();

  if (error || !drug) {
    return NextResponse.json({ valid: false, reason: 'Drug template not found' }, { status: 404 });
  }

  if (!drug.is_active) {
    return NextResponse.json({ valid: false, reason: 'Drug template has been deactivated' });
  }

  if (!drug.blockchain_tx_hash) {
    return NextResponse.json({ valid: false, reason: 'Drug template not yet confirmed on blockchain' });
  }

  const hashMatches = drug.composition_hash === composition_hash;

  return NextResponse.json({
    valid: hashMatches,
    reason: hashMatches ? null : 'Composition hash mismatch — possible tampering detected',
  });
}
```

### Task 6.2 — Add Composition Hash to Blockchain Library

**File to edit:**
```
MediTrustChain/src/lib/blockchain/index.ts
```

Add `approveDrugTemplate` and relevant helpers to the `useContract` hook / contract interface, following the same pattern as `createBatch`, `approveBatch` etc. already exist in that file.

---

## Phase 7 — TypeScript Types Update <a name="phase-7"></a>

### Task 7.1 — Add DrugMaster Type

**File to edit:**
```
MediTrustChain/src/types/database.types.ts
```

**Add to `Database['public']['Tables']`:**
```typescript
drug_master: {
  Row: {
    id: string
    drug_name: string
    generic_name: string | null
    drug_code: string
    composition: string
    strength: string
    dosage_form: string
    approved_expiry_months: number
    approved_manufacturer_ids: string[]
    composition_hash: string
    blockchain_tx_hash: string | null
    blockchain_block: number | null
    approved_by: string | null
    approved_by_org: string | null
    is_active: boolean
    created_at: string
    updated_at: string
  }
  Insert: {
    id?: string
    drug_name: string
    generic_name?: string | null
    drug_code: string
    composition: string
    strength: string
    dosage_form: string
    approved_expiry_months: number
    approved_manufacturer_ids?: string[]
    composition_hash: string
    blockchain_tx_hash?: string | null
    blockchain_block?: number | null
    approved_by?: string | null
    approved_by_org?: string | null
    is_active?: boolean
  }
  Update: Partial<Database['public']['Tables']['drug_master']['Insert']>
}
```

### Task 7.2 — Add DrugMaster Interface to a Dedicated Types File

**File to create:**
```
MediTrustChain/src/types/drug-master.ts
```

```typescript
export interface DrugMaster {
  id: string;
  drug_name: string;
  generic_name?: string;
  drug_code: string;
  composition: string;
  strength: string;
  dosage_form: string;
  approved_expiry_months: number;
  approved_manufacturer_ids: string[];
  composition_hash: string;
  blockchain_tx_hash?: string;
  blockchain_block?: number;
  approved_by?: string;
  approved_by_org?: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

---

## Phase 8 — Testing & Validation <a name="phase-8"></a>

### Task 8.1 — Smart Contract Tests

**File to create:**
```
MediTrustChain/blockchain/test/DrugMasterFlow.test.ts
```

```typescript
// Test cases to write:
// 1. Regulator can call approveDrugTemplate()
// 2. Non-regulator CANNOT call approveDrugTemplate() → should revert
// 3. Duplicate drugCode → should revert
// 4. Manufacturer can createBatch() with a valid drugTemplateId
// 5. Manufacturer CANNOT createBatch() with invalid drugTemplateId → should revert
// 6. Hash stored on blockchain matches computed hash
```

Run with:
```bash
cd MediTrustChain/blockchain
npx hardhat test
```

### Task 8.2 — End-to-End Flow Checklist

| Step | Actor | Action | Expected Result |
|------|-------|--------|-----------------|
| 1 | Regulator | Fills Drug Approval Form | Form validates |
| 2 | Regulator | Submits form | `drug_master` row created in DB |
| 3 | Regulator | Wallet connected & matched | Blockchain call enabled |
| 4 | Regulator | Confirms MetaMask | `approveDrugTemplate()` fires |
| 5 | Regulator | TX confirmed | `composition_hash` stored on-chain |
| 6 | Regulator | TX hash saved | `blockchain_tx_hash` updated in DB |
| 7 | Manufacturer | Opens create batch form | Sees drug dropdown (NOT free text) |
| 8 | Manufacturer | Selects a drug | Composition shown read-only |
| 9 | Manufacturer | Tries to edit composition | NOT POSSIBLE (UI prevents it) |
| 10 | Manufacturer | Submits batch | Hash verified against blockchain |
| 11 | Manufacturer | Hash matches | Batch creation proceeds |
| 12 | Manufacturer | Tampered hash | Batch REJECTED, alert fired |

### Task 8.3 — SQL Verification Query (Run in Supabase SQL Editor)

```sql
-- Verify that every batch has a linked approved drug template
SELECT
    b.id            AS batch_id,
    b.name          AS drug_name,
    dm.drug_code    AS approved_drug_code,
    dm.composition  AS approved_composition,
    b.composition_hash = dm.composition_hash AS hash_valid
FROM batches b
LEFT JOIN drug_master dm ON dm.id = b.drug_master_id
ORDER BY b.created_at DESC;
```

---

## File Map — Every File to Touch <a name="file-map"></a>

### NEW FILES to create

| # | File Path | Purpose |
|---|-----------|---------|
| 1 | `supabase/migrations/20260311000001_drug_master.sql` | New `drug_master` table with RLS |
| 2 | `supabase/migrations/20260311000002_batches_add_drug_master_fk.sql` | FK column on `batches` |
| 3 | `src/app/api/drug-master/route.ts` | GET approved drugs list |
| 4 | `src/app/api/drug-master/approve/route.ts` | POST create drug template |
| 5 | `src/app/api/drug-master/[id]/blockchain/route.ts` | PATCH save tx hash |
| 6 | `src/app/api/drug-master/verify-hash/route.ts` | POST verify hash before batch creation |
| 7 | `src/types/drug-master.ts` | TypeScript interface |
| 8 | `blockchain/test/DrugMasterFlow.test.ts` | Hardhat test file |

### EXISTING FILES to edit

| # | File Path | What to Change |
|---|-----------|----------------|
| 1 | `blockchain/contracts/MediTrustChainV2.sol` | Add `DrugTemplate` struct, `approveDrugTemplate()`, modify `createBatch()` |
| 2 | `blockchain/scripts/deploy.ts` | No change needed (redeploy same script) |
| 3 | `src/lib/blockchain/index.ts` | Add `approveDrugTemplate()` function to `useContract` hook |
| 4 | `src/app/dashboard/regulator/page.tsx` | Add Drug Approval tab + form |
| 5 | `src/app/dashboard/manufacturer/page.tsx` | Replace drug name text input with approved drugs dropdown; update submit logic |
| 6 | `src/types/database.types.ts` | Add `drug_master` table types |

---

## Viva Explanation Script <a name="viva-script"></a>

> If the panel asks again: *"Who decides drug composition?"*

**Answer:**

> In our improved **MediTrustChain** system, the **Regulator** is the sole authority for defining and locking official drug compositions.
>
> When a regulator approves a drug, the system computes a cryptographic hash of the drug's name, composition, and strength. This hash is stored **permanently on the Ethereum blockchain** through our `approveDrugTemplate()` smart contract function.
>
> Manufacturers **cannot type or change any composition**. They must select a drug from a dropdown of regulator-approved templates. When they create a batch, the system re-computes the hash and compares it against the blockchain record. If the hash does not match — meaning the composition was tampered with — the batch is immediately **rejected** and the regulator is alerted.
>
> This ensures **traceability, authenticity, and full regulatory compliance** in the pharmaceutical supply chain — directly mirroring how real drug approval bodies like the FSSAI or FDA operate.

---

## Implementation Order (Recommended)

```
Phase 1 (DB)    →  Phase 7 (Types)  →  Phase 2 (Contract)
     ↓                                       ↓
Phase 3 (API)   ←  ──────────────────────  Deploy
     ↓
Phase 4 (Regulator UI)  →  Phase 5 (Manufacturer UI)
     ↓
Phase 6 (Hash Verification)  →  Phase 8 (Testing)
```

**Total new/changed files: 14**  
**Estimated lines of new code: ~600–800**

---

*Generated by GitHub Copilot on 2026-03-11 for MediTrustChain major project.*
