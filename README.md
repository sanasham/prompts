# prompts

You are a senior banking solutions architect.

Design a production-ready loan product ingestion system using ReactJS (frontend), Node.js with Express (backend), and SQL Server (database).

The system ingests monthly Excel uploads (~2500 loan products) for a banking loan department. Each loan product includes product ID, product name, start date, withdrawn date, and pricing. Pricing may change multiple times per year.

Key requirements:

Single-writer system (no concurrent updates from multiple systems)

Use a staging + batch-driven architecture for auditability and regulatory compliance

Track upload batches with full lineage (who, when, status, counts)

Validate data before committing to production tables

Use TVP-based MERGE for optimized bulk insert/update into core tables

Maintain full history of changes (pricing changes over time, with timestamps and batch IDs)

Allow replay and recovery of failed batches

Provide end-user feedback after upload showing:

total records processed

how many were newly created

how many were updated

list of created and updated product names

Deliver:

High-Level Design (HLD)

Low-Level Design (LLD)

Database schema (staging, core, history, batch tables)

SQL Server MERGE with OUTPUT examples

Node.js API flow and response mapping

UI behavior for displaying pricing change history

Non-functional requirements (performance, audit, security, recovery)

The design should follow bank-grade best practices and be suitable for production deployment.

====================================================================================================

Short Prompt (When You Want Fast but Correct Output)

Design a banking-grade batch ingestion system for loan products using ReactJS, Node.js (Express), and SQL Server.

Monthly Excel uploads (~2500 records) must be staged, validated, batch-tracked, merged into core tables using TVP + MERGE, and fully audited with change history.

Include HLD, LLD, schema, MERGE OUTPUT, Node.js response mapping, retry/replay strategy, and UI behavior for pricing changes.


==========================================================================================================

Architecture-Only Prompt (For HLD Reviews / Interviews)

As a banking architect, explain a staging-driven, batch-controlled data ingestion architecture for loan products with monthly Excel uploads.

Cover auditability, recovery, TVP-based MERGE optimization, history tracking, and user feedback.



==================================================================================================================



Production-Ready System Prompt: Loan Product Management with Staging + Batch Processing
You are a senior software architect and backend engineer specializing in banking and financial systems.

Design and implement a production-ready Loan Product Management system using the STAGING + BATCH PROCESSING pattern.

## BUSINESS CONTEXT

A bank's loan department manages ~49 loan products. Every month, they receive an Excel file with ~2500 loan product records representing active products for that month.

Each loan product record includes:
- Product ID (unique identifier)
- Product Name
- Loan Start Date
- Withdrawn Date (NULL = active, date value = inactive)
- Pricing (interest rate percentage, e.g., 5.25%)

## CRITICAL BUSINESS RULES

1. **Compliance**: Loan products must NEVER be deleted (banking regulation requirement)
2. **Active/Inactive**: A product's status is determined by its withdrawn date (NULL or future = active, past = inactive)
3. **Monthly Source of Truth**: Excel uploads are the authoritative data source
4. **Change Detection**: When processing uploads:
   - If product exists and nothing changed → skip (no database write)
   - If product exists and pricing/withdrawn date/name changed → update
   - If product doesn't exist → create new
5. **Complete Audit Trail**: Every change must be tracked with:
   - Who made the change (username)
   - When it was made (timestamp)
   - What changed (old value → new value)
6. **Pricing History**: Must support queries like "How many times did this product's pricing change in the last 12 months?"
7. **Reconciliation Summary**: After each upload, business users must receive a detailed report showing:
   - Total records processed
   - Number created
   - Number updated  
   - Number with validation errors
   - List of created products (ID, Name, Pricing)
   - List of updated products (ID, Name, what changed)
   - List of invalid products (ID, validation errors)

## TECHNICAL ARCHITECTURE REQUIREMENTS

### Technology Stack
- **Frontend**: React.js
- **Backend**: Node.js with Express.js
- **Database**: Microsoft SQL Server
- **File Processing**: Excel files (.xlsx)

