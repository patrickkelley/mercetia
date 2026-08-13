# Paycheck Calculator - Product Requirements Document

## 1. Project Overview

**Project Name:** Paycheck Calculator  
**Type:** Java/Gradle multi-module project  
**Java Version:** 21 (LTS)  
**Framework:** Spring Boot 3.x  
**Build Tool:** Gradle (Kotlin DSL)

### 1.1 Purpose
A comprehensive paycheck calculation library and service supporting multiple pay types, US federal/state taxes, various pay frequencies, and full deduction/benefit modeling. Deployable as a library, CLI tool, and REST API.

### 1.2 Scope
- **Core Library:** Pure Java calculation engine (no Spring dependencies)
- **CLI Module:** Command-line interface for interactive/bulk calculations
- **API Module:** Spring Boot REST service with JSON responses
- **Persistence:** PostgreSQL with JPA/Hibernate for employee records, tax brackets, calculation history

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
- Single calculation < 50ms
- Batch processing: 10,000 calculations < 30 seconds
- API response time < 200ms (p95)

### 3.2 Accuracy
- Decimal precision: BigDecimal (no floating-point errors)
- Rounding: Banker's rounding (HALF_EVEN) per IRS rules
- Audit trail: Full calculation steps logged

### 3.3 Security
- No PII in logs (mask SSN, bank accounts)
- API authentication (JWT/OAuth2)
- Role-based access (Admin, Payroll, Employee)

### 3.4 Compliance
- IRS Publication 15-T compliance
- State-specific withholding rules
- ACA affordability calculations
- 401k contribution limits enforcement

---

## 4. Technical Architecture

### 4.1 Module Structure
```
paycheck-calculator/
├── paycheck-core/          # Pure Java calculation library
├── paycheck-persistence/   # JPA entities, repositories
├── paycheck-api/           # Spring Boot REST API
├── paycheck-cli/           # Command-line interface
├── paycheck-tax-data/      # Tax bracket seed data, migrations
└── paycheck-integration-tests/
```

### 4.2 Core Domain Model
```
Employee
  - id, firstName, lastName, ssn (encrypted), hireDate, status
  - payType: HOURLY | SALARIED | COMMISSION | HYBRID
  - payFrequency: WEEKLY | BI_WEEKLY | SEMI_MONTHLY | MONTHLY
  - compensation: CompensationDetails
  - taxProfile: TaxProfile (filing status, allowances, additional withholding)
  - deductions: List<Deduction>
  - address: Address (for state tax)

CompensationDetails
  - hourlyRate / annualSalary / commissionRate
  - overtimeEligible, doubleTimeEligible
  - commissionBase (gross sales, net revenue, etc.)

TaxProfile
  - federalFilingStatus: SINGLE | MARRIED_JOINT | MARRIED_SEPARATE | HEAD_OF_HOUSEHOLD
  - federalAllowances / additionalWithholding
  - stateFilingStatus, stateAllowances
  - exemptFromFICA, exemptFromFUTA

Deduction
  - id, type: PRE_TAX | POST_TAX | EMPLOYER_PAID
  - category: RETIREMENT | HEALTH | INSURANCE | GARNISHMENT | OTHER
  - amountType: FIXED | PERCENTAGE_OF_GROSS | PERCENTAGE_OF_TAXABLE
  - value, limits (annual, per-paycheck)
  - pretaxFor: FEDERAL | STATE | FICA | ALL
```

### 4.3 Database Schema (Key Tables)
- `employees` - Employee records
- `tax_brackets_federal` - Yearly federal brackets
- `tax_brackets_state` - Yearly state brackets
- `fica_limits` - Yearly Social Security wage base, Medicare thresholds
- `deductions` - Deduction definitions
- `employee_deductions` - Employee-specific deduction enrollments
- `paycheck_calculations` - Calculation history (immutable)
- `pay_periods` - Pay period definitions

### 4.4 API Endpoints (REST)
```
POST   /api/v1/calculate           # Single calculation
POST   /api/v1/calculate/batch     # Bulk calculations
GET    /api/v1/employees/{id}/paychecks  # History
GET    /api/v1/tax-brackets/federal/{year}
GET    /api/v1/tax-brackets/state/{state}/{year}
POST   /api/v1/tax-brackets        # Admin: update brackets
```

---

## 5. CLI Interface

### 5.1 Commands
```bash
# Interactive mode
paycheck-calc interactive

# Single calculation
paycheck-calc calculate --employee-id 123 --pay-period 2024-01-15

# Batch from CSV
paycheck-calc batch --input employees.csv --output results.json

# Tax bracket management
paycheck-calc tax-brackets import --year 2024 --file brackets.csv
paycheck-calc tax-brackets export --year 2024
```

---

## 6. Configuration

### 6.1 Application Properties
```yaml
paycheck:
  default-pay-frequency: BI_WEEKLY
  rounding-mode: HALF_EVEN
  precision: 4
  tax-year: 2024

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/paycheck
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

### Phase 3: REST API (Week 4)
- Spring Boot API module
- OpenAPI/Swagger docs
- Authentication skeleton

### Phase 4: CLI & Polish (Week 5)
- CLI module with Picocli
- Batch processing
- Documentation, examples

---

## 9. Open Questions

1. **State coverage:** All 50 states + DC, or subset initially?
2. **Local taxes:** City/county taxes (NYC, Philadelphia, etc.)?
3. **Multi-state employees:** Employees working in multiple states?
4. **Benefits administration:** Full benefits enrollment workflow or just deduction modeling?
5. **Reporting:** Quarter/year-end forms (W-2, 941, state equivalents)?
6. **Integration:** Webhooks for payroll provider sync?

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
