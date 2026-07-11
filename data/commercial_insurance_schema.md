# Commercial Insurance SQL Training Schema

Expected dataframe/query result schema is inferred from each SQL `select` list. All generated examples target PostgreSQL and this relational mock schema.

## Tables
### `carriers`
- `carrier_id bigserial primary key`
- `carrier_name text not null`
- `naic_code text not null unique`
- `am_best_rating text not null`
- `domicile_state char(2) not null`
- `active_flag boolean not null default true`

### `regions`
- `region_id bigserial primary key`
- `region_name text not null unique`
- `country_code char(2) not null default 'US'`

### `branches`
- `branch_id bigserial primary key`
- `region_id bigint not null references regions(region_id)`
- `branch_name text not null`
- `state_code char(2) not null`
- `opened_date date not null`

### `teams`
- `team_id bigserial primary key`
- `branch_id bigint not null references branches(branch_id)`
- `team_name text not null`
- `specialty text not null`

### `employees`
- `employee_id bigserial primary key`
- `team_id bigint not null references teams(team_id)`
- `employee_name text not null`
- `role_title text not null`
- `hire_date date not null`
- `active_flag boolean not null default true`

### `brokers`
- `broker_id bigserial primary key`
- `broker_name text not null`
- `broker_tier text not null`
- `appointed_date date not null`
- `active_flag boolean not null default true`

### `broker_offices`
- `broker_office_id bigserial primary key`
- `broker_id bigint not null references brokers(broker_id)`
- `office_name text not null`
- `state_code char(2) not null`
- `opened_date date not null`

### `producers`
- `producer_id bigserial primary key`
- `broker_office_id bigint not null references broker_offices(broker_office_id)`
- `producer_name text not null`
- `license_state char(2) not null`
- `active_flag boolean not null default true`

### `accounts`
- `account_id bigserial primary key`
- `account_name text not null`
- `industry_code text not null`
- `annual_revenue numeric(18,2) not null check (annual_revenue >= 0)`
- `employee_count integer not null check (employee_count >= 0)`
- `headquarter_state char(2) not null`
- `created_date date not null`

### `account_locations`
- `location_id bigserial primary key`
- `account_id bigint not null references accounts(account_id)`
- `state_code char(2) not null`
- `county_name text not null`
- `latitude numeric(9,6)`
- `longitude numeric(9,6)`
- `location_open_date date not null`

### `insured_assets`
- `asset_id bigserial primary key`
- `location_id bigint not null references account_locations(location_id)`
- `asset_type text not null`
- `asset_value numeric(18,2) not null check (asset_value >= 0)`
- `construction_year integer`
- `protection_class integer`

### `policies`
- `policy_id bigserial primary key`
- `account_id bigint not null references accounts(account_id)`
- `carrier_id bigint not null references carriers(carrier_id)`
- `producer_id bigint not null references producers(producer_id)`
- `underwriter_id bigint not null references employees(employee_id)`
- `policy_number text not null unique`
- `line_of_business text not null`
- `policy_status text not null`
- `effective_date date not null`
- `expiration_date date not null`
- `written_premium numeric(18,2) not null check (written_premium >= 0)`
- `commission_rate numeric(7,6) not null check (commission_rate >= 0)`
- `retention_ratio numeric(7,6) not null check (retention_ratio between 0 and 1)`
- `risk_score numeric(9,4) not null check (risk_score >= 0)`

### `policy_transactions`
- `policy_transaction_id bigserial primary key`
- `policy_id bigint not null references policies(policy_id)`
- `transaction_date date not null`
- `transaction_type text not null`
- `premium_change numeric(18,2) not null`
- `limit_change numeric(18,2) not null default 0`

### `coverages`
- `coverage_id bigserial primary key`
- `policy_id bigint not null references policies(policy_id)`
- `coverage_type text not null`
- `coverage_limit numeric(18,2) not null check (coverage_limit >= 0)`
- `deductible numeric(18,2) not null check (deductible >= 0)`
- `exposure_basis text not null`

### `exposure_snapshots`
- `exposure_snapshot_id bigserial primary key`
- `coverage_id bigint not null references coverages(coverage_id)`
- `snapshot_month date not null`
- `exposure_amount numeric(18,4) not null check (exposure_amount >= 0)`
- `earned_premium numeric(18,2) not null check (earned_premium >= 0)`
- `earned_exposure_units numeric(18,4) not null check (earned_exposure_units >= 0)`

