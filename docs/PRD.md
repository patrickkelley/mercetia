# Mercetia Calculator - Product Requirements Document

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
| **Hourly** | Regular hours, overtime (1.5x after 40hrs/week), double time (2x) | **Given** an hourly employee with rate r and hours h > 40: **When** calculating gross pay: **Then** gross = regular(40*r) + overtime((h-40)*r*1.5) + doubleTime(0). Accuracy: within $0.01 of IRS test vectors. |
| **Salaried** | Fixed amount per pay period (annual salary / pay periods) | **Given** a salaried employee with annual salary S and pay frequency F: **When** calculating gross pay per period: **Then** gross = S / periods_per_year(F). Accuracy: within $0.01 of test vectors. |
| **Commission** | Base salary + commission percentage on sales/revenue | **Given** a commission employee with base salary B, commission rate c, and sales S: **When** calculating gross pay: **Then** gross = B + (S * c). Commission base source (gross vs net) configured per employee. |
| **Hybrid** | Combination of above (e.g., hourly + commission) | **Given** a hybrid employee with hourly component and commission component: **When** calculating gross pay: **Then** gross = hourly_gross + commission_gross - overlapping deductions. End-to-end accuracy within $0.01. |

### 2.2 Pay Frequencies

| Frequency | Periods/Year | Date Range Generation | Given/When/Then |
|-----------|-------------|----------------------|-----------------|
| **Weekly** | 52 | Start = Monday of week, End = Sunday; or custom start/end | **Given** a weekly pay period with start_date S: **When** generating period range: **Then** end_date = S + 6 days, and next_period_start = end_date + 1 day. |
| **Bi-weekly** | 26 | Start = Monday of bi-week, End = Sunday of bi-week | **Given** a bi-weekly pay period with start_date S: **When** generating period range: **Then** end_date = S + 13 days, and next_period_start = end_date + 1 day. |
| **Semi-monthly** | 24 | 1st and 15th of month | **Given** a semi-monthly pay period: **When** generating period range for month M: **Then** periods are {1st of M, 15th of M}. Edge: if 1st or 15th falls on weekend/holiday, roll to next business day. |
| **Monthly** | 12 | 1st to last day of month | **Given** a monthly pay period with start_date S (1st of month): **When** generating period range: **Then** end_date = last day of month(M). |
| **Custom** | Configurable | Defined by employer | **Given** a custom pay period definition with start_offset and end_offset: **When** generating period range: **Then** end_date = start_date + end_offset days, and next_period_start = end_date + 1 day. |

**Configurable:** Custom pay period definitions stored in database, updatable without code changes via API.

### 2.3 Tax Calculations (US Federal + State)

| Calculation Type | Rules | Given/When/Then |
|------------------|-------|-----------------|
| **Federal** | Progressive tax brackets per IRS Publication 15-T, yearly updates | **Given** taxable income T, year Y, and federal brackets for Y: **When** calculating federal tax: **Then** apply progressive brackets (rate on bracket_max - bracket_min), rounding: HALF_EVEN. Accuracy: within $0.01 of IRS test vectors for year Y. |
| **State** | Configurable per state (flat or progressive), stored in database | **Given** taxable income T, state code S, year Y, and state brackets for S,Y: **When** calculating state tax: **Then** apply state-specific progressive or flat rate. If flat: rate * T. If progressive: same logic as federal. Accuracy: within $0.01 of state test vectors. |
| **FICA - Social Security** | 6.2% up to yearly wage base W; excess income exempt | **Given** gross income G, year Y, and SS wage base W for Y: **When** calculating FICA Social Security: **Then** ss_amount = min(G, W) * 0.062. If G > W: ss_amount = W * 0.062. |
| **FICA - Medicare** | 1.45% on all gross; additional 0.9% on income > threshold T_addl | **Given** gross income G, year Y, and Medicare thresholds (standard T_std, additional T_addl): **When** calculating Medicare: **Then** standard = G * 0.0145; additional = max(0, G - T_addl) * 0.009; total = standard + additional. |
| **FUTA/SUTA** | Employer-side unemployment taxes, yearly limits | **Given** employee wages G and employer state S, year Y: **When** calculating FUTA/SUTA: **Then** apply state-specific rate up to state wage base. FUTA base rate 6.0% first $7,000, credit for state taxes paid. |

