# MediTrustChain — Complete Project Verification Checklist
## For Agent: Check Every Aspect, Every Scenario, End-to-End

> **Instructions for Agent:**
> Go through every task below in order. For each task, open the relevant file(s), read the actual code, and verify the described behavior is implemented correctly. Mark `[x]` when confirmed, note issues inline. Do NOT skip any task. Cover all edge cases and failure scenarios listed under each section.

---

## Module 0 — Project Bootstrap & Environment

- [ ] **0.1** `MediTrustChain/package.json` — Confirm all dependencies are present: `next`, `react`, `@supabase/supabase-js`, `ethers`, `react-hook-form`, `zod`, `@hookform/resolvers`, `qrcode.react`, `html5-qrcode`, `date-fns`, `framer-motion`, `lucide-react`, `tailwindcss`, `@radix-ui/*` components
- [ ] **0.2** `MediTrustChain/next.config.ts` — Check for proper image domains, environment variable references, and that no `NEXT_PUBLIC_` secrets are exposed through `env` config
- [ ] **0.3** `MediTrustChain/tsconfig.json` — Verify `@/` path alias maps to `src/`, strict mode is on
- [ ] **0.4** `MediTrustChain/.env.local` (or env setup) — Confirm required variables exist: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_CONTRACT_ADDRESS`, `NEXT_PUBLIC_RPC_URL`
- [ ] **0.5** `MediTrustChain/blockchain/hardhat.config.cjs` — Confirm network config is valid, Solidity version matches contract (`^0.8.24`)
- [ ] **0.6** `MediTrustChain/blockchain/package.json` — Confirm `hardhat`, `@openzeppelin/contracts`, `ethers`, `@nomicfoundation/hardhat-toolbox` are listed

---

## Module 1 — Database Schema (Supabase Migrations)

### 1A — Core Tables Exist

- [ ] **1.1** Migration `01_create_enums.sql` — Verify `organization_type` and `stakeholder_role` enums both contain: `manufacturer`, `distributor`, `regulator`, `pharmacy`, `logistics`
- [ ] **1.2** Migration `02_create_organizations.sql` — Verify `organizations` table has: `id (UUID PK)`, `name`, `organization_type`, `registration_number (UNIQUE)`, `is_active`, `created_by`, `verification_documents (JSONB)`, `created_at`, `updated_at`
- [ ] **1.3** Migration `03_create_stakeholders.sql` — Verify `stakeholders` table has: `id (UUID PK)`, `organization_id (FK→organizations)`, `full_name`, `email (UNIQUE)`, `user_id (FK→auth.users)`, `wallet_address (UNIQUE)`, `role`, `is_active`, `created_at`
- [ ] **1.4** Migration `04_create_admin_users.sql` — Verify `admin_users` table exists with `id`, `email`, `is_active`
- [ ] **1.5** Migration `06_update_batches.sql` — Verify `batches` table has: `id (VARCHAR(100) PK)`, `name`, `mfg`, `exp`, `qty`, `status`, `organization_id (FK→organizations)`, `created_by_stakeholder_id`, `blockchain_tx_hash`, `blockchain_data_hash`, `created_at`, `updated_at`
- [ ] **1.6** Migration `11_create_batch_history.sql` — Verify `batch_history` table has: `id`, `batch_id (FK→batches)`, `location`, `status`, `timestamp`, `updated_by`
- [ ] **1.7** Migration `12_production_hardening.sql` — Verify `batches.id` is `VARCHAR(100)` (not UUID), foreign keys on `batch_history`, `anomalies`, `notifications` reference `VARCHAR(100)` batch IDs correctly
- [ ] **1.8** Migration `fix_status_mapping.sql` — Verify `batches.status` CHECK constraint allows exactly: `Pending`, `Approved`, `Rejected`, `In-Transit`, `At-Pharmacy`, `Sold`, `Expired`, `Recalled`

### 1B — Drug Master Table (New Feature)

- [ ] **1.9** Migration `20260311000001_drug_master.sql` — Verify `drug_master` table exists with: `id (UUID PK)`, `drug_name`, `drug_code (UNIQUE)`, `composition`, `strength`, `dosage_form`, `approved_expiry_months`, `composition_hash (VARCHAR(66))`, `blockchain_tx_hash`, `approved_by (FK→stakeholders)`, `is_active`, `created_at`, `updated_at`
- [ ] **1.10** Migration `20260311000002_batches_add_drug_master_fk.sql` — Verify `batches` table has new columns: `drug_master_id (FK→drug_master)`, `composition_hash`, `composition`, `strength`

### 1C — Row Level Security (RLS)

- [ ] **1.11** `organizations` RLS — Admin has full access; stakeholders can SELECT their own org; no cross-org info leak
- [ ] **1.12** `stakeholders` RLS — Admin full access; stakeholder can only SELECT/UPDATE their own row; cannot see other orgs' stakeholders
- [ ] **1.13** `batches` RLS — Manufacturer INSERT only for own org; all active stakeholders can SELECT (supply chain needs cross-org read); all active stakeholders can UPDATE (supply chain status updates); verify migration `fix_supply_chain_rls.sql` is applied
- [ ] **1.14** `batch_history` RLS — All active stakeholders can SELECT and INSERT; admin full access
- [ ] **1.15** `drug_master` RLS — Admin full access; regulator can INSERT/UPDATE; all active stakeholders can SELECT
- [ ] **1.16** `cbac_permissions` table — Verify it exists with permission flags: `can_create_batches`, `can_approve_batches`, `can_reject_batches`, `can_recall_batches`, `can_update_status`, `can_view_analytics`, `can_manage_stakeholders`

---

## Module 2 — Smart Contract

**File:** `MediTrustChain/blockchain/contracts/MediTrustChainV2.sol`

### 2A — Enums & Data Structures

- [ ] **2.1** `BatchStatus` enum — Verify all 9 states exist in order: `CREATED(0)`, `PENDING_APPROVAL(1)`, `APPROVED(2)`, `REJECTED(3)`, `IN_TRANSIT(4)`, `AT_PHARMACY(5)`, `SOLD(6)`, `EXPIRED(7)`, `RECALLED(8)`
- [ ] **2.2** `UserRole` enum — Verify: `MANUFACTURER(0)`, `REGULATOR(1)`, `DISTRIBUTOR(2)`, `LOGISTICS(3)`, `PHARMACY(4)`, `PATIENT(5)`
- [ ] **2.3** `BatchCore` struct — Verify immutable fields: `id`, `batchCode`, `manufacturer (address)`, `drugName`, `quantity`, `mfgDate`, `expDate`, `createdAt`, `dataHash (bytes32)`
- [ ] **2.4** `BatchState` struct — Verify mutable fields: `status`, `approvedAt`, `approvalHash`, `currentHolder`, `lastLocation`, `isRecalled`, `lastUpdated`
- [ ] **2.5** `DrugTemplate` struct — Verify fields: `id`, `drugCode`, `drugName`, `composition`, `strength`, `compositionHash (bytes32)`, `approvedBy (address)`, `approvedAt`, `isActive`

### 2B — State Variables & Mappings

- [ ] **2.6** Verify mappings exist: `batchCores`, `batchStates`, `batchHistories`, `users`, `approvedRegulatorsMap`, `batchCodeToBatchId`
- [ ] **2.7** Verify drug template mappings: `drugTemplates`, `drugCodeToTemplateId`, `approvedCompositionHashes`
- [ ] **2.8** Verify counters: `batchIdCounter`, `drugTemplateCounter`

### 2C — Core Functions

- [ ] **2.9** `registerUser(address, UserRole, string)` — Only owner can call; prevents duplicate registration; emits `UserRegistered`
- [ ] **2.10** `approveDrugTemplate(string drugCode, string drugName, string composition, string strength)` — Only registered `REGULATOR` can call; verifies drugCode is unique; computes `keccak256(abi.encodePacked(drugName, composition, strength))`; stores in `drugTemplates` mapping; sets `approvedCompositionHashes[hash] = true`; emits `DrugTemplateCreated`
- [ ] **2.11** `createBatch(string batchCode, string drugName, uint qty, uint mfgDate, uint expDate, uint drugTemplateId)` — Only `MANUFACTURER` role; validates `drugTemplateId` is a real active template; validates dates (mfg < exp); verifies composition hash is approved; computes `dataHash` incorporating `drugTemplateId`; sets status to `CREATED`; emits `BatchCreated`
- [ ] **2.12** `submitForApproval(uint batchId)` — Only batch manufacturer can call; status must be `CREATED`; transitions to `PENDING_APPROVAL`
- [ ] **2.13** `approveBatch(uint batchId, bytes32 documentHash)` — Only approved `REGULATOR`; status must be `PENDING_APPROVAL`; transitions to `APPROVED`; sets `approvalHash`; emits `BatchApproved`
- [ ] **2.14** `rejectBatch(uint batchId, string reason)` — Only approved `REGULATOR`; status must be `PENDING_APPROVAL`; transitions to `REJECTED` (terminal); emits `BatchRejected`
- [ ] **2.15** `updateBatchStatus(uint batchId, BatchStatus newStatus, string location)` — State machine enforced (no skipping); correct roles for each transition (distributor/logistics moves to IN_TRANSIT, logistics to AT_PHARMACY, pharmacy to SOLD); emits `BatchStatusUpdated`
- [ ] **2.16** `recallBatch(uint batchId, string reason)` — Only approved `REGULATOR` or owner; sets `isRecalled = true`; transitions to `RECALLED`; emits `BatchRecalled`
- [ ] **2.17** `verifyBatchAuthenticity(uint batchId)` — Read-only; returns `(bool isAuthentic, bytes32 storedHash, address manufacturer)`
- [ ] **2.18** `pauseContract()` / `unpauseContract()` — Only owner; all write functions have `whenNotPaused` modifier

### 2D — Edge Cases & Security

- [ ] **2.19** Manufacturer cannot call `approveDrugTemplate()` → must revert with role error
- [ ] **2.20** Creating batch with invalid/inactive `drugTemplateId` → must revert
- [ ] **2.21** Contract is paused → all write functions revert; read functions still work
- [ ] **2.22** Non-owner cannot call `pauseContract()` → reverts
- [ ] **2.23** Duplicate `batchCode` → `createBatch` reverts
- [ ] **2.24** Duplicate `drugCode` → `approveDrugTemplate` reverts
- [ ] **2.25** State machine: `APPROVED → CREATED` transition → must revert (backwards skip)
- [ ] **2.26** Updating batch after `SOLD` or `RECALLED` (terminal states) → must revert

---

## Module 3 — Blockchain Library (Frontend)

**Directory:** `MediTrustChain/src/lib/blockchain/`

- [ ] **3.1** `config.ts` — `CONTRACT_ADDRESS` and `RPC_URL` are read from env vars (`NEXT_PUBLIC_*`); fallback handled gracefully; `isBlockchainConfigured()` returns false when vars missing
- [ ] **3.2** `abi.ts` — ABI matches the deployed `MediTrustChainV2.sol`; includes `approveDrugTemplate`, `createBatch`, `approveBatch`, `rejectBatch`, `recallBatch`, `updateBatchStatus`, `verifyBatchAuthenticity`, `registerUser`, `pauseContract`
- [ ] **3.3** `hooks.ts` — `useBlockchain()` hook: exposes `wallet.address`, `wallet.isConnected`, `connectWallet()`, `isLoading`; handles MetaMask not installed; handles wrong network
- [ ] **3.4** `hooks.ts` — `useContract()` hook: exposes `createBatch()`, `approveBatch()`, `rejectBatch()`, `recallBatch()`, `approveDrugTemplate()`, `verifyBatchAuthenticity()`, `getBatchByCode()`, `isRegulatorApproved()`, `generateDataHash()`, `isReady`
- [ ] **3.5** `write.ts` — `createBatchOnChain()`: sends transaction, waits for confirmation, returns `{ txHash, batchId, dataHash }`
- [ ] **3.6** `write.ts` — `approveBatch()`, `rejectBatch()`, `recallBatch()`: each waits for confirmation before returning
- [ ] **3.7** `verification.ts` — `verifyBatchOnBlockchain()`: queries blockchain read-only (no wallet needed); returns `{ isAuthentic, manufacturer, status, dataHash, batchCode }`
- [ ] **3.8** `audit-trail.ts` — Reads on-chain `BatchHistory` events and returns formatted history array
- [ ] **3.9** `wallet-role.ts` — `getWalletRole()`: reads `users[address]` from contract; returns `UserRole` or null if unregistered
- [ ] **3.10** `event-listener.ts` — Listens for `BatchCreated`, `BatchApproved`, `BatchRecalled` events; properly cleans up listeners on unmount

---

## Module 4 — Supabase Client Library

**Directory:** `MediTrustChain/src/lib/supabase/`

- [ ] **4.1** `client.ts` (browser) — Creates Supabase client with `NEXT_PUBLIC_` env vars; singleton pattern to avoid multiple instances
- [ ] **4.2** `server.ts` (server-side) — Creates Supabase client with service role key for API routes; NOT exported for client use
- [ ] **4.3** `batches.ts` — `fetchBatches()`: fetches for current user's organization; `upsertBatch()`: inserts/updates batch; `updateBatchStatusInDb()`: updates status + timestamp; `addBatchHistory()`: inserts into `batch_history`
- [ ] **4.4** `batches.ts` — Error responses are properly typed and surfaced (not silently swallowed)
- [ ] **4.5** Verify `drug_master` table is queryable: `fetch('/api/drug-master')` returns `{ drugs: DrugMaster[] }` for all active stakeholders

---

## Module 5 — Authentication & Authorization

### 5A — Auth Context

**File:** `MediTrustChain/src/contexts/cbac-auth-context.tsx`

- [ ] **5.1** `loginStakeholder(email, password)` — Calls Supabase auth; on success, fetches stakeholder row by `user_id`; sets `stakeholder` and `organization` in context; redirects to `dashboard/[role]`
- [ ] **5.2** `loginAdmin(email, password)` — Checks `admin_users` table after auth; redirects to `dashboard/admin`
- [ ] **5.3** `logout()` — Calls Supabase `signOut()`; clears context; redirects to `/login`
- [ ] **5.4** Context provides: `stakeholder`, `organization`, `isAdmin`, `isLoading`, `loginStakeholder`, `loginAdmin`, `logout`
- [ ] **5.5** On page refresh — Auth state is restored from existing Supabase session (not lost)
- [ ] **5.6** Inactive stakeholder (`is_active = false`) — `loginStakeholder` should throw error "Account is inactive"

### 5B — Middleware (Route Protection)

**File:** `MediTrustChain/src/middleware.ts`

- [ ] **5.7** Unauthenticated user accessing `/dashboard/*` → redirected to `/login`
- [ ] **5.8** Authenticated admin accessing `/dashboard/admin` → allowed through
- [ ] **5.9** Authenticated manufacturer accessing `/dashboard/regulator` → blocked, redirected to their own dashboard
- [ ] **5.10** Authenticated regulator accessing `/dashboard/manufacturer` → blocked
- [ ] **5.11** Public routes accessible without auth: `/`, `/login`, `/signup`, `/verify`, `/about`
- [ ] **5.12** Middleware reads stakeholder role from Supabase (not from client localStorage)

### 5C — Protected Route Component

**File:** `MediTrustChain/src/components/protected-route.tsx`

- [ ] **5.13** `<ProtectedRoute allowedTypes={['manufacturer']}>` — Renders children only if current user's role matches; otherwise shows "Access Denied" or redirects
- [ ] **5.14** Each dashboard page wraps content in the correct `ProtectedRoute` allowedTypes:
  - Admin: `['admin']`
  - Regulator: `['regulator']`
  - Manufacturer: `['manufacturer']`
  - Distributor: `['distributor']`
  - Logistics: `['logistics']`
  - Pharmacy: `['pharmacy']`
  - Analytics: `['manufacturer', 'regulator', 'admin']`

---

## Module 6 — API Routes

**Directory:** `MediTrustChain/src/app/api/`

- [ ] **6.1** `GET /api/drug-master` — Returns `{ drugs: DrugMaster[] }` (active only); accessible to all authenticated stakeholders; returns `[]` not error if none exist
- [ ] **6.2** `POST /api/drug-master/approve` — Validates required fields: `drug_name`, `drug_code`, `composition`, `strength`, `dosage_form`, `approved_expiry_months`; computes `composition_hash`; inserts into `drug_master`; returns `{ drug_master: DrugMaster }`; rejects duplicate `drug_code` with 400
- [ ] **6.3** `PATCH /api/drug-master/[id]/blockchain` — Updates `blockchain_tx_hash` and `blockchain_block` on existing drug master record; returns 404 if `id` not found
- [ ] **6.4** `POST /api/drug-master/verify-hash` — Checks `composition_hash` in DB matches supplied hash; checks `is_active = true`; checks `blockchain_tx_hash` is set (confirmed on-chain); returns `{ valid: boolean, reason: string | null }`
- [ ] **6.5** `GET /api/stakeholders` — Returns stakeholders list; admin can see all; stakeholder can only see their org's stakeholders
- [ ] **6.6** All API routes — Confirm they use server-side Supabase client (not browser client); no service role key exposed to client

---

## Module 7 — Admin Dashboard

**File:** `MediTrustChain/src/app/dashboard/admin/page.tsx`

- [ ] **7.1** Admin can create an Organization — Form with: name, type, registration_number, phone, address, country; validates registration_number is unique; shows success toast
- [ ] **7.2** Admin can create a Stakeholder — Form with: org selection, full_name, email, wallet_address, role, position; creates Supabase Auth user; creates `stakeholders` row; handles email already exists error
- [ ] **7.3** Admin can edit a Stakeholder — Can change: name, phone, position, role, is_active; cannot change email (unique key)
- [ ] **7.4** Admin can delete a Stakeholder — Confirmation dialog before delete; removes from `stakeholders` and `auth.users`
- [ ] **7.5** Admin can register a wallet as Regulator on blockchain — Calls `registerUser(address, REGULATOR, orgName)` on contract; requires admin wallet connected; handles "already registered" error
- [ ] **7.6** Admin can register a wallet as Manufacturer on blockchain — Same flow as 7.5 but with MANUFACTURER role
- [ ] **7.7** Admin can grant Regulator approval on-chain — Calls addApprovedRegulator (or equivalent) so regulator can approve batches
- [ ] **7.8** Admin can pause / unpause the contract — Buttons visible; confirmation dialog; calls `pauseContract()` / `unpauseContract()`; shows current pause state
- [ ] **7.9** Admin can view all Organizations and their Stakeholders in a table
- [ ] **7.10** Admin can view Audit Logs — Table with: timestamp, actor, action, resource; paginated

---

## Module 8 — Drug Master Approval Flow (Regulator)

**Files:**
- `MediTrustChain/src/app/dashboard/regulator/page.tsx`
- `MediTrustChain/src/components/dashboard/regulator-drug-approval.tsx`

### 8A — Drug Template Creation

- [ ] **8.1** Regulator dashboard has a "Drug Templates" tab (or section) with a drug approval form
- [ ] **8.2** Form fields present: Drug Code, Drug Name, Generic Name, Composition, Strength, Dosage Form, Approved Expiry (months)
- [ ] **8.3** Validation with `zod`: all required fields enforced; drug code must be unique format (e.g. `DRG-XXX`)
- [ ] **8.4** Submit step 1 — Calls `POST /api/drug-master/approve`; shows loading spinner; handles API error
- [ ] **8.5** Submit step 2 — After DB record created, calls `approveDrugTemplate()` on blockchain via MetaMask; wallet must be connected and matched to stakeholder wallet
- [ ] **8.6** Submit step 3 — After blockchain TX confirmed, calls `PATCH /api/drug-master/[id]/blockchain` to save tx hash
- [ ] **8.7** On success — Shows toast with TX hash; refreshes approved drugs list; clears form
- [ ] **8.8** Wallet mismatch warning — If connected wallet ≠ stakeholder's assigned wallet, blockchain section is disabled with a clear error message

### 8B — Viewing Approved Templates

- [ ] **8.9** Approved drugs table shows: Drug Code, Drug Name, Composition, Strength, Dosage Form, Blockchain Status (confirmed/pending), Approved Date
- [ ] **8.10** Blockchain TX hash is a clickable link to the block explorer
- [ ] **8.11** Inactive (deactivated) templates are visually distinct (greyed out or hidden)

---

## Module 9 — Batch Creation (Manufacturer)

**File:** `MediTrustChain/src/app/dashboard/manufacturer/page.tsx`

### 9A — Form & Validation

- [ ] **9.1** Batch creation form does NOT have a free-text composition or drug name field — drug is selected from dropdown only
- [ ] **9.2** Drug dropdown is populated from `GET /api/drug-master` on component mount; shows `{ drug_name } — { drug_code } ({ strength })`
- [ ] **9.3** When a drug is selected from dropdown — Composition, Strength, and Dosage Form are displayed as read-only (not editable)
- [ ] **9.4** "Regulator Approved" badge is shown next to the selected drug's composition
- [ ] **9.5** Form fields: Batch ID, Drug (dropdown), Manufacturing Date (date picker), Expiry Date (date picker), Quantity
- [ ] **9.6** Zod validation: Batch ID required; Drug selection required (`uuid`); mfg date required; exp date required and must be after mfg date; quantity > 0

### 9B — Submission Flow

- [ ] **9.7** On submit — First calls `POST /api/drug-master/verify-hash` with selected drug's `id` and `composition_hash`; if `valid: false` → stops with error toast; batch NOT created
- [ ] **9.8** Blockchain toggle switch — If enabled, calls `createBatch()` on contract with `drugTemplateId` from selected drug; waits for MetaMask confirmation
- [ ] **9.9** After blockchain success — Calls `addBatch()` from `useBatches()` context which upserts to Supabase `batches` table with `drug_master_id`, `composition_hash` fields set
- [ ] **9.10** On Supabase success — Shows QR code dialog with batch ID and blockchain TX hash
- [ ] **9.11** Batch appears in the "My Batches" table immediately (optimistic update or re-fetch)
- [ ] **9.12** Wallet not connected — Blockchain section shows a "Connect Wallet" prompt; form can still be submitted for DB-only mode
- [ ] **9.13** Wallet mismatch — Blockchain submission disabled; warns user

### 9C — Batch Table

- [ ] **9.14** Batches table shows: Batch ID, Drug Name, Mfg Date, Exp Date, Qty, Status badge, Blockchain TX (link)
- [ ] **9.15** QR code can be generated for any batch in the table
- [ ] **9.16** Status badge colors are correct: Pending=yellow, Approved=green, Rejected=red, In-Transit=blue, etc.

---

## Module 10 — Batch Approval (Regulator)

**File:** `MediTrustChain/src/app/dashboard/regulator/page.tsx`

- [ ] **10.1** Batches table shows all batches in `Pending` status (from manufacturer) — across all organizations
- [ ] **10.2** Regulator can click "View" on a batch to see full details including: Drug Name, Composition (from `drug_master`), Blockchain data hash, history
- [ ] **10.3** Batch details show the **approved drug template** that was used — composition is cross-referenced
- [ ] **10.4** "Approve" button — Opens confirmation dialog; calls `approveBatch(batchId, documentHash)` on blockchain; on TX confirmed, calls `updateBatchStatus('Approved')` in Supabase; sends notification to manufacturer
- [ ] **10.5** "Reject" button — Opens dialog with rejection reason textarea; requires reason to be non-empty; calls `rejectBatch(batchId, reason)` on blockchain; updates DB to `Rejected`
- [ ] **10.6** "Recall" button — Available for `Approved`, `In-Transit`, `At-Pharmacy` batches; requires recall reason; calls `recallBatch(batchId, reason)`; updates DB to `Recalled`
- [ ] **10.7** Anomaly Detection section — Shows detected anomalies (timing, location, quantity); regulator can acknowledge or escalate
- [ ] **10.8** Compliance log tab — Shows audit trail of regulator actions with timestamps

---

## Module 11 — Distributor Dashboard

**File:** `MediTrustChain/src/app/dashboard/distributor/page.tsx`

- [ ] **11.1** Shows only batches in `Approved` status (ready for distribution)
- [ ] **11.2** Distributor can update batch status to `In-Transit` — enters location/shipment info; calls `updateBatchStatus()` on blockchain and Supabase
- [ ] **11.3** Cannot move batch BACK to `Approved` from `In-Transit` (state machine respected)
- [ ] **11.4** Batch history is appended with distributor's action (timestamp, location, actor)
- [ ] **11.5** Cannot access pharmacy or regulator functionality

---

## Module 12 — Logistics Dashboard

**File:** `MediTrustChain/src/app/dashboard/logistics/page.tsx`

- [ ] **12.1** Shows batches in `In-Transit` status
- [ ] **12.2** Logistics can update status to `At-Pharmacy` — enters delivery location and timestamp
- [ ] **12.3** Blockchain and Supabase updated atomically (or Supabase updated after blockchain confirmation)
- [ ] **12.4** Cannot approve or reject batches
- [ ] **12.5** History entry created with logistics actor's identity

---

## Module 13 — Pharmacy Dashboard

**File:** `MediTrustChain/src/app/dashboard/pharmacy/page.tsx`

- [ ] **13.1** Shows batches in `At-Pharmacy` status
- [ ] **13.2** Pharmacy can verify batch authenticity before recording sale — calls `verifyBatchAuthenticity()` on blockchain (read-only, no wallet needed)
- [ ] **13.3** If authenticity fails → sale is blocked; warning shown
- [ ] **13.4** "Record Sale" button — Updates status to `Sold` on blockchain and Supabase; this is a terminal state
- [ ] **13.5** Cannot perform any action on batches in `Recalled` or `Rejected` state

---

## Module 14 — Patient / Public Verification

**File:** `MediTrustChain/src/app/verify/page.tsx`

- [ ] **14.1** Page is publicly accessible (no login required)
- [ ] **14.2** Text input to enter Batch Code manually
- [ ] **14.3** QR scanner (using `html5-qrcode`) to scan physical drug QR code
- [ ] **14.4** On verify — Calls `verifyBatchOnBlockchain()` from `lib/blockchain/verification.ts`; NO wallet required (read-only)
- [ ] **14.5** Success result shows: Drug Name, Manufacturer, Mfg Date, Exp Date, Current Status, Blockchain TX, Data Hash
- [ ] **14.6** If batch not found on blockchain → clear "Not Found" message, no crash
- [ ] **14.7** If batch hash mismatch → "⚠ Authenticity Failed — Possible Counterfeit" warning shown prominently
- [ ] **14.8** If batch is `Recalled` → "⚠ This batch has been recalled. Do not use." shown prominently
- [ ] **14.9** If batch is `Expired` → expiry warning shown
- [ ] **14.10** Mobile-friendly (QR scanner works on phone camera)

---

## Module 15 — Analytics Dashboard

**File:** `MediTrustChain/src/app/dashboard/analytics/page.tsx`

- [ ] **15.1** Accessible to: `manufacturer`, `regulator`, `admin` (verify ProtectedRoute)
- [ ] **15.2** Batch status distribution chart — Correct counts per status
- [ ] **15.3** Geographic distribution / supply chain map — Shows batch locations from `batch_history`
- [ ] **15.4** Demand forecasting section — Uses AI/statistical data from `lib/analytics/`
- [ ] **15.5** Performance metrics — Avg approval time, avg transit time, recall rate
- [ ] **15.6** Data respects org isolation — Manufacturer sees only their org's analytics; regulator sees all

---

## Module 16 — Recall Management

**File:** `MediTrustChain/src/app/dashboard/recalls/page.tsx`
**Component:** `MediTrustChain/src/components/dashboard/recall-management.tsx`

- [ ] **16.1** Shows all batches with `Recalled` status
- [ ] **16.2** Recall reason is displayed for each recalled batch
- [ ] **16.3** Shows recall initiation timestamp and who initiated
- [ ] **16.4** Blockchain `RECALLED` status is confirmed (shows on-chain proof)
- [ ] **16.5** Recall affects all downstream actors — notifications sent to distributor, logistics, pharmacy

---

## Module 17 — Notifications System

**File:** `MediTrustChain/src/contexts/notifications-context.tsx`
**Component:** `MediTrustChain/src/components/dashboard/notifications.tsx`

- [ ] **17.1** Notifications are persisted in Supabase `notifications` table
- [ ] **17.2** Manufacturer gets notified when: batch is approved, batch is rejected, batch is recalled
- [ ] **17.3** Regulator gets notified when: new batch is submitted for approval
- [ ] **17.4** Distributor/Logistics/Pharmacy get notified when: batch enters their workflow stage
- [ ] **17.5** Notification bell icon in nav shows unread count badge
- [ ] **17.6** Notifications can be marked as read

---

## Module 18 — Tampering Detection Demo

**File:** `MediTrustChain/src/app/dashboard/demo/page.tsx`

- [ ] **18.1** Demo scenarios exist: Counterfeit batch, expired date tampering, composition mismatch
- [ ] **18.2** Each scenario shows: Original data vs Tampered data side-by-side
- [ ] **18.3** Hash comparison is demonstrated clearly — original hash ≠ tampered hash
- [ ] **18.4** Detection method is explained in plain text
- [ ] **18.5** "Reset Demo" button works correctly

---

## Module 19 — CBAC (Claims-Based Access Control) Library

**Directory:** `MediTrustChain/src/lib/cbac/`

- [ ] **19.1** `access-control.ts` — `validateOrganizationAccess(userId, orgId)`: returns true only if the user is an active stakeholder in that org
- [ ] **19.2** `service.ts` — provides functions to get permissions for a stakeholder: `getStakeholderPermissions(stakeholderId)`
- [ ] **19.3** Permissions are enforced in the batch API (not just in UI) — manufacturer cannot approve, regulator cannot create
- [ ] **19.4** `cbac-management.tsx` component — Admin can view and edit permissions per stakeholder

---

## Module 20 — IPFS Integration (if implemented)

**Directory:** `MediTrustChain/src/lib/ipfs/`

- [ ] **20.1** File upload to IPFS returns a valid CID
- [ ] **20.2** CID is stored in batch metadata or `drug_master` record
- [ ] **20.3** Files can be retrieved by CID
- [ ] **20.4** IPFS gateway URL is configurable via env var (not hardcoded)

---

## Module 21 — AI Integration (Genkit)

**Directory:** `MediTrustChain/src/ai/`

- [ ] **21.1** `genkit.ts` — AI client is configured; API key from env var (not hardcoded)
- [ ] **21.2** `about-page-ai-assistance.ts` flow — AI chatbot works on the About page; returns relevant responses
- [ ] **21.3** AI chatbot component (`components/ai-chatbot.tsx`) — Input is sanitized before sending to AI; no prompt injection risk
- [ ] **21.4** AI responses do not leak system prompts or internal data

---

## Module 22 — Email Integration

**Directory:** `MediTrustChain/src/lib/email/`
**API:** `MediTrustChain/src/app/api/email/`

- [ ] **22.1** Stakeholder account creation sends a welcome email with login credentials
- [ ] **22.2** Batch approval/rejection triggers email to manufacturer
- [ ] **22.3** Recall trigger sends urgent email to all relevant parties
- [ ] **22.4** Email API route is server-side only; email provider credentials NOT in client bundle

---

## Module 23 — Export & Reports

**Directory:** `MediTrustChain/src/lib/export/`

- [ ] **23.1** PDF export of batch details works (all fields populated)
- [ ] **23.2** CSV export of batch list works; columns match table
- [ ] **23.3** Export respects access control (user can only export their org's data)

---

## Module 24 — Sentry Error Monitoring

**Files:** `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`

- [ ] **24.1** Sentry DSN comes from env var, not hardcoded
- [ ] **24.2** Client-side errors are captured
- [ ] **24.3** Server-side API route errors are captured
- [ ] **24.4** No PII (email, name, wallet address) is sent to Sentry in error payloads

---

## Module 25 — Security Audit (OWASP Top 10)

- [ ] **25.1 Broken Access Control** — API routes check Supabase RLS; no endpoint skips auth; admin endpoints not accessible by stakeholders
- [ ] **25.2 Cryptographic Failures** — No plaintext passwords; Supabase handles auth; composition hash uses SHA-256/keccak256; no weak hashing (MD5, SHA1)
- [ ] **25.3 Injection** — All DB queries use Supabase SDK (parameterized); no raw SQL with user input; no `eval()` or dynamic code execution
- [ ] **25.4 XSS** — No `dangerouslySetInnerHTML` with unsanitized input; user-facing text is escaped by React
- [ ] **25.5 Insecure Design** — Composition is locked at Regulator level (panelist feedback addressed); manufacturer cannot self-certify composition
- [ ] **25.6 Security Misconfiguration** — `.env.local` is in `.gitignore`; service role key not in `NEXT_PUBLIC_`; RLS enabled on ALL tables
- [ ] **25.7 Vulnerable Components** — Run `npm audit` in both `MediTrustChain/` and `MediTrustChain/blockchain/`; no critical vulnerabilities
- [ ] **25.8 Auth Failures** — No hardcoded credentials anywhere in source; test accounts removed from seed.sql before production
- [ ] **25.9 Integrity Failures** — Smart contract functions emit events; DB changes produce audit logs; no unsigned data trusted from client
- [ ] **25.10 SSRF** — No server-side URL fetching from user input; IPFS gateway URL is env-configured, not user-supplied

---

## Module 26 — End-to-End Supply Chain Scenarios

> **For each scenario, trace the ENTIRE flow manually through the code and verify each step works.**

### Scenario A — Happy Path (Full lifecycle)

- [ ] **26.A.1** Admin creates Pharma Corp org + creates John (manufacturer stakeholder with wallet 0xABC)
- [ ] **26.A.2** Admin creates RegBody org + creates Jane (regulator stakeholder with wallet 0xDEF)
- [ ] **26.A.3** Admin registers 0xABC on blockchain as MANUFACTURER; registers 0xDEF as REGULATOR; grants 0xDEF regulator approval
- [ ] **26.A.4** Jane (Regulator) logs in → goes to Drug Templates → approves "Paracetamol 500mg Tablet" (DRG-001) → TX confirmed on blockchain
- [ ] **26.A.5** John (Manufacturer) logs in → creates batch BCH-001 → selects DRG-001 from dropdown → composition shown read-only → submits → hash verified → TX confirmed
- [ ] **26.A.6** Jane (Regulator) sees BCH-001 in Pending list → approves it → TX confirmed → John gets notification
- [ ] **26.A.7** Distributor logs in → sees BCH-001 in Approved → marks as "In-Transit" from "Mumbai Warehouse"
- [ ] **26.A.8** Logistics logs in → sees BCH-001 In-Transit → marks as "At-Pharmacy" at "Delhi Pharmacy"
- [ ] **26.A.9** Pharmacy logs in → sees BCH-001 At-Pharmacy → verifies authenticity → records sale → status = Sold
- [ ] **26.A.10** Patient goes to `/verify` → scans QR or enters BCH-001 → sees "✅ Authentic · Sold" with full details

### Scenario B — Rejected Batch

- [ ] **26.B.1** Manufacturer creates batch BCH-002
- [ ] **26.B.2** Regulator rejects BCH-002 with reason "Quality inspection failed"
- [ ] **26.B.3** BCH-002 status = `Rejected` in DB and on blockchain
- [ ] **26.B.4** Manufacturer receives notification with rejection reason
- [ ] **26.B.5** Distributor CANNOT see or act on BCH-002 (Rejected ≠ Approved)
- [ ] **26.B.6** Patient scans BCH-002 → sees "Rejected" status clearly

### Scenario C — Recalled Batch

- [ ] **26.C.1** Batch BCH-003 reaches `In-Transit` state
- [ ] **26.C.2** Regulator initiates recall with reason "Contamination detected"
- [ ] **26.C.3** BCH-003 transitions to `Recalled` on blockchain (terminal state)
- [ ] **26.C.4** All parties (distributor, logistics, pharmacy) receive recall notifications
- [ ] **26.C.5** Patient scans BCH-003 → sees prominent "⚠ RECALLED — Do Not Use" warning
- [ ] **26.C.6** No further status updates possible on BCH-003

### Scenario D — Composition Tampering Attempt

- [ ] **26.D.1** Manufacturer selects DRG-001 (Paracetamol 500mg)
- [ ] **26.D.2** The dropdown/form does NOT allow editing of composition
- [ ] **26.D.3** If frontend is bypassed (direct API call with wrong hash) → `POST /api/drug-master/verify-hash` returns `{ valid: false }`
- [ ] **26.D.4** Batch creation is blocked with "Hash Mismatch" error
- [ ] **26.D.5** If blockchain call is bypassed (someone calls createBatch with wrong drugTemplateId) → smart contract reverts
- [ ] **26.D.6** Demo page correctly illustrates this scenario

### Scenario E — Wrong Role Access Attempt

- [ ] **26.E.1** Manufacturer tries to access `/dashboard/regulator` → middleware blocks, redirected to `/dashboard/manufacturer`
- [ ] **26.E.2** Regulator tries to create a batch → ProtectedRoute blocks
- [ ] **26.E.3** Unauthenticated user tries to access any `/dashboard/*` → redirected to `/login`
- [ ] **26.E.4** Expired session → user automatically redirected to `/login` (not stuck on blank page)

### Scenario F — Blockchain Not Configured / Offline

- [ ] **26.F.1** `isBlockchainConfigured()` returns false → all blockchain UI sections show "Configure blockchain to enable" message
- [ ] **26.F.2** Batch creation WITHOUT blockchain → batch is saved to Supabase only; no TX hash; status = Pending
- [ ] **26.F.3** MetaMask not installed → "Install MetaMask" prompt shown (not a crash)
- [ ] **26.F.4** Wrong network in MetaMask → "Switch to [expected network]" prompt shown

---

## Module 27 — Performance & UX

- [ ] **27.1** Batch list loads with loading skeleton (not blank flash)
- [ ] **27.2** Drug dropdown in manufacturer form loads without noticeable delay (or shows skeleton/spinner while loading)
- [ ] **27.3** QR code renders correctly for batch codes with special characters
- [ ] **27.4** Date pickers are constrained (expiry must be after manufacturing date)
- [ ] **27.5** Forms show field-level validation errors (not just toast)
- [ ] **27.6** Mobile layout — All dashboards are usable on 375px width (no horizontal scroll)
- [ ] **27.7** Dark/light theme toggle works across all pages (`theme-toggle.tsx`, `theme-context.tsx`)
- [ ] **27.8** After a long blockchain TX wait, UI shows "Waiting for confirmation..." (not frozen)

---

## Module 28 — Build & Deployment Verification

```bash
# Run these commands and verify zero errors:
cd MediTrustChain
npm run build          # No TypeScript or Next.js build errors
npm run lint           # No ESLint errors

cd blockchain
npx hardhat compile    # No Solidity compilation errors
npx hardhat test       # All tests pass
```

- [ ] **28.1** `npm run build` completes successfully with no TypeScript errors
- [ ] **28.2** `npm run lint` — Zero errors (warnings acceptable)
- [ ] **28.3** `npx hardhat compile` — Zero errors
- [ ] **28.4** `npx hardhat test` — All tests pass (if test file exists)
- [ ] **28.5** No `any` TypeScript type in security-critical paths (auth, API routes, blockchain calls)
- [ ] **28.6** No unused imports that could cause dead code paths

---

## Quick Checklist Summary (For Viva / Demo Day)

| Actor | Can Do | Cannot Do |
|-------|--------|-----------|
| **Admin** | Create orgs, stakeholders, register wallets on-chain, pause contract, view all data, grant regulator approval | Create/approve batches |
| **Regulator** | Approve drug templates (lock composition), approve/reject/recall batches, view all batches, view analytics | Create batches, modify composition after approval |
| **Manufacturer** | Create batches (from approved templates only), view own batches, generate QR | Approve batches, define composition |
| **Distributor** | Update batch status to In-Transit | Approve/reject/create batches |
| **Logistics** | Update batch status to At-Pharmacy | Approve/reject/create batches |
| **Pharmacy** | Verify authenticity, record sale (Sold) | Approve/reject/recall/create batches |
| **Patient** | Scan QR/verify batch publicly | ANY data modification |

---

*Agent: Report each failed check with the file path, line number, and the specific issue. Group failures by module number.*