### STAGING + BATCH PROCESSING PATTERN (MANDATORY)

This is NOT a single-transaction system. You MUST implement:

1. **Three-Stage Processing Pipeline**:
Stage 1: Upload → Staging Table (immediate, no validation)
Stage 2: Validation → Mark valid/invalid in staging (fast)
Stage 3: Batch Processing → Process in chunks, commit to production tables (background, resilient)

2. **Database Tables Required**:
   - `LoanProducts` - Current state of all products (production data)
   - `LoanProductHistory` - Complete audit trail of all changes
   - `LoanProductsStaging` - Temporary holding area for uploaded data
   - `UploadBatches` - Track each upload batch with status
   - `ProcessingLog` - Granular log of chunk processing
   - `Users` - User information for audit trail

3. **Batch Processing Characteristics**:
   - Process in chunks of 500 records per transaction
   - Each chunk commits independently (if chunk 5 fails, chunks 1-4 are preserved)
   - Track progress in real-time (percentage complete)
   - Use separate INSERT and UPDATE statements (NOT MERGE - see below)
   - Support recovery and retry mechanisms

4. **Why NOT Use MERGE Statement**:
   - MERGE has documented concurrency issues in SQL Server
   - Difficult to debug in production
   - Microsoft recommends separate INSERT/UPDATE for critical systems
   - Banking systems require predictable, traceable operations

### NON-FUNCTIONAL REQUIREMENTS

**Performance**:
- Upload acknowledgment must return within 5 seconds
- Batch processing should complete 2500 records in under 60 seconds
- UI should show real-time progress (poll every 2 seconds)
- Database queries for product list/detail must complete in < 2 seconds

**Resilience**:
- If processing fails mid-batch, already-processed chunks must be preserved
- Support for retry of failed batches
- No data loss on network/server failures
- Idempotent operations (can safely re-run)

**Auditability**:
- Every upload stored in staging table permanently (7-year retention)
- Every change tracked in history table
- Complete lineage from Excel row → staging → production
- All timestamps in UTC

**Security**:
- File upload size limit: 10MB
- File type validation: Only .xlsx files
- Input sanitization and validation
- SQL injection prevention
- Authentication required (username tracked)

**Scalability**:
- Current: 2,500 records/month
- Must support: Up to 10,000 records/month
- Chunk size configurable
- Connection pooling enabled

## DETAILED DELIVERABLES REQUIRED

### 1. DATABASE DESIGN

Provide complete SQL scripts including:

**Table: LoanProducts**
- ProductID (PK, NVARCHAR(50))
- ProductName (NVARCHAR(255))
- LoanStartDate (DATE)
- WithdrawnDate (DATE, nullable)
- Pricing (DECIMAL(5,2))
- IsActive (computed column based on WithdrawnDate)
- CreatedDate, CreatedBy, UpdatedDate, UpdatedBy
- Appropriate indexes

**Table: LoanProductHistory**
- HistoryID (PK, BIGINT IDENTITY)
- ProductID
- ProductName
- OldPricing, NewPricing
- OldWithdrawnDate, NewWithdrawnDate
- ChangeType (INSERT/UPDATE)
- ChangeDate, ChangedBy
- Appropriate indexes for querying by ProductID and date range

**Table: LoanProductsStaging**
- StagingID (PK, BIGINT IDENTITY)
- BatchID (UNIQUEIDENTIFIER, groups this upload)
- RowNumber (INT, original Excel row number)
- ProductID, ProductName, LoanStartDate, WithdrawnDate, Pricing (from Excel)
- ValidationStatus (PENDING/VALID/INVALID/PROCESSED)
- ValidationErrors (NVARCHAR(MAX), stores error messages)
- ProcessedDate (when moved to production)
- UploadedDate, UploadedBy
- Appropriate indexes

