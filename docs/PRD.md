# Mercetia Calculator - Product Requirements Document

## Table of Contents
{toc}

## 1. Project Overview

**Project Name:** Mercetia Calculator  
**Type:** Java/Gradle multi-module project  
**Java Version:** 21 (LTS)  
**Framework:** Spring Boot 3.x  
**Build Tool:** Gradle (Kotlin DSL)

### 1.1 Purpose
Mercetia Calculator: a production-grade Java calculation library and service supporting multiple pay types, US federal/state taxes, various pay frequencies, and full deduction modeling. Deployable as a Gradle-published library, CLI tool (Picocli), and Spring Boot REST API.

**SLA:** Single calculation < 50ms wall-clock (2.5GHz+, 8GB RAM); Batch 10K calculations < 30s; API p95 response < 200ms; Memory < 500MB heap for typical employee payloads (< 200 comp elements).

### 1.2 Scope

**In Scope:**
- **Core Library (mercetia-core):** Pure Java calculation engine (no Spring dependencies) — gross pay, tax, and deduction calculations with BigDecimal precision, Banker's rounding (HALF_EVEN) per IRS Publication 15-T
- **CLI Module (mercetia-cli):** Command-line interface for interactive/bulk calculations with Picocli; supports single calculation, batch from CSV, and tax bracket management
- **API Module (mercetia-api):** Spring Boot REST service with JSON responses, JWT authentication, and OpenAPI/Swagger documentation
- **Persistence (mercetia-persistence):** PostgreSQL with JPA/Hibernate for employee records, tax brackets, calculation history, and Flyway migrations
- **Tax Data (mercetia-tax-data):** Tax bracket seed data, yearly migrations, and state-specific withholding configurations
- **Security (mercetia-security):** JWT authentication, RBAC authorization, PII masking, and secrets management

**Out of Scope:**
- Full benefits administration/enrollment workflow (deduction modeling only)
- City/county local taxes (NYC, Philadelphia, etc.) — tracked as future enhancement
- Multi-state employee workflows — tracked as future enhancement
- Full reporting/forms generation (W-2, 941 equivalents) — tracked as future enhancement
- Webhook/payroll provider sync integrations — tracked as future enhancement
- Multi-currency support
- Real-time payroll processing

**Boundaries:**
- `mercetia-core` remains Spring-free; Spring Boot dependencies confined to `mercetia-api` and `mercetia-cli`
- Deduction modeling covers pre-tax, post-tax, and employer-paid categories as defined in the domain model
- Tax bracket updates via API without code changes, subject to version gating

---

## 2. Functional Requirements

### 2.1 Pay Types Supported

| Pay Type | Description | Given/When/Then |
|----------|-------------|-----------------|
| **Hourly** | Regular hours, overtime (1.5x after 40hrs/week), double time (2x optional) | **Given** an hourly employee with rate r and hours h: **When** calculating gross pay: **Then** gross = regular(40*r) + overtime((h-40)*r*1.5) [doubleTime optional] + doubleTime(0). Overtime kicks in after 40 hours in a Mon-Sun calendar week. Accuracy: within $0.01 of IRS test vectors. |
| **Salaried** | Fixed amount per pay period (annual salary / pay periods) | **Given** a salaried employee with annual salary S and pay frequency F: **When** calculating gross pay per period: **Then** gross = S / periods_per_year(F). Accuracy: within $0.01 of test vectors. |
| **Commission** | Base salary + commission percentage on sales/revenue | **Given** a commission employee with base salary B, commission rate c, and sales S: **When** calculating gross pay: **Then** gross = B + (S * c). Commission base source (gross vs net) configured per employee. |
| **Hybrid** | Combination of above (e.g., hourly + commission) | **Subsection 2.1.1:** Hybrid Pay Type Definition — hybrid_gross = hourly_gross + commission_gross. The 40-hour OT threshold applies only to the hourly component. Example: Employee with $20/hr base (40hrs) + 10% commission on $5K sales: Gross = (40 × $20) + ($5K × 0.10) = $800 + $500 = $1,300. Clarification: "Salaried employees do not automatically qualify for overtime; manual override required if company policy differs." |

### 2.1.1 Hybrid Pay Type Definition *(New)*

- Formula: `hybrid_gross = hourly_gross + commission_gross`
- The 40-hour OT threshold applies only to the hourly component
- Example: Employee with $20/hr base (40hrs) + 10% commission on $5K sales → Gross = $1,300
- Clarification: "Salaried employees do not automatically qualify for overtime; manual override required if company policy differs."

### 2.2 Pay Frequencies

| Frequency | Min Days | Max Days | Validation |
|-----------|----------|----------|------------|
| Weekly | 7 | 7 | Fixed: always 7 days |
| Bi-weekly | 13 | 14 | Fixed: 13-14 days |
| Semi-monthly | 13 | 16 | Two periods/month: 1st–15th, 15th–EOM |
| Monthly | 28 | 31 | Calendar month start–end |
| Custom | 7 | 35 | Employer-defined: start_offset + end_offset (days) |

**Constraint:** Custom cycles >35 days or <7 days require executive approval (compliance check).