**Yearly Updates:** Tax brackets and FICA limits stored in database (`tax_brackets_federal`, `tax_brackets_state`, `fica_limits`), updatable via API without code changes, subject to version gating.

### 2.4 Deductions & Benefits

| Category | Examples | Tax Treatment | Given/When/Then |
|----------|----------|---------------|-----------------|
| **Pre-tax** | 401k, HSA, FSA, Health/Dental/Vision premiums | Reduces taxable income | **Given** gross pay G, pre-tax deduction D with amount A: **When** calculating taxable income: **Then** taxable = G - A (capped by annual limits per IRS). Reduces federal/state/FICA taxable income. |
| **Post-tax** | Roth 401k, Garnishments, Union dues, Life insurance | After-tax | **Given** taxable income T, post-tax deduction D with amount A: **When** calculating net pay: **Then** net = T - A (after all tax calculations). Does not reduce taxable income. |
| **Employer-paid** | Company 401k match, Employer health premiums, HSA contributions | Not employee deduction | **Given** employee compensation C, employer deduction E with amount A: **When** calculating total compensation cost: **Then** employer_cost = C + A. Employee gross pay unchanged. |
| **Custom** | Configurable deduction types with rules | Configurable | **Given** a custom deduction with type T, amount A, and rules R: **When** applying deduction: **Then** behavior follows rules R (amount_type, limits, pretaxFor). Validation: amount must satisfy category constraints. |

**Deduction Limits:** Annual and per-mercetia limits enforced per deduction. Limits stored in `limits` map (annual, per_merkeria). Enforcement: deduction becomes inactive after limit reached (unless overridden).

### 2.5 Calculation Outputs

| Output | Description | Given/When/Then |
|--------|-------------|-----------------|
| **Gross pay** | Regular + overtime + commission + bonuses | **Given** employee with pay type P, hours/sales, and deductions D: **When** calculating: **Then** gross = compute_gross(P, ...) - per pay type rules. |
| **Taxable income** | Gross pay minus pre-tax deductions | **Given** gross pay G and pre-tax deductions D: **When** calculating taxable income: **Then** taxable = G - sum(pretaxFor=FEDERAL|STATE|FICA deductions). |
| **Federal tax** | Progressive tax on taxable income | **Given** taxable income T, year Y, federal brackets: **When** calculating: **Then** return computed federal tax (HALF_EVEN rounding). |
| **State tax** | Progressive or flat tax on taxable income | **Given** taxable income T, state S, year Y, state brackets: **When** calculating: **Then** return computed state tax. |
| **FICA employee tax** | Social Security + Medicare | **Given** taxable income T: **When** calculating: **Then** return ss_amount + med_amount per FICA rules. |
| **FICA employer tax** | Employer Social Security + Medicare | **Given** employee wages G: **When** calculating: **Then** return employer_ss + employer_med per FICA rules. |
| **Net pay** | Gross - all taxes - post-tax deductions | **Given** all above calculations: **When** calculating net pay: **Then** net = gross - federal_tax - state_tax - fica_employee_tax - post_tax_deductions. |
| **Breakdown** | Per-component detail | **Given** full calculation result: **When** requesting breakdown: **Then** return structured JSON with all components: grossPay, taxableIncome, federalTax, stateTax, ficaEmployeeTax,ficaEmployerTax, netPay, and per-component notes. |
| **YTD accumulators** | Year-to-date totals | **Given** employee with calculations across N periods: **When** computing YTD: **Then** return sum of all gross, tax, and deduction amounts for current calendar year. |

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