**Table: UploadBatches**
- BatchID (PK, UNIQUEIDENTIFIER)
- FileName (original Excel filename)
- TotalRecords, ValidRecords, InvalidRecords, ProcessedRecords
- BatchStatus (UPLOADED/VALIDATING/VALIDATED/PROCESSING/COMPLETED/FAILED)
- UploadedDate, UploadedBy
- ProcessingStarted, ProcessingCompleted
- Appropriate indexes

**Table: ProcessingLog**
- LogID (PK, BIGINT IDENTITY)
- BatchID, ChunkNumber
- RecordsProcessed, RecordsCreated, RecordsUpdated, RecordsSkipped
- ProcessingTime (milliseconds)
- LogDate
- Appropriate indexes

**Table: Users**
- UserID, Username, Email, FullName, IsActive, CreatedDate

### 2. STORED PROCEDURES

**usp_ValidateStagingBatch**
- Input: @BatchID
- Validates all staging records for this batch
- Updates ValidationStatus to VALID/INVALID
- Sets ValidationErrors for invalid records
- Updates UploadBatches with validation counts

**usp_ProcessChunk**
- Input: @BatchID, @ChunkNumber, @ChunkSize, @Username
- Processes one chunk of valid staging records
- Uses separate UPDATE then INSERT (NOT MERGE)
- Inserts audit records into LoanProductHistory
- Marks staging records as PROCESSED
- Returns counts (created, updated, skipped)
- Must be transactional (all or nothing per chunk)

**usp_GetBatchStatus**
- Input: @BatchID
- Returns comprehensive batch status including progress percentage

**usp_GetProducts** (with pagination)
- Input: @PageNumber, @PageSize, @SearchTerm, @ActiveOnly
- Returns products from LoanProducts table with total count

**usp_GetProductById**
- Input: @ProductID
- Returns single product details

**usp_GetProductHistory**
- Input: @ProductID, @MonthsBack
- Returns pricing change history for analysis

### 3. BACKEND IMPLEMENTATION (Node.js + Express)

Provide complete, production-ready code:

**Project Structure**:
src/
├── config/
│   ├── database.js (SQL Server connection with pooling)
│   └── app.config.js
├── controllers/
│   ├── uploadController.js
│   ├── productController.js
│   └── batchController.js
├── services/
│   ├── excelParserService.js (parse Excel, transform to objects)
│   ├── uploadService.js (insert to staging)
│   ├── validationService.js (validate batch)
│   ├── batchProcessingService.js (background chunk processing)
│   └── productService.js (query products, history)
├── middleware/
│   ├── auth.js (authentication)
│   ├── errorHandler.js
│   └── fileUpload.js (multer config)
├── routes/
│   ├── upload.routes.js
│   ├── product.routes.js
│   └── batch.routes.js
└── app.js

**Key Service Requirements**:

**ExcelParserService**:
- Use 'xlsx' library to read .xlsx files
- Handle Excel date serial numbers correctly
- Validate data types during parsing
- Return array of product objects with detailed error reporting
- Extract from columns: ProductID, ProductName, LoanStartDate, WithdrawnDate, Pricing

**UploadService**:
- Method: `insertToStaging(batchId, products, username)`
- Create batch record in UploadBatches
- Bulk insert all products to LoanProductsStaging (using sql.Table)
- All inserts in single transaction
- Return batchId immediately to user

**ValidationService**:
- Method: `validateBatch(batchId)`
- Run SQL UPDATE to validate all staging records
- Check: Pricing between 0-100, ProductID not empty, ProductName not empty, dates valid
- Update ValidationStatus and ValidationErrors columns
- Update UploadBatches with validation counts
- Can run asynchronously