*Replace vague "Configurable" with concrete validation rules above.*

### 2.3 Tax Calculations (US Federal + State)

| Calculation Type | Rules | Given/When/Then |
|------------------|-------|-----------------|
| **Federal** | Progressive tax brackets per IRS Publication 15-T, yearly updates | **Given** taxable income T, year Y, and federal brackets for Y: **When** calculating federal tax: **Then** apply progressive brackets (rate on bracket_max - bracket_min), rounding: HALF_EVEN. Accuracy: within $0.01 of IRS test vectors for year Y. |
| **State** | Configurable per state (flat or progressive), stored in database | **Given** taxable income T, state code S, year Y, and state brackets for S,Y: **When** calculating state tax: **Then** apply state-specific progressive or flat rate. If flat: rate * T. If progressive: same logic as federal. Accuracy: within $0.01 of state test vectors. |
| **FICA - Social Security** | 6.2% up to yearly wage base W; excess income exempt | **Given** gross income G, year Y, and SS wage base W for Y: **When** calculating FICA Social Security: **Then** ss_amount = min(G, W) * 0.062. If G > W: ss_amount = W * 0.062. |
| **FICA - Medicare** | 1.45% on all gross; additional 0.9% on income > threshold | **Given** gross income G, year Y, and Medicare thresholds (standard T_std, additional T_addl): **When** calculating Medicare: **Then** standard = G * 0.0145; additional = max(0, G - T_addl) * 0.009; total = standard + additional. |
| **FUTA/SUTA** | Employer-side unemployment taxes, yearly limits | **Given** employee wages G and employer state S, year Y: **When** calculating FUTA/SUTA: **Then** apply state-specific rate up to state wage base. FUTA base rate 6.0% first $7,000, credit for state taxes paid. |

**Yearly Updates:** Tax brackets and FICA limits stored in database (`tax_brackets_federal`, `tax_brackets_state`, `fica_limits`), updatable via API without code changes, subject to version gating.

### 2.3.1 Federal–State Tax Interaction Rules *(New)*

**Standard Deduction:** Federal-only; states have independent brackets. No cross-reduction: Federal standard deduction does NOT reduce CA/NY/TX state taxable income.

**Example (2024 simplified):**
- Gross: $60,000
- Federal standard deduction: $14,600
- Federal taxable: $45,400
- CA taxable: $60,000 (CA has own brackets, ignores federal std ded)
- CA state tax computed on full $60,000 (or CA std ded amount)

**Example configs:**
```yaml
states:
  CA:
    type: progressive
    standard_deduction: 5202  # 2024
    brackets: [...]
  TX:
    type: flat
    rate: 0.0  # No income tax
    standard_deduction: 0
  NY:
    type: progressive
    standard_deduction: 4300  # Varies by filing status
    brackets: [...]
  FL:
    type: flat
    rate: 0.0  # No state income tax
    standard_deduction: 0
```

### 2.3.2 FICA Wage Limits & Medicare Additional Tax *(New)*

**Social Security (6.2% + 6.2% employer):**
- Applied up to yearly wage base (~$168K in 2024)
- Excess income exempt from SS withholding
- Per-employee, per-calendar-year: no reconciliation across employers

**Medicare (1.45% + 1.45% employer + 0.9% additional):**
- Standard 1.45% on all gross income
- Additional 0.9% threshold: $200K (single), $250K (MFJ), $125K (MFS)
- Mercetia requires `tax_filing_status` to determine correct threshold

**Out of Scope (Phase 1):** Multi-employer SS wage reconciliation

### 2.3.3 Tax Bracket Versioning & Updates *(New)*

**Lifecycle:**
- All calculations for year Y use brackets locked as of Jan 1, Y
- Mid-year bracket changes (rare) are NOT supported in Phase 1
- Phase 2 spike: Version-gating and retroactive recalculation policy

**Database Versioning:**
- Add columns to `tax_brackets_federal`/`state`:
  - `fetched_at`: TIMESTAMP (when brackets were loaded)
  - `source`: VARCHAR (DB | API | SEED)
  - `version_id`: UUID (for audit trail)

**Fallback Chain:** If brackets for year Y unavailable:
1. Try year Y
2. Fallback to Y-1, Y-2, ... (earliest available)
3. If none found → 503 Unavailable (not degraded mode with warning)

### 2.4 Deductions & Benefits

| Category | Examples | Federal | State | FICA SS | FICA Medicare | Example |
|----------|----------|---------|-------|---------|---------------|---------|
| **401k** | Pre-tax 401k | ✓ | ✓ | ✓ | ✓ | Reduces all three |
| **HSA** | Health Savings Account | ✓ | ✓ | ✓ | ✓ | Reduces all three |
| **Health Premiums** | Medical/dental/vision | ✓ | ✓ | ✗ | ✗ | Federal/state only (Section 106) |
| **Roth 401k** | Post-tax 401k | ✗ | ✗ | ✗ | ✗ | Post-tax only |

*Replace the vague "Reduces taxable income" column with specific rules per deduction type above.*

### 2.4.1 Pre-tax Deduction Processing Order *(New)*