### `premium_transactions`
- `premium_transaction_id bigserial primary key`
- `policy_id bigint not null references policies(policy_id)`
- `transaction_date date not null`
- `accounting_month date not null`
- `written_premium_delta numeric(18,2) not null`
- `earned_premium_delta numeric(18,2) not null`
- `fees numeric(18,2) not null default 0`
- `taxes numeric(18,2) not null default 0`

### `claims`
- `claim_id bigserial primary key`
- `policy_id bigint not null references policies(policy_id)`
- `coverage_id bigint references coverages(coverage_id)`
- `claim_number text not null unique`
- `loss_date date not null`
- `reported_date date not null`
- `closed_date date`
- `claim_status text not null`
- `cause_of_loss text not null`
- `severity_score numeric(9,4) not null check (severity_score >= 0)`
- `initial_case_reserve numeric(18,2) not null check (initial_case_reserve >= 0)`

### `claim_reserve_snapshots`
- `reserve_snapshot_id bigserial primary key`
- `claim_id bigint not null references claims(claim_id)`
- `snapshot_month date not null`
- `case_reserve numeric(18,2) not null check (case_reserve >= 0)`
- `ibnr_reserve numeric(18,2) not null check (ibnr_reserve >= 0)`
- `allocated_loss_adjustment_expense numeric(18,2) not null check (allocated_loss_adjustment_expense >= 0)`

### `claim_payments`
- `claim_payment_id bigserial primary key`
- `claim_id bigint not null references claims(claim_id)`
- `payment_date date not null`
- `payment_month date not null`
- `payment_type text not null`
- `paid_amount numeric(18,2) not null check (paid_amount >= 0)`
- `recovery_amount numeric(18,2) not null default 0 check (recovery_amount >= 0)`

### `claim_notes`
- `claim_note_id bigserial primary key`
- `claim_id bigint not null references claims(claim_id)`
- `employee_id bigint not null references employees(employee_id)`
- `note_date date not null`
- `note_category text not null`
- `follow_up_required boolean not null default false`

### `risk_assessments`
- `risk_assessment_id bigserial primary key`
- `policy_id bigint not null references policies(policy_id)`
- `assessment_date date not null`
- `assessment_type text not null`
- `risk_score numeric(9,4) not null check (risk_score >= 0)`
- `loss_control_score numeric(9,4) not null check (loss_control_score >= 0)`
- `recommended_rate_change numeric(9,6) not null`

### `reinsurance_contracts`
- `reinsurance_contract_id bigserial primary key`
- `carrier_id bigint not null references carriers(carrier_id)`
- `contract_name text not null`
- `treaty_type text not null`
- `effective_date date not null`
- `expiration_date date not null`

### `reinsurance_layers`
- `reinsurance_layer_id bigserial primary key`
- `reinsurance_contract_id bigint not null references reinsurance_contracts(reinsurance_contract_id)`
- `attachment_point numeric(18,2) not null check (attachment_point >= 0)`
- `occurrence_limit numeric(18,2) not null check (occurrence_limit >= 0)`
- `ceded_share numeric(7,6) not null check (ceded_share between 0 and 1)`

### `claim_reinsurance`
- `claim_reinsurance_id bigserial primary key`
- `claim_id bigint not null references claims(claim_id)`
- `reinsurance_layer_id bigint not null references reinsurance_layers(reinsurance_layer_id)`
- `ceded_paid_amount numeric(18,2) not null check (ceded_paid_amount >= 0)`
- `ceded_reserve_amount numeric(18,2) not null check (ceded_reserve_amount >= 0)`
- `valuation_month date not null`

### `calendar_dates`
- `calendar_date date primary key`
- `calendar_year integer not null`
- `calendar_quarter integer not null check (calendar_quarter between 1 and 4)`
- `calendar_month integer not null check (calendar_month between 1 and 12)`
- `month_start date not null`
- `quarter_start date not null`
- `year_start date not null`
- `is_month_end boolean not null`

## Analytics Patterns
- Weighted averages use `sum(metric * weight) / nullif(sum(weight), 0)`.
- Moving averages use PostgreSQL window frames such as `rows between 2 preceding and current row`.
- Time series queries use `date_trunc('month', date_column)::date` and bounded date predicates.
- Resilient ratios use `nullif(denominator, 0)` and nullable aggregates use `coalesce` where appropriate.