**BatchProcessingService**:
- Method: `processBatchAsync(batchId)` - fire and forget
- Runs in background (don't await in controller)
- Calculate total chunks based on valid record count
- Loop through chunks, calling `processChunk()` for each
- Each chunk runs in its own transaction
- Log each chunk to ProcessingLog table
- Update UploadBatches status throughout (PROCESSING → COMPLETED/FAILED)
- On any chunk failure: log error, mark batch as FAILED, but keep already-processed chunks

**ProcessChunk Implementation**:
- Fetch chunk from staging (OFFSET/FETCH)
- Execute usp_ProcessChunk stored procedure
- Handle transaction per chunk
- Return statistics (created, updated, skipped)
- Log to ProcessingLog

**ProductService**:
- `getProducts(page, pageSize, searchTerm, activeOnly)`
- `getProductById(productId)`
- `getProductHistory(productId, monthsBack)`

### 4. API CONTRACTS

**POST /api/upload**
- Request: multipart/form-data with file field
- Response (immediate, within 5 seconds):
```json
{
  "success": true,
  "batchId": "uuid",
  "totalRecords": 2500,
  "status": "VALIDATING",
  "message": "Upload received. Validation in progress...",
  "statusUrl": "/api/batches/{batchId}/status"
}
```

**GET /api/batches/:batchId/status**
- Used for polling batch progress
- Response:
```json
{
  "batchId": "uuid",
  "status": "PROCESSING",
  "fileName": "monthly_loan_products.xlsx",
  "totalRecords": 2500,
  "validRecords": 2485,
  "invalidRecords": 15,
  "processedRecords": 1500,
  "progressPercentage": 60.4,
  "chunksCompleted": 3,
  "totalChunks": 5,
  "statistics": {
    "created": 12,
    "updated": 98,
    "skipped": 1390
  },
  "uploadedDate": "2024-02-01T10:00:00Z",
  "uploadedBy": "jsmith",
  "processingStarted": "2024-02-01T10:00:05Z"
}
```

**GET /api/batches/:batchId/reconciliation**
- Returns final reconciliation report after completion
- Response:
```json
{
  "batchId": "uuid",
  "status": "COMPLETED",
  "summary": {
    "totalRecords": 2500,
    "created": 15,
    "updated": 127,
    "unchanged": 2343,
    "invalid": 15
  },
  "createdProducts": [
    { "productId": "LP-2501", "productName": "Green Loan", "pricing": 5.25 }
  ],
  "updatedProducts": [
    {
      "productId": "LP-002",
      "productName": "Home Loan",
      "changes": {
        "pricing": { "from": 6.25, "to": 6.50 },
        "withdrawnDate": { "from": null, "to": "2024-12-31" }
      }
    }
  ],
  "invalidProducts": [
    {
      "rowNumber": 145,
      "productId": "LP-9999",
      "errors": ["Pricing must be between 0 and 100"]
    }
  ],
  "processingTime": 23456
}
```

**GET /api/products**
- Query params: page, pageSize, search, activeOnly
- Returns paginated product list from LoanProducts

**GET /api/products/:id**
- Returns single product details

**GET /api/products/:id/history**
- Query param: months (default 12)
- Returns pricing change history

### 5. FRONTEND BEHAVIOR (React.js)

Provide guidance for implementing:

**Upload Component**:
- File input accepting only .xlsx
- On file select, POST to /api/upload
- Immediately receive batchId
- Start polling /api/batches/:batchId/status every 2 seconds
- Display progress bar with percentage
- Show real-time statistics (records processed, created, updated)
- When status becomes COMPLETED, fetch reconciliation report
- Display reconciliation summary in modal or dedicated view
- Allow download of reconciliation report as PDF

**Reconciliation Summary View**:
- Summary statistics prominently displayed
- Tabbed interface: "Created Products" | "Updated Products" | "Invalid Products"
- Created Products: table with ProductID, Name, Pricing
- Updated Products: table showing what changed (old → new values)
- Invalid Products: table with row number and error messages
- Action buttons: "Download Report", "View All Products", "Upload New File"

**Product List View**:
- Server-side paginated table
- Search by ProductID or ProductName
- Filter by Active/Inactive status
- Columns: ProductID, ProductName, Pricing, Status badge, Last Updated
- Click row to view details

**Product Detail View**:
- Display all product fields
- Pricing History section with timeline/chart
- Show: Date, Old Pricing → New Pricing, Changed By
- Calculate and display: "X pricing changes in last 12 months"

### 6. ERROR HANDLING & EDGE CASES

Handle these scenarios explicitly:

**Upload Errors**:
- No file uploaded → 400 error
- Wrong file type → 400 error
- File too large (>10MB) → 413 error
- Malformed Excel → Parse error with details
- Empty Excel → 400 error

**Validation Errors**:
- Invalid pricing (negative, >100) → Mark invalid, continue with others
- Missing required fields → Mark invalid
- Invalid dates → Mark invalid
- Allow partial batch success (some valid, some invalid)

**Processing Errors**:
- Database connection lost mid-processing → Chunk rollback, retry
- Chunk processing failure → Log error, mark batch FAILED, preserve completed chunks
- Concurrent uploads → Each batch independent, no conflicts

**Recovery Scenarios**:
- Server restart during processing → Resume from last completed chunk
- Network timeout → Retry chunk (idempotent operations)
- Duplicate batch upload → Detect and reject or mark as duplicate

### 7. MONITORING & OBSERVABILITY

Include:
- Structured logging (Winston or similar)
- Log levels: INFO (upload started), WARN (validation errors), ERROR (processing failures)
- Performance metrics: chunk processing time, total batch duration
- Database query performance logging
- Health check endpoint: GET /health

### 8. CONFIGURATION & DEPLOYMENT

Provide:
- .env.example with all required variables
- Database connection string format
- Chunk size configuration (default 500, adjustable)
- Polling interval configuration
- File upload limits
- Connection pool settings

### 9. TESTING SCENARIOS

Include test data and scenarios:
- Sample Excel with 10 products (mix of new and existing)
- Scenario: Upload with all new products
- Scenario: Upload with all updates (pricing changes)
- Scenario: Upload with mix (new + updates + unchanged)
- Scenario: Upload with validation errors
- Scenario: Large file (2500 records)

### 10. PRODUCTION READINESS CHECKLIST

Ensure the solution includes:
- [ ] SQL Server indexes on all foreign keys and frequently queried columns
- [ ] Database connection pooling configured
- [ ] Transaction isolation levels set appropriately
- [ ] Error handling with meaningful messages
- [ ] Input validation at every layer
- [ ] SQL injection prevention (parameterized queries)
- [ ] File upload security (type, size validation)
- [ ] Authentication middleware
- [ ] Audit logging for all data changes
- [ ] Graceful error responses (no stack traces to client)
- [ ] CORS configuration
- [ ] Request timeout settings
- [ ] Helmet.js security headers
- [ ] Rate limiting considerations
- [ ] Documentation comments in code

## CRITICAL SUCCESS CRITERIA

Your solution will be evaluated on:

1. **Resilience**: Can it recover from mid-processing failures without data loss?
2. **Auditability**: Can we trace every change from Excel upload to production?
3. **Performance**: Does it process 2500 records in under 60 seconds?
4. **User Experience**: Do users get immediate feedback and real-time progress?
5. **Data Integrity**: Are all changes tracked? Is history preserved?
6. **Production Readiness**: Can this be deployed to a bank's production environment today?
7. **Compliance**: Does it meet banking regulatory requirements (no deletes, full audit trail)?
8. **Code Quality**: Is the code maintainable, well-structured, and documented?

## OUTPUT FORMAT

Provide:
1. Complete database schema (CREATE TABLE scripts)
2. All stored procedures with comments
3. Complete Node.js backend code (all files)
4. API documentation with example requests/responses
5. React component structure and key implementation details
6. Sample data and test scenarios
7. Deployment guide
8. Architecture diagrams (described in text/ASCII art)

Assume this will be reviewed by bank architects, security teams, and compliance officers. Every decision should be defensible for a production banking environment.


==========================================================


export interface ApiResponse<T = any> {
  success: boolean;
  message?: string;
  data?: T;
  error?: string;
}

export interface UploadResponse {
  batchId: string;
  totalRecords: number;
  status: BatchStatus;
  message: string;
  statusUrl: string;
}

export interface BatchStatusResponse {
  batchId: string;
  status: BatchStatus;
  fileName: string;
  totalRecords: number;
  validRecords: number;
  invalidRecords: number;
  processedRecords: number;
  progressPercentage: number;
  chunksCompleted: number;
  totalChunks: number;
  statistics: {
    created: number;
    updated: number;
    skipped: number;
  };
  uploadedDate: Date;
  uploadedBy: string;
  processingStarted: Date | null;
  processingCompleted: Date | null;
}

export interface ReconciliationResponse {
  batchId: string;
  status: BatchStatus;
  summary: {
    totalRecords: number;
    created: number;
    updated: number;
    unchanged: number;
    invalid: number;
  };
  createdProducts: Array<{
    productId: string;
    productName: string;
    pricing: number;
  }>;
  updatedProducts: ProductWithChanges[];
  invalidProducts: InvalidProduct[];
  processingTime: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    totalRecords: number;
    totalPages: number;
  };
}