---

## 4. Architecture & Data Model

### 4.1 Database Schemas (ER Diagram)

**Entity: employees**
- `id`: UUID (primary key, generated)
- `first_name`: VARCHAR(100)
- `last_name`: VARCHAR(100)
- `ssn`: CHAR(9) — encrypted (AES-256, database column encryption)
- `hire_date`: DATE
- `status`: VARCHAR(20) — enum: ACTIVE, TERMINATED, ON_LEAVE (state machine)
- `version`: BIGINT (optimistic locking)
- `created_at`: TIMESTAMP
- `updated_at`: TIMESTAMP

**Entity: tax_brackets_federal**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `bracket_min`: DECIMAL(15,2) — lower bound of tax bracket (inclusive)
- `bracket_max`: DECIMAL(15,2) — upper bound of tax bracket (exclusive)
- `rate`: DECIMAL(5,4) — marginal tax rate (decimal, e.g., 0.22 for 22%)
- `UNIQUE(year, bracket_min)`

**Entity: tax_brackets_state**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `state_code`: CHAR(2) — ISO 3166-2:US state code
- `bracket_min`: DECIMAL(15,2)
- `bracket_max`: DECIMAL(15,2)
- `rate`: DECIMAL(5,4)
- `UNIQUE(year, state_code, bracket_min)`

**Entity: fica_limits**
- `id`: BIGINT (primary key)
- `year`: SMALLINT
- `ss_wage_base`: DECIMAL(15,2) — Social Security wage base
- `medicare_threshold_standard`: DECIMAL(15,2)
- `medicare_threshold_additional`: DECIMAL(15,2)
- `UNIQUE(year)`

**Entity: deductions**
- `id`: UUID (primary key, generated)
- `type`: VARCHAR(20) — enum: PRE_TAX, POST_TAX, EMPLOYER_PAID
- `category`: VARCHAR(30) — enum: RETIREMENT, HEALTH, INSURANCE, GARNISHMENT, OTHER
- `amount_type`: VARCHAR(30) — enum: FIXED, PERCENTAGE_OF_GROSS, PERCENTAGE_OF_TAXABLE
- `value`: DECIMAL(15,4) — default deduction amount
- `limits`: JSONB — map: {annual: DECIMAL, per_mercetia: DECIMAL}
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

**Entity: mercetia_calculations**
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

**Entity: pay_periods**
- `id`: BIGINT (primary key)
- `start_date`: DATE
- `end_date`: DATE
- `pay_frequency`: VARCHAR(20) — enum: WEEKLY, BI_WEEKLY, SEMI_MONTHLY, MONTHLY
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
- tax_brackets_federal 1:N (yearly brackets; historical tracking)
- tax_brackets_state 1:N (state-specific brackets)
- fica_limits 1:N (yearly limits)
- deductions 1:N employee_deductions (enrollments)
- pay_periods status state machine: OPEN → CLOSED → LOCKED

### 4.2 State Machine Transitions

**Employee Status States:** ACTIVE | TERMINATED | ON_LEAVE
- ACTIVE → TERMINATED: on termination date entered
- ACTIVE → ON_LEAVE: on leave of absence start
- TERMINATED → (terminal, no further calculations)
- ON_LEAVE → ACTIVE: on return from leave
- **Transition validation:** status change requires effective_date; historical audit logged

**Pay Period Status States:** OPEN | CLOSED | LOCKED
- OPEN: pay period is active; calculations can be submitted
- CLOSED: pay period is closed; no new calculations accepted; existing results read-only
- LOCKED: pay period is locked for audit/compliance; no modifications allowed
- **Transitions:** OPEN → CLOSED (after pay period end date); CLOSED → LOCKED (admin action); LOCKED → CLOSED (admin revert)

### 4.3 API & Protocol Dependencies

