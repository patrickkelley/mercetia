# Mercetia Calculator - Product Requirements Document

## 1. Project Overview

**Project Name:** Mercetia Calculator  
**Type:** Java/Gradle multi-module project  
**Java Version:** 21 (LTS)  
**Framework:** Spring Boot 3.x  
**Build Tool:** Gradle (Kotlin DSL)

### 1.1 Purpose
A comprehensive mercetia calculation library and service supporting multiple pay types, US federal/state taxes, various pay frequencies, and full deduction/benefit modeling. Deployable as a library, CLI tool, and REST API.

### 1.2 Scope
**In Scope:**
- **Core Library:** Pure Java calculation engine (no Spring dependencies) — gross pay, tax, deduction calculations only
- **CLI Module:** Command-line interface for interactive/bulk calculations with Picocli
- **API Module:** Spring Boot REST service with JSON responses and authentication
- **Persistence:** PostgreSQL with JPA/Hibernate for employee records, tax brackets, calculation history

**Out of Scope:**
- Full benefits administration/enrollment workflow (deduction modeling only)
- City/county local taxes (NYC, Philadelphia, etc.) — tracked as future enhancement
- Multi-state employee workflows — tracked as future enhancement
- Full reporting/forms generation (W-2, 941 equivalents) — tracked as future enhancement
- Webhook/payroll provider sync integrations — tracked as future enhancement
- Multi-currency support
- Real-time payroll processing

### 1.3 Clarifications
- "Comprehensive" refers to coverage of all US federal pay types, tax brackets, and standard deductions per IRS Publication 15-T
- "Full deduction/benefit modeling" covers pre-tax, post-tax, and employer-paid deduction categories as defined in the domain model
- Module separation: `mercetia-core` remains Spring-free; Spring Boot dependencies confined to `mercetia-api` and `mercetia-cli`

---

## 2. Functional Requirements

### 2.1 Pay Types Supported
| Pay Type | Description |
|----------|-------------|
| **Hourly** | Regular hours, overtime (1.5x after 40hrs/week), double time (2x) |
| **Salaried** | Fixed amount per pay period (annual salary / pay periods) |
| **Commission** | Base salary + commission percentage on sales/revenue |
| **Hybrid** | Combination of above (e.g., hourly + commission) |

### 2.2 Pay Frequencies
- Weekly (52 periods/year)
- Bi-weekly (26 periods/year)
- Semi-monthly (24 periods/year - 1st & 15th)
- Monthly (12 periods/year)
- **Configurable:** Custom pay period definitions

### 2.3 Tax Calculations (US Federal + State)
- **Federal:** Progressive tax brackets (configurable per year)
- **State:** Configurable per state (flat or progressive)
- **FICA:** Social Security (6.2% up to wage base), Medicare (1.45% + 0.9% additional)
- **FUTA/SUTA:** Employer-side unemployment taxes
- **Yearly Updates:** Tax brackets stored in database, updatable without code changes

### 2.4 Deductions & Benefits
| Category | Examples | Tax Treatment |
|----------|----------|---------------|
| **Pre-tax** | 401k, HSA, FSA, Health/Dental/Vision premiums | Reduces taxable income |
| **Post-tax** | Roth 401k, Garnishments, Union dues, Life insurance | After-tax |
| **Employer-paid** | Company 401k match, Employer health premiums, HSA contributions | Not employee deduction |
| **Custom** | Configurable deduction types with rules | Configurable |

### 2.5 Calculation Outputs
- Gross pay (regular + overtime + commission + bonuses)
- Taxable income (after pre-tax deductions)
- Federal/State/FICA tax amounts
- Net pay
- Employer tax liability
- Detailed breakdown per pay component
- Year-to-date accumulators

---

## 3. Non-Functional Requirements

### 3.1 Performance
- Single calculation < 50ms wall-clock time (benchmark hardware: 2.5 GHz+, 8GB RAM)
- Batch processing: 10,000 calculations < 30 seconds wall-clock time
- API response time < 200ms (p95) under normal load
- Concurrent request target: 100+ simultaneous calculations without degradation
- Memory boundary: < 500MB heap usage for typical employee payloads (< 200 comp elements)

### 3.2 Accuracy
- Decimal precision: BigDecimal throughout — no floating-point types for monetary calculations
- Rounding: Banker's rounding (HALF_EVEN) per IRS Publication 15-T rules
- Audit trail: Full calculation steps logged in structured JSON format with correlation ID
- Accuracy tolerance: Within $0.01 of IRS test vectors for federal tax calculations

### 3.3 Security
- **Authentication:** JWT with short-lived access tokens (15min) + refresh tokens (30 days)
- **Authorization:** RBAC with fine-grained permissions matrix (see AuthZ model below)
  - Admin: full CRUD + tax bracket management
  - Payroll: calculate, view history, manage deductions for assigned employees
  - Employee: view own calculations, update personal tax profile
- **PII Handling:** 
  - SSN encrypted at rest (AES-256) via database column encryption
  - No PII in logs — mask SSN, bank accounts, full names in all log output
  - Input sanitization: validate all API payloads against schema before processing
- **Secrets Management:** Database credentials, JWT secrets via environment variables or secret manager (Vault/Secrets Manager)
- **Transport Security:** TLS 1.3+ for all external communications

### 3.4 Compliance
- IRS Publication 15-T compliance for federal withholding calculations
- State-specific withholding rules as configured per state tax table
- ACA affordability calculations (safe harbor methods)
- 401k contribution limits enforcement (IRS annual limits)
- Data privacy: GDPR/CCPA-conscious data handling, user export/right-to-be-forgotten endpoints

### 3.5 Observability
- **Logging:** Structured JSON logs (key-value pairs) with correlation ID per request
- **Key metrics:** calculation duration, success/failure counts, error types, active request gauge
- **Tracing:** OpenTelemetry SDK integrated across all modules — end-to-end request tracing
- **Error telemetry:** Automatic error reporting with stack trace, user context, and calculation ID
- **Health checks:** /actuator/health endpoint with custom components (DB connectivity, tax data validity)
- **Metrics exposure:** Prometheus-compatible metrics endpoint (/actuator/prometheus)

---

## 4. Technical Architecture

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

## 5. CLI Interface

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

## 6. Configuration

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

## 7. Testing Requirements

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

## 8. Delivery Phases

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

## 9. Open Questions

1. **State coverage:** All 50 states + DC, or subset initially? — *Phase 1: subset (top 10 by employee count), Phase 2: expand*
2. **Local taxes:** City/county taxes (NYC, Philadelphia, etc.) — *Out of Scope (tracked as future enhancement)*
3. **Multi-state employees:** Employees working in multiple states — *Out of Scope (tracked as future enhancement)*
4. **Benefits administration:** Full benefits enrollment workflow or just deduction modeling? — *Deduction modeling only (Out of Scope)*
5. **Reporting:** Quarter/year-end forms (W-2, 941, state equivalents) — *Out of Scope (tracked as future enhancement)*
6. **Integration:** Webhooks for payroll provider sync? — *Out of Scope (tracked as future enhancement)*

---

## 10. Acceptance Criteria

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