**Pre-tax deductions applied in this sequence:** 401k → HSA → Health premiums. State can opt out of federal deduction precedence via config.

**Example:** $1000 gross, $500 401k, $100 HSA, $50 health premium:
- Federal taxable = $1000 - $500 - $100 - $50 = $350
- State taxable = $1000 - $500 - $100 - $50 = $350 (depends on state)
- FICA taxable = $1000 - $500 - $100 (health excluded) = $400

*See Section 2.4.1 for pre-tax deduction order.*

### 2.5 YTD Accumulator Strategy *(New Subsection 2.5.1)*

**YTD Accumulator Source:**
- **Option A:** Client passes YTD figures in request (stateless API)
  - Pros: Simpler, no DB dependency
  - Cons: Client must track state
  - Use case: Real-time API calls with external state management
- **Option B:** API fetches from DB (stateful)
  - Pros: Single source of truth
  - Cons: Requires DB query per calculation
  - Use case: Batch processing, internal use

**Mercetia Phase 1:** Implement Option B
- Add endpoint: `GET /api/v1/employees/{id}/ytd?year=2024`
- Returns: `{ ytdGross, ytdFederal, ytdState, ytdFicaSS, ytdFicaMedicare, ... }`

**Validation:**
- YTD totals = SUM(all calculations for employee in calendar year)
- Within $0.01 of aggregated period totals (acceptance criteria)

---

## 3. Non-Functional Requirements

### 3.1 Performance (SLA)
- **Single calculation:** < 50ms wall-clock time (benchmark: 2.5GHz+ CPU, 8GB RAM, JVM 21, default GC)
- **Batch processing:** 10,000 calculations < 30 seconds wall-clock time (sequential, no parallelization)
- **API response time:** < 200ms p95 under normal load (measured at /api/v1/calculate endpoint)
- **Concurrent request target:** 100+ simultaneous calculations without throughput degradation (target: >95% of single-request throughput)
- **Memory boundary:** < 500MB heap usage for typical employee payloads (< 200 compensation elements)
- **Garbage collection pause:** < 100ms max pause time during calculation workloads
- **Startup time:** < 3 seconds for API module; < 1 second for CLI module

### 3.2 Accuracy
- **Decimal precision:** BigDecimal throughout — no floating-point types (float/double) for monetary calculations at any layer
- **Rounding:** Banker's rounding (BigDecimal.ROUND_HALF_EVEN) per IRS Publication 15-T rules
- **Audit trail:** Full calculation steps logged in structured JSON format with correlation ID per request
- **Accuracy tolerance:** Within $0.01 of IRS test vectors for federal tax calculations (all 4 pay types, all 4 pay frequencies)
- **YTD accumulator accuracy:** Within $0.01 of aggregated period totals for same employee across calendar year
- **Tax bracket lookup:** Within $0.01 of stored database values (no floating-point rounding in bracket metadata)

### 3.3 Security
- **Authentication:** JWT with short-lived access tokens (15min TTL) + refresh tokens (30 day TTL)
  - Access token included in `Authorization: Bearer <token>` header
  - Refresh via `POST /api/v1/auth/token` with valid refresh token
  - Token invalidation on logout, password reset, or admin forced reset
- **Authorization:** RBAC with fine-grained permissions matrix
  - **Admin:** `calculate:*`, `view:*`, `manage:tax-brackets`, `manage:users`, `system:config`
  - **Payroll:** `calculate:employee*`, `view:history`, `manage:deductions[ownAssignments]`
  - **Employee:** `view:own`, `update:ownTaxProfile`