| Dependency | Direction | Protocol | Details |
|------------|-----------|----------|---------|
| **mercetia-core → mercetia-persistence** | Outbound | Java API | JPA Repository calls; no Spring dependencies in-core |
| **mercetia-api → mercetia-core** | Inbound | Java API | Calculation service façade; DTO mapping |
| **mercetia-api → tax-brackets service** | Outbound | HTTP REST | `GET /api/v1/tax-brackets/federal/{year}` and `GET /api/v1/tax-brackets/state/{state}/{year}`; fallback to seeded data if unavailable |
| **mercetia-api → authentication service** | Outbound | OAuth2/JWT | Token validation at API gateway; introspection endpoint |
| **CLI → mercetia-core** | Outbound | Java API | Calculation engine invoked programmatically |
| **CLI → PostgreSQL** | Outbound | JDBC | Direct database access for tax data, employee records |
| **API Actuator → Prometheus** | Outbound | HTTP | `/actuator/prometheus` endpoint; metrics exposition |
| **API Actuator → Health** | Outbound | HTTP | `/actuator/health` endpoint; custom component health checks |
| **External payroll provider (future)** | Outbound | REST/Webhook | Plugged via integration module; not in-scope Phase 1-4 |

### 4.4 Technical Constraints

- **Java Version:** 21 (LTS) — `java.lang.ModuleSystem` encapsulation; reflection usage limited
- **Gradle:** Kotlin DSL; multi-module publication to local Maven repository
- **Database:** PostgreSQL 15+; JSONB for flexible schema (limits map, ER notes); pgcrypto for column encryption
- **JVM:** OpenJDK 21; default GC (G1GC); heap max 500MB for API module
- **Spring Boot:** 3.x; WebFlux not used (reactive not in-scope); MVC for REST API
- **Flyway:** Database migrations; baseline-on-migrate; per-environment tagging
- **OpenAPI:** 3.1+ specification; SpringDoc-OpenGenerator for auto-generation
- **JWT:** HS256 signatory algorithm; short-lived access (15min), refresh (30 days)
- **Correlation ID:** Propagated across all modules via MDC (Mapped Diagnostic Context); unique per API request

---

## 5. Edge Cases & Failure Recovery

### 4.1 Network Failures

- **Tax bracket service unavailable:** If `GET /api/v1/tax-brackets/federal/{year}` or `GET /api/v1/tax-brackets/state/{state}/{year}` returns 5xx or times out, fallback to last-known-good tax brackets from database seed data (`mercetia-tax-data` module). Calculation proceeds with stale brackets and logs warning with correlation ID.
- **Database connectivity loss:** If PostgreSQL connection is lost during calculation, transaction is rolled back and `IllegalStateException` thrown with message "Database unavailable; calculation not persisted". Retry logic: 3 attempts with exponential backoff (500ms, 1s, 2s) before final failure.
- **API gateway timeout:** If upstream calculation service does not respond within 5 seconds, return 504 Gateway Timeout with `{ error: "UPSTREAM_TIMEOUT", message, correlationId, retryAfter }`.

### 4.2 Invalid Input Handling

- **Malformed employee ID:** If `employeeId` is not a valid UUID, return 400 `{ error: "VALIDATION_ERROR", message: "Invalid employee ID format", correlationId, fields: {employeeId: "Must be a valid UUID"} }`.
- **Invalid pay period date:** If `payPeriod` is before `LocalDate.MIN` or after current date + 1 year, return 400 with descriptive message.
- **Missing required fields:** If any required request field is missing, return 400 with `fields` map indicating which field(s) are missing and why.
- **Schema validation failure:** If API payload does not match JSON Schema, return 422 `{ error: "VALIDATION_FAILED", message, correlationId, fields: {...} }` with specific validation errors.
- **Unsupported pay type:** If `payType` is not one of HOURLY/SALARIED/COMMISSION/HYBRID, return 400.

### 4.3 Retry Logic