Assume a single-writer system and strict regulatory requirements.
=


================================================================

batch types

export type BatchStatus = 
  | 'UPLOADED' 
  | 'VALIDATING' 
  | 'VALIDATED' 
  | 'PROCESSING' 
  | 'COMPLETED' 
  | 'FAILED';

export type ValidationStatus = 
  | 'PENDING' 
  | 'VALID' 
  | 'INVALID' 
  | 'PROCESSED';

export interface UploadBatch {
  BatchID: string;
  FileName: string;
  TotalRecords: number;
  ValidRecords: number;
  InvalidRecords: number;
  ProcessedRecords: number;
  BatchStatus: BatchStatus;
  UploadedDate: Date;
  UploadedBy: string;
  ProcessingStarted: Date | null;
  ProcessingCompleted: Date | null;
}

export interface StagingProduct {
  StagingID: number;
  BatchID: string;
  RowNumber: number;
  ProductID: string;
  ProductName: string;
  LoanStartDate: Date | string;
  WithdrawnDate: Date | string | null;
  Pricing: number;
  ValidationStatus: ValidationStatus;
  ValidationErrors: string | null;
  ProcessedDate: Date | null;
  UploadedDate: Date;
  UploadedBy: string;
}

export interface ProcessingLog {
  LogID: number;
  BatchID: string;
  ChunkNumber: number;
  RecordsProcessed: number;
  RecordsCreated: number;
  RecordsUpdated: number;
  RecordsSkipped: number;
  ProcessingTime: number;
  LogDate: Date;
}

export interface ChunkResult {
  created: number;
  updated: number;
  skipped: number;
  processingTime: number;
}

export interface InvalidProduct {
  rowNumber: number;
  productId: string;
  errors: string[];
}
===================================================================

export interface LoanProduct {
  ProductID: string;
  ProductName: string;
  LoanStartDate: Date | string;
  WithdrawnDate: Date | string | null;
  Pricing: number;
}

export interface LoanProductDB extends LoanProduct {
  IsActive: boolean;
  CreatedDate: Date;
  CreatedBy: string;
  UpdatedDate: Date | null;
  UpdatedBy: string | null;
}

export interface LoanProductHistory {
  HistoryID: number;
  ProductID: string;
  ProductName: string;
  OldPricing: number | null;
  NewPricing: number;
  OldWithdrawnDate: Date | null;
  NewWithdrawnDate: Date | null;
  ChangeType: 'INSERT' | 'UPDATE';
  ChangeDate: Date;
  ChangedBy: string;
}

export interface ProductChange {
  pricing?: {
    from: number;
    to: number;
  };
  withdrawnDate?: {
    from: Date | null;
    to: Date | null;
  };
}

export interface ProductWithChanges {
  productId: string;
  productName: string;
  changes: ProductChange;
}