- **PII Handling:**
  - SSN encrypted at rest (AES-256) via PostgreSQL column encryption (pgcrypto)
  - No PII in logs — mask SSN (format: ***-**-####), bank accounts, full names in all log output and error responses
  - Input sanitization: validate all API payloads against JSON Schema before processing; reject with 400 and correlation ID
  - PII_READ privilege required for roles that need to view unmasked SSN
- **Secrets Management:**
  - Database credentials, JWT secrets, encryption keys via environment variables or secret manager (HashiCorp Vault/AWS Secrets Manager)
  - No hardcoded secrets in source code or configuration
  - Secret rotation without restart via Spring Cloud Config / Kubernetes Secrets
- **Transport Security:** TLS 1.3+ for all external communications (HTTPS, gRPC over TLS)
- **Session Fixation Protection:** Regenerate session ID after authentication

### 3.4 Compliance
- **IRS Publication 15-T compliance** for federal withholding calculations (yearly brackets, rounding rules)
- **State-specific withholding rules** as configured per state tax table (flat or progressive)
- **ACA affordability calculations** (safe harbor methods: 9.12% of household income, W-2 wages, rate of pay)
- **401k contribution limits enforcement** (IRS annual limits: $23,000 employee elective 2024; $30,500 catch-up age 50+)
- **Data privacy:** GDPR/CCPA-conscious data handling, user export endpoint, right-to-be-forgotten delete
- **Employer identification:** EIN stored encrypted at rest; used only for tax filing forms

### 3.5 Observability
- **Logging:** Structured JSON logs (key-value pairs) with correlation ID per request ID; log correlation across all modules
  - Log format: `{timestamp, level, correlationId, service, endpoint, calculationId, durationMs, message}`
  - No PII in log output (masked as per security policy)
- **Key metrics:** calculation duration (histogram), success/failure counts (counter), error types (counter), active request gauge
- **Tracing:** OpenTelemetry SDK integrated across all modules — end-to-end request tracing with span IDs
- **Error telemetry:** Automatic error reporting with stack trace, user context (role/employee ID, masked), calculation ID, correlation ID
- **Health checks:** `/actuator/health` endpoint with custom components (DB connectivity, tax data validity, tax bracket version)
- **Metrics exposure:** Prometheus-compatible metrics endpoint (`/actuator/prometheus`)
  - Custom metrics: `mercetia.calculations.duration`, `mercetia.calculations.success`, `mercetia.calculations.failure`, `mercetia.active.requests`
- **Audit log:** Immutable calculation history stored in `mercetia_calculations` table; queryable by employee_id, period, calculation_id

### 3.5.1 Correlation ID Propagation Strategy *(New Subsection)*

**Single Calculation:**
- API generates unique correlationId (UUID v4) per request
- Propagated via MDC to all logs
- Returned in response & error messages
- Format: `{ timestamp, level, correlationId, service, endpoint, durationMs, message }`

**Batch Calculation:**
- Parent job: correlationId = jobId (UUID)
- Each child calculation: correlationId = jobId:child-{index}
  - Example: "550e8400-e29b-41d4-a716-446655440000:child-1"
- All logs use full correlationId for tracking
- Results queryable by parent jobId or child correlationId

---

## 4. Architecture & Data Model

### 4.1 Database Schemas (ER Diagram) — Updated

**Entity: employees** — *Updated with status state machine fields*
- `id`: UUID (primary key, generated)
- `first_name`: VARCHAR(100)
- `last_name`: VARCHAR(100)
- `ssn`: CHAR(9) — encrypted (AES-256, database column encryption)
- `hire_date`: DATE
- `status`: VARCHAR(20) — enum: ACTIVE, TERMINATED, ON_LEAVE (state machine)
- `tax_filing_status`: VARCHAR(20) — enum: SINGLE, MFJ, MFS, HOH, DEPENDENT (default: SINGLE)
- `terminated_at`: DATE
- `leave_start_date`: DATE
- `leave_end_date`: DATE
- `version`: BIGINT (optimistic locking)
- `created_at`: TIMESTAMP
- `updated_at`: TIMESTAMP

**Entity: tax_brackets_federal**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `bracket_min`: DECIMAL(15,2) — lower bound of tax bracket (inclusive)
- `bracket_max`: DECIMAL(15,2) — upper bound of tax bracket (exclusive)
- `rate`: DECIMAL(5,4) — marginal tax rate (decimal, e.g., 0.22 for 22%)
- `fetched_at`: TIMESTAMP (when brackets were loaded)
- `source`: VARCHAR (DB | API | SEED)
- `version_id`: UUID (for audit trail)
- `UNIQUE(year, bracket_min)`

**Entity: tax_brackets_state**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `state_code`: CHAR(2) — ISO 3166-2:US state code
- `bracket_min`: DECIMAL(15,2)
- `bracket_max`: DECIMAL(15,2)
- `rate`: DECIMAL(5,4)
- `fetched_at`: TIMESTAMP (when brackets were loaded)
- `source`: VARCHAR (DB | API | SEED)
- `version_id`: UUID (for audit trail)
- `UNIQUE(year, state_code, bracket_min)`

**Entity: fica_limits**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `ss_wage_base`: DECIMAL(15,2) — Social Security wage base
- `medicare_threshold_standard`: DECIMAL(15,2)
- `medicare_threshold_additional`: DECIMAL(15,2)

**Entity: deductions**
- `id`: UUID (primary key, generated)
- `type`: VARCHAR(20) — enum: PRE_TAX, POST_TAX, EMPLOYER_PAID
- `category`: VARCHAR(30) — enum: RETIREMENT, HEALTH, INSURANCE, GARNISHMENT, OTHER
- `amount_type`: VARCHAR(30) — enum: FIXED, PERCENTAGE_OF_GROSS, PERCENTAGE_OF_TAXABLE
- `value`: DECIMAL(15,4) — default deduction amount
- `limits`: JSONB — map: {annual: DECIMAL, per_pay_period: DECIMAL}
- `pretax_for`: VARCHAR(10) — enum: FEDERAL, STATE, FICA, ALL
- `is_active`: BOOLEAN
- `effective_date`: DATE
- `termination_date`: DATE

**Entity: employee_deductions**
- `id`: BIGINT (primary key)
- `employee_id`: UUID (foreign key → employees)
- `deduction_id`: UUID (foreign key → deductions)
- `effective_date`: DATE
- `termination_date`: DATE
- `is_active`: BOOLEAN

**Entity: employee_deduction_snapshots** — *New table for limit tracking*
- `id`: UUID (primary key, generated)
- `pay_period_id`: BIGINT (foreign key → pay_periods)
- `employee_id`: UUID (foreign key → employees)
- `deduction_id`: UUID (foreign key → deductions)
- `value_on_period_start`: DECIMAL(15,4) — Amount as of pay period start
- `amount_applied`: DECIMAL(15,4) — Actual amount deducted this period
- `ytd_total`: DECIMAL(15,4) — YTD cumulative
- `is_limit_exceeded`: BOOLEAN

**Entity: mercetia_calculations** — *Updated with immutability constraints*
- `id`: UUID (primary key, generated)
- `employee_id`: UUID (foreign key → employees)
- `period_start`: DATE
- `period_end`: DATE
- `gross_pay`: DECIMAL(15,2)
- `taxable_income`: DECIMAL(15,2)
- `federal_tax`: DECIMAL(15,2)
- `state_tax`: DECIMAL(15,2)
- `fica_employee_tax`: DECIMAL(15,2)
- `fica_employer_tax`: DECIMAL(15,2)
- `net_pay`: DECIMAL(15,2)
- `created_at`: TIMESTAMP (immutable)
- `updated_at`: TIMESTAMP (immutable, = created_at)
- `deleted_at`: TIMESTAMP DEFAULT NULL (soft delete for compliance)
- `bracket_version_id`: UUID (FOREIGN KEY → tax_brackets_federal.id)
- `correlation_id`: VARCHAR(50) UNIQUE

**Entity: pay_periods**
- `id`: BIGINT (primary key)
- `start_date`: DATE
- `end_date`: DATE
- `pay_frequency`: VARCHAR(20) — enum: WEEKLY, BI_WEEKLY, SEMI_MONTHLY, MONTHLY, CUSTOM
- `status`: VARCHAR(20) — enum: OPEN, CLOSED, LOCKED (state machine)
- `version`: BIGINT (optimistic locking)

**Entity: auth_tokens**
- `id`: BIGINT (primary key)
- `employee_id`: UUID (foreign key → employees)
- `token_hash`: CHAR(64) — SHA-256 hash of JWT
- `issued_at`: TIMESTAMP
- `expires_at`: TIMESTAMP
- `revoked`: BOOLEAN
- `created_at`: TIMESTAMP

**ER Diagram Notes:**
- employees 1:N mercetia_calculations (one employee has many calculations; immutable history)
- employees 1:N employee_deductions (active enrollments)
- employees 1:N pay_periods (pay periods assigned)
- employees 1:N employee_deduction_snapshots (limit tracking)
- tax_brackets_federal 1:N (yearly brackets; historical tracking)
- tax_brackets_state 1:N (state-specific brackets)
- fica_limits 1:N (yearly limits)
- deductions 1:N employee_deductions (enrollments)
- employees status state machine: ACTIVE → TERMINATED/OON_LEAVE; TERMINATED terminal; ON_LEAVE → ACTIVE
- mercetia_calculations immutability: CHECK (created_at = updated_at), CHECK (deleted_at IS NULL or soft delete)

### 4.2 State Machine Transitions — Extended

**Employee Status States:** ACTIVE | TERMINATED | ON_LEAVE

- **ACTIVE:**
  - Can transition to TERMINATED (with terminated_at date)
  - Can transition to ON_LEAVE (with leave_start_date)
- **TERMINATED:**
  - Terminal state; no further calculations allowed
  - Re-hiring creates new employee record (different UUID)
- **ON_LEAVE:**
  - Can transition back to ACTIVE (when leave_end_date passes)
  - No calculations during ON_LEAVE period

**Transition validation:** status change requires effective_date; historical audit logged

### 4.3 API & Protocol Dependencies *(unchanged from original)*

### 4.4 Technical Constraints *(unchanged from original)*

### 4.5 Calculation Request/Response Schemas *(Revised with full JSON Schema)*

**POST /api/v1/calculate**

**REQUEST Schema:**
```json
{
  "employeeId": "uuid",
  "payPeriodStart": "2024-01-01",
  "payPeriodEnd": "2024-01-15",
  "hoursWorked": 80.0,  // REQUIRED for hourly employees
  "grossAmountOverride": 5000.00,  // Optional: override computed gross (e.g., bonus)
  "ytdGrossPay": 10000.00,  // Optional: for FICA wage base limits
  "ytdFicaSocialSecurity": 6200.00,  // Optional: accumulator
  "ytdFicaMedicare": 1450.00,  // Optional: accumulator
  "taxFilingStatus": "SINGLE",  // Optional: overrides employee default
  "includeBreakdown": true,
  "payFrequency": "BI_WEEKLY"  // Optional: override employee default
}
```

**RESPONSE Schema (200):**
```json
{
  "calculationId": "uuid",
  "grossPay": 1600.00,
  "taxableIncome": 1500.00,
  "federalTax": 180.00,
  "stateTax": 75.00,
  "ficaSocialSecurityEmployee": 92.40,
  "ficaMedicareEmployee": 23.20,
  "ficaSocialSecurityEmployer": 92.40,
  "ficaMedicareEmployer": 23.20,
  "netPay": 1129.40,
  "breakdown": {
    "components": [
      { "name": "Gross Pay", "value": 1600.00 },
      { "name": "Pre-tax Deductions", "details": [...] },
      { "name": "Taxable Income", "value": 1500.00 },
      { "name": "Federal Tax", "value": 180.00 },
      { "name": "State Tax", "value": 75.00 },
      { "name": "FICA", "details": {...} },
      { "name": "Post-tax Deductions", "details": [...] },
      { "name": "Net Pay", "value": 1129.40 }
    ],
    "warnings": ["Used fallback tax brackets for year 2023"],
    "degradedMode": false,
    "bracketVersion": "uuid",
    "correlationId": "uuid",
    "calculatedAt": "2024-01-15T10:00:00Z"
  }
}
```

### 4.6 Batch Async Job Polling *(New endpoint)*

**GET /api/v1/calculate/batch/{jobId}**

**RESPONSE (200):**
```json
{
  "jobId": "uuid",
  "status": "PROCESSING",  // QUEUED | PROCESSING | COMPLETED | FAILED
  "submittedCount": 1000,
  "processedCount": 750,
  "progress": 75,
  "results": {
    "successful": 745,
    "failed": 5
  },
  "resultsUrl": "/api/v1/calculate/batch/{jobId}/results",
  "errors": [
    { "employeeId": "uuid", "error": "INVALID_EMPLOYEE_ID" }
  ],
  "estimatedCompletionTime": "2024-01-15T10:05:00Z"
}
```

### 4.7 Deduction Limits & Snapshots *(Updated)*

**`limits` JSONB structure:**
```json
{
  "annual": 23000.00,        // IRS annual cap (401k 2024)
  "per_pay_period": null,    // Optional per-period cap
  "enforcement": "STOP"      // STOP (inactive) | WARN (alert) | EXCEED (allow over)
}
```

**Enforcement:**
- annual: Cumulative YTD amount for calendar year
- per_pay_period: Cap per single pay cycle
- After limit reached (annual): deduction becomes inactive unless enforcement=EXCEED

### 4.8 Auditability & Immutability *(Updated)*

**DB constraints to mercetia_calculations:**
```sql
ALTER TABLE mercetia_calculations 
  ADD CONSTRAINT calc_immutable CHECK (created_at = updated_at),
  ADD CONSTRAINT calc_no_delete CHECK (deleted_at IS NULL),
  ADD COLUMN deleted_at TIMESTAMP DEFAULT NULL,  -- Soft delete for compliance
  ADD COLUMN bracket_version_id UUID FOREIGN KEY,
  ADD COLUMN correlation_id VARCHAR(50) UNIQUE;
```

**Application-level enforcement:** "All reads from mercetia_calculations must use SELECT ... WHERE deleted_at IS NULL."

---

## 5. Edge Cases & Failure Recovery

*(unchanged from original with minor renumbering)*

### 5.1 Network Failures *(renumbered from 4.1)*
### 5.2 Invalid Input Handling *(renumbered from 4.2)*
### 5.3 Retry Logic *(renumbered from 4.3)*
### 5.4 Fallback Modes *(renumbered from 4.4)*
### 5.5 Error Response Format *(renumbered from 4.5)*
### 5.6 Recovery Time Objectives *(renumbered from 4.6)*

---

## 6. CLI Interface

### 6.1 Commands *(unchanged)*
```bash
meretia-calc interactive
meretia-calc calculate --employee-id 123 --pay-period 2024-01-15
meretia-calc batch --input employees.csv --output results.json
meretia-calc tax-brackets import --year 2024 --file brackets.csv
meretia-calc tax-brackets export --year 2024
```

### 6.2 Batch CSV Formats *(New subsection 6.2)*

**Input CSV (--input employees.csv):**
```
employee_id,first_name,last_name,hire_date,status,pay_type,hourly_rate,
annual_salary,commission_rate,pay_frequency,hours_worked,gross_override,
deductions_json
550e8400-e29b-41d4-a716-446655440000,John,Doe,2020-01-15,ACTIVE,
HOURLY,25.00,,0.0,BI_WEEKLY,80.0,,
"{""401k"": {""type"": " "PRE_TAX"", " "amount"": 500.00}}"
```

**Output CSV (--output results.json):**
```
calculation_id,employee_id,period_start,period_end,gross_pay,
federal_tax,state_tax,fica_employee,net_pay,success,error_message
```

---

## 7. Configuration

### 7.1 Application Properties *(unchanged)*
```yaml
mercetia:
  default-pay-frequency: BI_WEEKLY
  rounding-mode: HALF_EVEN
  precision: 4
  tax-year: 2024

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mercetia
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
```

---

## 8. Testing Requirements

### 8.1 Unit Tests (Expanded)

**IRS Test Vectors:** Attach reference spreadsheet (20-30 scenarios from Publication 15-T)

**Overtime edge cases:** 40.5 hrs, 50 hrs, >80 hrs/week

**Deduction limits:** exactly at annual limit, $0.01 over, $0.01 under

**Pay frequencies:** Feb 29 (leap year), month-end edge cases

**Year-over-year regression:** 2020–2024, same employee through each year's brackets

### 8.2 CLI Batch Test *(New subsection 8.2)*

**Input CSV schema:**
```
employee_id, hire_date, pay_type, hourly_rate, salary, commission_rate, 
pay_frequency, hours_worked, gross_override, deductions_json
```

**Output CSV schema:**
```
employee_id, calculation_id, gross_pay, federal_tax, state_tax, fica_employee, net_pay
```

**Acceptance:** 1000 employees < 10s

### 8.3 Compliance Tests *(Expanded)*

- **State coverage (Phase 1):** CA, TX, FL, NY, PA, IL, OH, GA, NC, MI
- **Per-state validation:** 1 test case per state, hand-verified against state tax agency
- **Year-over-year regression:** 2020–2024, same employee through each year's brackets

---

## 9. Delivery Phases

### Phase 1: Core Library (Week 1-2)
- Domain model, calculation engine
- Unit tests with IRS test vectors
- Gradle publication setup

### Phase 2: Persistence & Tax Data (Week 3)
- JPA entities, Flyway migrations
- Tax bracket seed data (2020-2025)
- Repository layer
- **Spike: Tax bracket update workflow with version gating**

### Phase 3: REST API (Week 4)
- Spring Boot API module
- OpenAPI/Swagger docs
- Authentication skeleton (JWT login/refresh)
- **Spike: Correlation ID propagation across modules**

### Phase 4: CLI & Polish (Week 5)
- CLI module with Picocli
- Batch processing
- Documentation, examples

---

## 10. Open Questions

### 10.1 Open Question Status

1. **State coverage:** All 50 states + DC, or subset initially? — *Phase 1: subset (top 10 by employee count), Phase 2: expand*
2. **Local taxes:** City/county taxes (NYC, Philadelphia, etc.) — *Out of Scope (tracked as future enhancement)*
3. **Multi-state employees:** Employees working in multiple states — *Out of Scope (tracked as future enhancement)*
4. **Benefits administration:** Full benefits enrollment workflow or just deduction modeling? — *Deduction modeling only (Out of Scope)*
5. **Reporting:** Quarter/year-end forms (W-2, 941, state equivalents) — *Out of Scope (tracked as future enhancement)*
6. **Integration:** Webhooks for payroll provider sync? — *Out of Scope (tracked as future enhancement)*
7. **Multi-employer FICA SS wage reconciliation** — *Out of Scope (Phase 1 limitation, tracked for Phase 2+)*

**Phase 1 Limitations (Tracked for Phase 2+):**
- Mid-year tax bracket updates not supported (see Section 2.3.3)
- Multi-employer FICA SS wage reconciliation not supported
- City/county local taxes deferred (NYC, Philadelphia, etc.)
- Multi-state employee workflows deferred
- Full W-2/941 form generation deferred
- Webhook/payroll provider sync deferred
- 2024-only IRS brackets; older years available but not auto-updated

### 10.2 Resolution Requirements (What Is Needed to Answer)

Each open question requires specific data or decisions before it can be resolved. The table below captures the information gap, the information needed, and the impact of leaving it unresolved.

| # | Question | Information Needed to Resolve | Impact if Unresolved | Criticality |
|---|----------|-------------------------------|----------------------|-------------|
| 1 | **State coverage** | Employee geographic distribution data; top 10 states by employee count; per-state compliance cost (filing frequency, registration, reporting); expansion metric definitions | Blocks Phase 1 subset selection; ~80% of users affected; delays time-to-market for high-density states | **High (P0)** |
| 2 | **Local taxes** | Local jurisdiction rule database (NYC, Philadelphia, Baltimore, Detroit, Milwaukee); per-locality withholding rules (rates, brackets, thresholds); employer registration requirements; local tax layer architecture decision (separate service vs. embedded rules) | Architectural impact deferred; risk of rework if local tax layer not designed into tax engine abstraction | Medium (P1) |
| 3 | **Multi-state employees** | Work location tracking requirements (primary vs. secondary state); reciprocal tax agreement database; primary-withholding coordination logic; mid-year move / dual-residency edge cases; UI/UX pattern for state selection in requests | Compliance risk for remote/relocating employees; no data model for work_state/prior_states today (Section 4.1) | Medium (P1) |
| 4 | **Benefits administration** | Confirm whether "deduction modeling only" is a hard constraint or Phase 1 boundary; minimum viable benefits functionality; benefits vendor integration map (401k/HSA/COBRA); eligibility rules & data model extensions | Scope ambiguity; possible re-scope churn if enrollment workflow expected by customers | Low (P2) |
| 5 | **Tax reporting forms** | W-2 box specifications; 941 quarterly requirements; state equivalent forms & deadlines; e-file requirements (IRS + state portals); data mapping from calculation model to form fields | Regulatory exposure; customers cannot complete year-end without form generation; may require manual workarounds | Medium (P1) |
| 6 | **Payroll provider webhooks** | Target provider APIs (ADP, Gusto, Paychex, QuickBooks); webhook security (HMAC, token verification, retry/idempotency policies); data mapping to provider schemas; partner agreements & sandbox access | Market-dependent; low current impact but blocks provider sync integrations | Low (P2) |
| 7 | **Multi-employer FICA SS reconciliation** | IRS Form 943 procedures (per-employer vs. aggregate); per-employee SS wage base tracking across employers per calendar year; credit/offset logic for excess withholding; cross-employer aggregation data model | Wage base accuracy risk; employees with multiple jobs may be over-withheld with no reconciliation path | **High (P0)** |

### 10.3 Prioritization & Recommended Decision Sequence

**Phase A (Immediate — Phase 1 boundary clarification):**
1. **Q1 State coverage** — Confirm the 10 Phase 1 states and the expansion metric (employee count vs. revenue vs. industry)
2. **Q7 Multi-employer FICA SS reconciliation** — Design the Phase 2 approach now; document per-employee YTD SS wage tracking requirements (Section 2.3.2 already flags this as out of scope for Phase 1)
3. **Q5 Tax reporting forms** — Define minimum viable forms for Phase 1 vs. Phase 2; document W-2/941 data mapping requirements

**Phase B (Medium term — Phase 1 deferred items):**
4. **Q2 Local taxes** — Decide: defer entirely or design the local tax abstraction layer into the tax engine now to avoid rework
5. **Q3 Multi-state employees** — Document work-location tracking boundaries and reciprocal agreement handling for Phase 2

**Phase C (Ongoing — scope clarification):**
6. **Q4 Benefits administration** — Confirm "deduction modeling only" is a product decision, not a temporary limitation
7. **Q6 Payroll provider webhooks** — Assess market demand and target provider partnership requirements

**Cross-Cutting Note:** All Phase 1 decisions must be reviewed before Phase 2 spikes begin (tax bracket versioning, Section 2.3.3) to avoid rework in the tax engine abstraction.

---

## 11. Appendices

### Appendix A: Tax Bracket Examples

**Federal 2024 Progressive Brackets (Single):**
```yaml
single:
  - bracket_min: 0
    bracket_max: 11600
    rate: 0.10
  - bracket_min: 11600
    bracket_max: 47150
    rate: 0.12
  - bracket_min: 47150
    bracket_max: 100525
    rate: 0.22
  - bracket_min: 100525
    bracket_max: 191950
    rate: 0.24
  - bracket_min: 191950
    bracket_max: 243725
    rate: 0.32
  - bracket_min: 243725
    bracket_max: 609350
    rate: 0.35
  - bracket_min: 609350
    bracket_max: none
    rate: 0.37
```

**CA 2024 Progressive Brackets (Single):**
```yaml
type: progressive
standard_deduction: 5202
brackets:
  - bracket_min: 0
    bracket_max: 10471
    rate: 0.01
  - bracket_min: 10471
    bracket_max: 24688
    rate: 0.02
  # ... continued
```

**TX 2024:**
```yaml
type: flat
rate: 0.00  # No state income tax
standard_deduction: 0
```

### Appendix B: Deduction Disallow Matrix by State

| State | Allows Fed Std Ded | 401k Reduction | HSA Reduction | Notes |
|-------|-------------------|----------------|---------------|-------|
| CA | ✓ | ✓ | ✓ | Standard |
| TX | N/A | ✓ | ✓ | No state income tax |
| NY | ✓ | ✓ | ✓ | Standard |
| IL | ✗ | ✗ | ✗ | Flat tax; ignores deductions |
| FL | N/A | ✓ | ✓ | No state income tax |

### Appendix C: Correlation ID Format

**Single Calculation:**
- Format: UUID v4
- Example: `550e8400-e29b-41d4-a716-446655440000`

**Batch Calculation:**
- Parent job: correlationId = jobId (UUID)
- Child calculation: correlationId = jobId:child-{index}
  - Example: `550e8400-e29b-41d4-a716-446655440000:child-1`

**Log Format:**
```
{timestamp: 2024-01-15T10:00:00Z, level: INFO, correlationId: 550e8400-e29b-41d4-a716-446655440000, service: calc-api, endpoint: POST /api/v1/calculate, durationMs: 34, message: "Calculation completed successfully"}
```

---

## 12. Decision Log

- **ADR-001:** Core library Spring-free positioning — justify `mercetia-core` dependency scope
- **ADR-002:** API versioning strategy — v1 as default, backwards-compatibility approach
- **ADR-003:** Database migration branching model — Flyway per-environment vs. single baseline
- **ADR-004:** CQRS vs. simple CRUD for calculation history — event sourcing light vs. immutable log
- **ADR-005:** JWT vs. session-based auth — trade-offs for stateless vs. server-side session store
- **ADR-006:** Encryption-at-rest for SSN — database column encryption vs. application-level encryption
- **ADR-007:** OpenAPI-first vs. code-first approach — Springdoc vs. Swagger annotations
- **ADR-008:** Pay frequency validation — concrete day ranges vs. flexible config
- **ADR-009:** Pre-tax deduction processing order — sequential application for federal/state/FICA
- **ADR-010:** YTD accumulator strategy — Option B (DB-sourced) for Phase 1
- **ADR-011:** Correlation ID propagation — UUID v4 per request, jobId:child-{index} for batch