- **Transient calculation failures:** For calculation timeouts or concurrent modification exceptions, retry up to 3 times with 200ms initial delay, doubling each attempt.
- **FICA limit lookups:** If `fica_limits` table row not found for requested year, fallback to most recent available year with logged warning.
- **Tax bracket lookups:** If bracket row not found for requested year/state, fallback to prior year's brackets with logged warning.

### 4.4 Fallback Modes

- **Degraded mode:** If tax data validity check fails (section 3.5), API continues accepting calculation requests but marks calculations as `UNVERIFIED` and returns warning in breakdown. Critical tax calculations (federal) blocked until data restored.
- **Read-only mode:** If database connectivity is lost, API accepts read requests for existing calculation history only; new calculations rejected with 503.
- **Cache fallback:** If BigDecimal performance degrades beyond SLA, optional `StrictMath`-based fallback for edge-case operations (verified against test vectors).

### 4.5 Error Response Format (All Failure Modes)

```
HTTP/{code}: { 
  error: ERROR_TYPE, 
  message: human-readable description, 
  correlationId: UUID, 
  timestamp: ISO-8601, 
  requestId: UUID (for support tracking), 
  fields: {field: "error message"} (if validation), 
  retryAfter: seconds (if 429 or 503)
}
```

### 4.6 Recovery Time Objectives (RTO)

- **Calculation timeout:** 5 seconds before 504 response
- **Database reconnection:** 30 seconds maximum before read-only mode
- **Tax data restoration:** 15 minutes before degraded mode alerts escalate
- **Full system recovery:** 1 hour (includes manual tax bracket re-deployment if needed)

### 4.1 Module Structure
```
mercetia-calculator/
├ mercetia-core/          # Pure Java calculation library (Spring-free)
│   └─ Calculations module — no Spring dependencies
├ mercetia-persistence/   # JPA entities, repositories, Flyway migrations
├ mercetia-api/           # Spring Boot REST API + OpenAPI/Swagger
├ mercetia-cli/           # Command-line interface (Picocli)
├ mercetia-tax-data/      # Tax bracket seed data, migrations
├ mercetia-security/      # Auth (JWT, OAuth2), AuthZ model, PII masking
└ mercetia-integration-tests/
```

### 4.2 Core Domain Model
```
Employee
  - id: UUID (generated)
  - firstName, lastName
  - ssn: Encrypted (AES-256, database column encryption)
  - hireDate: LocalDate
  - status: ACTIVE | TERMINATED | ON_LEAVE
  - payType: HOURLY | SALARIED | COMMISSION | HYBRID
  - payFrequency: WEEKLY | BI_WEEKLY | SEMI_MONTHLY | MONTHLY
  - compensation: CompensationDetails
  - taxProfile: TaxProfile
  - deductions: List<Deduction>
  - address: Address (for state tax jurisdiction)
  - version: Long (optimistic locking)

CompensationDetails (interface)
  - hourlyRate: BigDecimal (nullable for salaried/commission)
  - annualSalary: BigDecimal (nullable for hourly)
  - commissionRate: BigDecimal (nullable for hourly/salaried)
  - overtimeEligible: Boolean
  - doubleTimeEligible: Boolean
  - commissionBase: BigDecimal (gross sales, net revenue depending on pay type)

TaxProfile
  - federalFilingStatus: SINGLE | MARRIED_JOINT | MARRIED_SEPARATE | HEAD_OF_HOUSEHOLD
  - federalAllowances: BigDecimal
  - additionalWithholding: BigDecimal
  - stateFilingStatus: String (state-specific code)
  - stateAllowances: BigDecimal
  - exemptFromFICA: Boolean
  - exemptFromFUTA: Boolean
  - stateTaxIds: List<String> (state-specific withholding IDs)

Deduction
  - id: UUID
  - type: PRE_TAX | POST_TAX | EMPLOYER_PAID
  - category: RETIREMENT | HEALTH | INSURANCE | GARNISHMENT | OTHER
  - amountType: FIXED | PERCENTAGE_OF_GROSS | PERCENTAGE_OF_TAXABLE
  - value: BigDecimal
  - limits: Map<String, BigDecimal> (annual, per-mercetia)
  - pretaxFor: FEDERAL | STATE | FICA | ALL
  - isActive: Boolean
  - effectiveDate, terminationDate: LocalDate
```

### 4.3 Database Schema (Key Tables + ER Notes)
- `employees` - Employee records with encrypted SSN, version column
- `tax_brackets_federal` - year, bracket_min, bracket_max, rate (decimal)
- `tax_brackets_state` - year, state_code, bracket_min, bracket_max, rate
- `fica_limits` - year, ss_wage_base, medicare_threshold_standard, medicare_threshold_additional
- `deductions` - Deduction definitions (id, type, category, amount_type, value, limits)
- `employee_deductions` - employee_id, deduction_id, effective_date, termination_date, is_active
- `mercetia_calculations` - calculation_id (UUID), employee_id, period_start, period_end, gross_pay, net_pay, federal_tax, state_tax, fica_employee, fica_employer, created_at (immutable)
- `pay_periods` - period_id, start_date, end_date, pay_frequency, status (OPEN | CLOSED | LOCKED)
- `auth_tokens` - token_hash, employee_id, issued_at, expires_at, revoked (for JWT session tracking)
- **ER Diagram:** TBD — create ERD in Phase 2 spike

### 4.4 API Endpoints (REST) + Request/Response Schemas
```
POST   /api/v1/calculate           # Single calculation
  Request: { employeeId: UUID, payPeriod: LocalDate, includeBreakdown: Boolean }
  Response: 200 { calculationId, grossPay, taxableIncome, federalTax, stateTax, ficaEmployeeTax, ficaEmployerTax, netPay, breakdown: BreakdownDTO }

POST   /api/v1/calculate/batch     # Bulk calculations
  Request: { employeeIds: List<UUID>, payPeriod: LocalDate }
  Response: 202 { jobId, status, submittedCount }

GET    /api/v1/employees/{id}/mercetias  # History
  Response: 200 { calculations: List<CalculationSummary> }

GET    /api/v1/tax-brackets/federal/{year}
  Response: 200 { year, brackets: List<BracketDTO> }

GET    /api/v1/tax-brackets/state/{state}/{year}
  Response: 200 { year, stateCode, brackets: List<BracketDTO> }

POST   /api/v1/tax-brackets        # Admin: update brackets
  Request: { year, stateCode?, brackets: List<BracketDTO> }
  Response: 200 { updatedCount }

GET    /api/v1/auth/me             # Current user profile
POST   /api/v1/auth/token          # JWT token refresh

**Error Response Formats:**
- 400: { error: "VALIDATION_ERROR", message, correlationId, timestamp, fields: {field: "error message"} }
- 401: { error: "AUTHENTICATION_FAILED", message, correlationId }
- 403: { error: "AUTHORIZATION_FAILED", message, correlationId }
- 404: { error: "RESOURCE_NOT_FOUND", message, correlationId, resourceId }
- 422: { error: "VALIDATION_FAILED", message, correlationId, fields: {...} }
- 500: { error: "INTERNAL_ERROR", message, correlationId, requestId (for support tracking) }
```

### 4.5 Authentication & Authorization Model
- **Auth Flow:** OAuth2 Resource Password Credentials + JWT access tokens
  1. POST /api/v1/auth/login {username, password} → 200 {accessToken, refreshToken, expiresIn}
  2. Access token (15min TTL) included in Authorization: Bearer <token> header
  3. Refresh token (30 day TTL) used when access token expires → POST /api/v1/auth/token {refreshToken} → 200 {accessToken, refreshToken}
  4. Tokens revoked on logout, password reset, or admin forced reset

- **AuthZ Model (RBAC):**
  | Role | Permissions |
  |------|-------------|
  | Admin | calculate:*, view:*, manage:tax-brackets, manage:users, system:config |
  | Payroll | calculate:employee*, view:history, manage:deductions[ownAssignments] |
  | Employee | view:own, update:ownTaxProfile |

- **PII Masking Middleware:** Interceptor that masks SSN, bank accounts, full names in all log output and error responses (unless role has PII_READ privilege)

### 4.6 Required Architectural Decision Records (ADRs) / Technical Spikes
- ADR-001: Core library Spring-free positioning — justify `mercetia-core` dependency scope
- ADR-002: API versioning strategy — v1 as default, backwards-compatibility approach
- ADR-003: Database migration branching model — Flyway per-environment vs. single baseline
- ADR-004: CQRS vs. simple CRUD for calculation history — event sourcing light vs. immutable log
- ADR-005: JWT vs. session-based auth — trade-offs for stateless vs. server-side session store
- ADR-006: Encryption-at-rest for SSN — database column encryption vs. application-level encryption
- ADR-007: OpenAPI-first vs. code-first approach — Springdoc vs. Swagger annotations
- **Technical Spike 1:** Prototype tax bracket update workflow with version gating (1 day)
- **Technical Spike 2:** Validate BigDecimal performance at 10K calculations with Banker's rounding (2 days)
- **Technical Spike 3:** Design correlation ID propagation across modules (1 day)

---

## 6. CLI Interface

### 5.1 Commands
```bash
# Interactive mode
mercetia-calc interactive

# Single calculation
mercetia-calc calculate --employee-id 123 --pay-period 2024-01-15

# Batch from CSV
mercetia-calc batch --input employees.csv --output results.json

# Tax bracket management
mercetia-calc tax-brackets import --year 2024 --file brackets.csv
mercetia-calc tax-brackets export --year 2024
```

---

## 7. Configuration

### 6.1 Application Properties
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

### 7.1 Unit Tests (Core)
- Tax calculation accuracy vs IRS test cases
- Overtime/double-time edge cases
- Deduction limit enforcement
- Pay frequency date calculations

### 7.2 Integration Tests
- Full calculation pipeline
- Database persistence
- API contract tests
- CSV import/export

### 7.3 Compliance Tests
- IRS Publication 15-T test scenarios
- State-specific test cases (CA, NY, TX, FL, etc.)
- Year-over-year regression tests

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

1. **State coverage:** All 50 states + DC, or subset initially? — *Phase 1: subset (top 10 by employee count), Phase 2: expand*
2. **Local taxes:** City/county taxes (NYC, Philadelphia, etc.) — *Out of Scope (tracked as future enhancement)*
3. **Multi-state employees:** Employees working in multiple states — *Out of Scope (tracked as future enhancement)*
4. **Benefits administration:** Full benefits enrollment workflow or just deduction modeling? — *Deduction modeling only (Out of Scope)*
5. **Reporting:** Quarter/year-end forms (W-2, 941, state equivalents) — *Out of Scope (tracked as future enhancement)*
6. **Integration:** Webhooks for payroll provider sync? — *Out of Scope (tracked as future enhancement)*

---

## 11. Acceptance Criteria

- [ ] Core library calculates federal tax within $0.01 of IRS test cases
- [ ] All 4 pay types produce correct gross pay
- [ ] All 4 pay frequencies generate correct pay periods
- [ ] Pre-tax deductions reduce taxable income correctly
- [ ] Employer taxes calculated separately from employee
- [ ] API returns JSON with full breakdown
- [ ] CLI processes 1000 employees from CSV in < 10s
- [ ] Tax brackets updatable via API without restart
- [ ] Calculation history persisted and queryable
- [ ] API authentication (JWT) validated with login/logout/refresh flow
- [ ] PII masking active in all log output and error responses
- [ ] Structured JSON logging with correlation ID per request