# Business Rules Catalog — Legacy HRMS (Oracle Forms / PL/SQL)

Source analysed: `ts-plsql-oracle-forms-legacy-codebase` @ `1c33787` (6 Forms XML exports, 2 PLL libraries, 1 menu module, 12 PL/SQL packages, 2 trigger files, 4 table DDL files, views, sequences, seed data).

Every rule cites the legacy file it comes from. Paths are relative to the legacy repo root. Rule IDs are stable so that test-harness scenarios and Java code can reference them (e.g. `// BR-LV-007`).

**Columns**

- **Disposition**: `PRESERVE` (port as-is), `FIX` (known defect; the target should behave differently, and the change must be signed off), `DECIDE` (business owner must confirm the intended behaviour before porting).
- Error codes are the legacy `RAISE_APPLICATION_ERROR` numbers. The Java target should map them to stable API error codes so equivalence tests can assert on them.

---

## 0. Scope corrections vs. existing plan documents

The existing [assessment report](assessment-report.md) and the legacy `README.md` describe a larger system than the source that is actually in the repo:

| Item | Documented | Found in source |
|---|---|---|
| Forms modules | 18 | 6 XML exports: `HRMS_EMPLOYEE`, `HRMS_LEAVE`, `HRMS_LOGIN`, `HRMS_MENU`, `HRMS_PAYROLL`, `HRMS_PERFORMANCE` |
| Forms referenced but missing | — | `HRMS_REPORTS`, `HRMS_ADMIN` (opened from menu), `HRMS_DEPARTMENT`, LOV/toolbar modules |
| PL/SQL packages | 12 | 11 with bodies (`PKG_AUDIT`, `PKG_COMMON`, `PKG_EMPLOYEE`, `PKG_INTEGRATION`, `PKG_LEAVE`, `PKG_NOTIFICATION`, `PKG_PAYROLL`, `PKG_PERFORMANCE`, `PKG_REPORTING`, `PKG_SECURITY`, `PKG_VALIDATION`). No `PKG_DEPARTMENT`. |
| Tables | 42 | 30 |
| Triggers | 200+ | 6 DB triggers (3 in `trg_employees.sql`, 3 in `trg_audit.sql`) + 27 Forms triggers |
| Scheduler jobs | DBMS_SCHEDULER jobs | No job definitions; only comments say "scheduled via DBMS_SCHEDULER" |
| `USER_CREDENTIALS` table | referenced in `PKG_SECURITY` comment | not present |

**Evidence request:** before migrating Reports, Admin and Department maintenance, obtain the missing `.fmb`/XML exports, the `DBMS_SCHEDULER` job definitions (`DBA_SCHEDULER_JOBS`), and the production DDL (to confirm the trigger/schema mismatch in BR-EMP-031).

---

## 1. Security & Authentication (`PKG_SECURITY`, `HRMS_LOGIN`, `HRMS_COMMON_LIB`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-SEC-001 | Login requires both username and password to be non-empty; otherwise "Please enter username and password." | `HRMS_LOGIN.xml` BTN_LOGIN | PRESERVE |
| BR-SEC-002 | Username is the employee e-mail, matched case-insensitively, and only `EMPLOYMENT_STATUS = 'ACTIVE'` employees can log in. | `PKG_SECURITY.authenticate` | PRESERVE |
| BR-SEC-003 | Unknown user raises `-20301 Invalid username or password`; the form shows the same generic message for every failure. | `PKG_SECURITY.authenticate`, `HRMS_LOGIN.xml` | PRESERVE (generic message) |
| BR-SEC-004 | If several active employees share the e-mail, the lowest `EMP_ID` is used. | `PKG_SECURITY.authenticate` | DECIDE (target should enforce unique e-mail instead) |
| BR-SEC-005 | **The password is never verified.** `authenticate` looks up the user and creates a session without comparing a hash. | `PKG_SECURITY.authenticate` | FIX (critical; BCrypt per ADR-003) |
| BR-SEC-006 | Passwords are hashed with MD5 (`hash_password`). | `PKG_SECURITY.hash_password` | FIX (TD-001) |
| BR-SEC-007 | No failed-login counter or lockout. | `PKG_SECURITY.authenticate` | FIX (TD-002) |
| BR-SEC-008 | A successful login inserts a `USER_SESSIONS` row (status `ACTIVE`, IP, login time) and sets the session context. | `PKG_SECURITY.authenticate` | PRESERVE (as JWT + audit event) |
| BR-SEC-009 | Session is valid for 30 minutes from **login time** (absolute, not idle timeout). | `PKG_SECURITY.is_session_valid` | DECIDE (absolute vs sliding) |
| BR-SEC-010 | Every form checks session validity on open (`WHEN-NEW-FORM-INSTANCE`) and refuses to open if expired. | all `HRMS_*.xml`, `HRMS_COMMON_LIB.check_session` | PRESERVE |
| BR-SEC-011 | Logout ends the session row. | `PKG_SECURITY.logout`, `HRMS_MENU.xml` BTN_LOGOUT | PRESERVE |
| BR-SEC-012 | Permission model: job grade id ≥ 8 → all actions on all modules; grade id ≥ 5 → `VIEW` on all modules; everyone → `LEAVE` CREATE/VIEW and `EMPLOYEE` VIEW; anything else denied. | `PKG_SECURITY.has_permission` | PRESERVE, then map to roles |
| BR-SEC-013 | The comparison uses `GRADE_ID` (a surrogate key), not a grade level column. Permissions therefore depend on id values in `JOB_GRADES` seed data. | `PKG_SECURITY.has_permission` | DECIDE |
| BR-SEC-014 | Menu disables Payroll/Admin/Reports items without `VIEW`; Payroll and Reports buttons re-check and show "Access denied." | `HRMS_MENU.xml` | PRESERVE |
| BR-SEC-015 | Employee form makes the EMPLOYEE block read-only without `EMPLOYEE/EDIT`. | `HRMS_EMPLOYEE.xml` WHEN-NEW-FORM-INSTANCE | PRESERVE |
| BR-SEC-016 | Payroll form requires `PAYROLL/VIEW`; approve requires `PAYROLL/APPROVE`. | `HRMS_PAYROLL.xml` | PRESERVE |
| BR-SEC-017 | New password: ≥ 8 chars (`-20310`), ≥ 1 uppercase (`-20311`), ≥ 1 digit (`-20312`). Old password is not checked; no credential is stored. | `PKG_SECURITY.change_password` | PRESERVE rules, FIX stub |
| BR-SEC-018 | SSNs are encrypted with AES-256/CBC/PKCS5 using a hard-coded key in the package body. | `PKG_SECURITY.encrypt_ssn/decrypt_ssn` | FIX (TD-003; key to secret store). Data migration must decrypt with the legacy key. |
| BR-SEC-019 | SSN display mask `***-**-NNNN`. | `PKG_COMMON.mask_ssn` | PRESERVE |

## 2. Employee (`PKG_EMPLOYEE`, `HRMS_EMPLOYEE`, `trg_employees.sql`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-EMP-001 | First and last name required (`-20010`). | `PKG_EMPLOYEE.create_employee` | PRESERVE |
| BR-EMP-002 | Department must exist and be active (`-20003`). | `validate_department` | PRESERVE |
| BR-EMP-003 | Manager, if given, must be an active employee (`-20004`); when an employee id is known the manager chain is walked (max depth 15) to reject circular reporting. On create no id exists, so the cycle check is skipped. | `validate_manager` | PRESERVE |
| BR-EMP-004 | Job title must exist and be active (`-20011`). | `create_employee` | PRESERVE |
| BR-EMP-005 | Base salary outside the job grade's `MIN_SALARY..MAX_SALARY` only logs a warning; it does not block. | `create_employee` | DECIDE |
| BR-EMP-006 | Location defaults to the department's location when not supplied. | `create_employee` | PRESERVE |
| BR-EMP-007 | Names stored `UPPER(TRIM())`, e-mail stored `LOWER(TRIM())`. | `create_employee`, `update_employee` | PRESERVE |
| BR-EMP-008 | Employee number format `EMP-NNNNNN`, generated as `MAX(existing)+1`, falling back to `SEQ_EMPLOYEE`. | `generate_emp_number` | FIX (TD-010, race) — keep the format, use a sequence |
| BR-EMP-009 | New employee gets status `ACTIVE`, `ACTIVE_FLAG='Y'`, an initial `SALARY_RECORDS` row with reason `NEW_HIRE`, a `HIRE` history row, an audit row, a welcome e-mail and a manager e-mail. | `create_employee` | PRESERVE |
| BR-EMP-010 | Duplicate key on insert is reported as `-20002`. | `create_employee` | PRESERVE |
| BR-EMP-011 | History rows are written in an autonomous transaction, so they survive a rollback of the business change. | `log_history` | FIX (history must be in the same transaction) |
| BR-EMP-012 | Update is partial: a `NULL` argument keeps the current value. It is therefore impossible to clear a field through the API. Unknown employee → `-20001`. | `update_employee` | PRESERVE (PATCH semantics) |
| BR-EMP-013 | Search filters (last/first name prefix, dept, status, location) are concatenated into dynamic SQL; ordered by last, first name. | `search_employees` | FIX (TD-004); keep filters and order |
| BR-EMP-014 | Transfer locks the row `NOWAIT` (lock conflict → "Record is locked by another user"), requires status `ACTIVE` (`-20012`), validates the new department and manager, keeps job/location/manager when not supplied, writes a `TRANSFER` history row. The change is applied immediately even if the effective date is in the future. | `transfer_employee`, `HRMS_EMPLOYEE.xml` ON-ERROR | PRESERVE; DECIDE on future-dated transfers |
| BR-EMP-015 | Promotion changes job and creates a new salary record (reason `PROMOTION`, change % rounded to 2 dp). No status check and no grade-range check. | `promote_employee` | DECIDE |
| BR-EMP-016 | Promotion reads the "current" salary with `ROWNUM = 1` before `ORDER BY`, so it can pick an arbitrary salary row. | `promote_employee` | FIX |
| BR-EMP-017 | Termination: locks row; already `TERMINATED` → `-20005`; cancels all `PENDING` leave requests; sets `TERMINATED` and `ACTIVE_FLAG='N'`; end-dates the active salary and pay elements at the termination date; history + audit; notifies the manager. | `terminate_employee` | PRESERVE |
| BR-EMP-018 | Termination cancels pending leave **without releasing the `PENDING` balance**. | `terminate_employee` vs `PKG_LEAVE.cancel_leave_request` | FIX |
| BR-EMP-019 | Termination TODOs (COBRA/benefits, access revocation, final pay) are not implemented. | `terminate_employee` | DECIDE (out of scope or new feature) |
| BR-EMP-020 | Rehire sets status `ACTIVE`, new department/job, overwrites `HIRE_DATE`, creates a salary record and history row. | `rehire_employee` | DECIDE (keep original hire date?) |
| BR-EMP-021 | DB trigger blocks any `TERMINATED → ACTIVE` update (`-20503`). Because `rehire_employee` does exactly that update, **rehire always fails as written**. | `trg_employees.sql`, `rehire_employee` | DECIDE (evidence needed from production) |
| BR-EMP-022 | Hire date ≤ today + 180 days (DB trigger, `-20501`) but ≤ today + 90 days in the form. The two limits differ. | `trg_employees.sql`, `HRMS_EMPLOYEE.xml` WHEN-VALIDATE-ITEM | DECIDE (one value) |
| BR-EMP-023 | E-mail must be unique among active employees (`-20502`). | `trg_employees.sql` | PRESERVE (unique index) |
| BR-EMP-024 | Physical delete of employees is forbidden (`-20504`); termination is the only exit. The form block still allows delete and the user gets the trigger error. | `trg_employees.sql`, `HRMS_EMPLOYEE.xml` | PRESERVE; hide delete in UI |
| BR-EMP-025 | Insert defaults: `ACTIVE_FLAG='Y'`, `EMPLOYMENT_STATUS='ACTIVE'`, audit columns. | `trg_employees.sql`, form PRE-INSERT | PRESERVE |
| BR-EMP-026 | The Employee form inserts/updates `EMPLOYEES` directly (base-table block, `DMLDataTargetName="HRMS.EMPLOYEES"`), **bypassing `create_employee`**: no salary record, no history, no notifications when hiring through the form. | `HRMS_EMPLOYEE.xml` EMPLOYEE block | DECIDE (target should use one path: the service) |
| BR-EMP-027 | Form validates e-mail with `PKG_VALIDATION.validate_email_format`; department/job must be active ("Invalid department", "Invalid job title"). | `HRMS_EMPLOYEE.xml` WHEN-VALIDATE-ITEM | PRESERVE |
| BR-EMP-028 | Form default query shows only `ACTIVE` + `ACTIVE_FLAG='Y'` employees. | `HRMS_EMPLOYEE.xml` | PRESERVE |
| BR-EMP-029 | Org chart via `CONNECT BY` up to depth 10; tenure = `MONTHS_BETWEEN/12` rounded to 1 dp. | `get_org_chart`, `get_tenure` | PRESERVE (recursive CTE) |
| BR-EMP-030 | Status values: `ACTIVE, ON_LEAVE, SUSPENDED, TERMINATED`; type: `FULL_TIME, PART_TIME, CONTRACT, INTERN`; gender `M, F, O`. | `01_core_tables.sql` | PRESERVE (enums) |
| BR-EMP-031 | `TRG_EMP_BEFORE_UPDATE` inserts into `EMPLOYEE_HISTORY` columns (`HISTORY_ID, CHANGE_DATE, OLD_VALUE, NEW_VALUE, CHANGED_BY, CHANGE_REASON`) that do not exist in the DDL (`HIST_ID, EFFECTIVE_DATE, OLD_DEPT_ID, …`). Deployed as-is the trigger would be invalid and every `UPDATE EMPLOYEES` would fail. Production must differ from this export. | `trg_employees.sql`, `01_core_tables.sql` | DECIDE (evidence needed) |
| BR-EMP-032 | Client-side e-mail check in the PLL rejects sub-domains (`user@mail.company.com`) while the server-side regex accepts them. | `HRMS_VALIDATION_LIB.validate_email` | FIX (single validator) |

## 3. Leave (`PKG_LEAVE`, `HRMS_LEAVE`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-LV-001 | Form requires leave type, start and end date before submit. | `HRMS_LEAVE.xml` BTN_SUBMIT | PRESERVE |
| BR-LV-002 | Employee must be active (`-20001`); leave type must be active and the employee must have served at least `MIN_TENURE_DAYS` since hire (`-20203`). | `submit_leave_request` | PRESERVE |
| BR-LV-003 | Start ≤ end (`-20210`; also DB check `CHK_LR_DATES`). | `submit_leave_request`, `03_leave_tables.sql` | PRESERVE |
| BR-LV-004 | Start date may be at most 5 days in the past (`-20211`). | `submit_leave_request` | PRESERVE |
| BR-LV-005 | Days = business days (Mon–Fri) minus `HOLIDAYS` for the employee's location or global holidays. Observed holidays are not handled. | `calculate_business_days` | PRESERVE; DECIDE on observed holidays (TD-023) |
| BR-LV-006 | Half-day request counts as 0.5 days regardless of the date range. | `submit_leave_request` | FIX (require start = end) |
| BR-LV-007 | Zero business days → `-20212`. | `submit_leave_request` | PRESERVE |
| BR-LV-008 | Overlap with an existing `PENDING` or `APPROVED` request (inclusive dates) → `-20202`. | `check_overlap` | PRESERVE |
| BR-LV-009 | For accruing leave types, available balance must cover the request (`-20201`). Balance = `OPENING + ACCRUED − USED + ADJUSTMENT − PENDING`. | `get_balance`, `submit_leave_request` | PRESERVE |
| BR-LV-010 | Balance check uses the **current calendar year** (`SYSDATE`), and the `PENDING` increment goes to the **start-date year**. A request across 31 Dec is charged entirely to the start year. | `submit_leave_request` | DECIDE |
| BR-LV-011 | Status on submit: `PENDING` when the type requires approval, else `APPROVED`. Approver = employee's manager; manager is notified. | `submit_leave_request` | PRESERVE |
| BR-LV-012 | For types without approval, the request is inserted as `APPROVED` and then `approve_leave_request` is called, which rejects non-`PENDING` requests (`-20204`). **Auto-approved leave types always fail as written.** | `submit_leave_request` L170–201, `approve_leave_request` | FIX (evidence needed whether production differs) |
| BR-LV-013 | Approve: only `PENDING` (`-20204`); moves days from `PENDING` to `USED`; notifies employee. No check that the approver is the manager or not the requester. | `approve_leave_request` | PRESERVE + FIX (add approver check) |
| BR-LV-014 | Reject: releases `PENDING`; notifies employee. | `reject_leave_request` | PRESERVE |
| BR-LV-015 | Cancel: allowed for `PENDING` (release pending) or `APPROVED` (restore used); the form shows "Only pending or approved requests can be cancelled." and asks for confirmation. | `cancel_leave_request`, `HRMS_LEAVE.xml` | PRESERVE |
| BR-LV-016 | Monthly accrual (1st of month): active employees meeting min tenure get `ACCRUAL_RATE`, capped at `MAX_BALANCE`; balance rows created on demand; logged in `LEAVE_ACCRUAL_LOG`; commits every 100 employees. Not idempotent: running twice accrues twice. | `run_monthly_accrual` | PRESERVE; FIX idempotency (Spring Batch job params) |
| BR-LV-017 | Year-end carryover: remaining balance (excluding pending) capped at `CARRYOVER_MAX` becomes next year's `OPENING_BALANCE`; expiry date derived from `LEAVE_TYPES.CARRYOVER_EXPIRY` and stored in `CARRYOVER_EXPIRY_DT`. | `process_carryover` | PRESERVE |
| BR-LV-018 | Expiry subtracts the full carried-over amount from `ADJUSTMENT`, even if part of it was already used. Source comments call out double subtraction on re-run (TD-012; the assessment report files it under `process_carryover`, but the comment is on `expire_carryover`). | `expire_carryover` | DECIDE |
| BR-LV-019 | Default list shows the current user's requests newest first, plus balances. | `HRMS_LEAVE.xml` | PRESERVE |
| BR-LV-020 | Request status values `PENDING, APPROVED, REJECTED, CANCELLED, TAKEN` (`TAKEN` is never set by code); half-day period `AM`/`PM`; one balance row per (employee, type, year). | `03_leave_tables.sql` | PRESERVE |

## 4. Payroll (`PKG_PAYROLL`, `HRMS_PAYROLL`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-PAY-001 | Salary must be > 0 (`-20101`). A new salary record end-dates the active one at effective date − 1. Basis `ANNUAL`, default currency `USD`, default frequency `MONTHLY`. | `create_salary_record` | PRESERVE |
| BR-PAY-002 | Current salary = active record, or 0 if none. | `get_current_salary` | PRESERVE |
| BR-PAY-003 | Monthly periods end on the last day of the month; pay date moves back to Friday if it falls on a weekend. Bi-weekly periods: 14 days, pay date = period end + 5 days. Period generation commits internally. | `generate_pay_periods` | PRESERVE |
| BR-PAY-004 | Closed period cannot be closed again or get a new run (`-20102`). | `close_pay_period`, `create_payroll_run` | PRESERVE |
| BR-PAY-005 | New run status `PENDING`; type `REGULAR` from the form (`SUPPLEMENTAL, BONUS, FINAL` allowed by DDL). | `create_payroll_run`, `HRMS_PAYROLL.xml` | PRESERVE |
| BR-PAY-006 | Calculation only from `PENDING` is enforced **in the form only**; the package does not check status, so a re-run inserts duplicate details. | `HRMS_PAYROLL.xml` BTN_CALCULATE, `calculate_payroll` | FIX (enforce in service) |
| BR-PAY-007 | Run status `CALCULATING` (committed) → all `ACTIVE`, `ACTIVE_FLAG='Y'` employees processed → `ERROR` if any employee failed, else `CALCULATED`. Hire/termination dates inside the period are ignored (no proration). | `calculate_payroll` | PRESERVE; DECIDE proration |
| BR-PAY-008 | Commit every 50 employees; a failure leaves a partially calculated run. | `calculate_payroll` | FIX (TD-011; Spring Batch chunk + restart) |
| BR-PAY-009 | Per-employee errors are stored as a `PAYROLL_DETAILS` row with status `ERROR`; processing continues. | `calculate_payroll` | PRESERVE (skip policy + error report) |
| BR-PAY-010 | Periods per year: WEEKLY 52, BIWEEKLY 26, SEMIMONTHLY 24, MONTHLY 12. Gross = annual salary / periods, rounded to 2 dp. | `calculate_employee_pay` | PRESERVE |
| BR-PAY-011 | Missing W-4 data defaults to `SINGLE`, 0 allowances. Taxable income = gross (no pre-tax deductions). | `calculate_employee_pay` | PRESERVE; DECIDE pre-tax |
| BR-PAY-012 | Federal tax: annualise, subtract standard deduction (14,600 single / 29,200 married joint) and allowances × 4,300, apply 2024 progressive brackets (single/married-separate and married-joint tables), divide by periods, round 2 dp, add additional withholding. | `calculate_federal_tax` | PRESERVE values; move to config (TD-020). `TAX_BRACKETS` table exists but is unused. |
| BR-PAY-013 | `HEAD_OF_HOUSEHOLD` matches no bracket table → federal tax 0. | `calculate_federal_tax` | FIX |
| BR-PAY-014 | State tax flat rates: CA 7.25%, NY 6.85%, TX/FL/WA 0%, IL 4.95%, PA 3.07%, OH 4.00%, NJ 6.37%, MA 5.00%, other 5.00%. Only applied when a state code exists. | `calculate_state_tax` | PRESERVE (config) |
| BR-PAY-015 | Social Security 6.2% up to YTD wage base 168,600; Medicare 1.45% + 0.9% on YTD wages over 200,000. | `calculate_employee_pay` | PRESERVE (config) |
| BR-PAY-016 | YTD wages are summed **after** the current gross row is inserted, so the current period is counted twice near the wage-base/threshold boundary. | `calculate_employee_pay` | DECIDE (evidence: compare with real payslips) |
| BR-PAY-017 | Employee deductions in priority order: `OVERRIDE_AMOUNT` > `FLAT` (amount or element default) > `PERCENTAGE` of gross. `HOURS`/`FORMULA` element types are not handled. Deductions and benefits are stored negative. | `calculate_employee_pay` | PRESERVE; DECIDE HOURS/FORMULA |
| BR-PAY-018 | Run totals: gross, total deductions (excluding `BENEFIT`), net, employee count. | `calculate_payroll` | PRESERVE |
| BR-PAY-019 | Approve only from `CALCULATED` (`-20103`), plus `PAYROLL/APPROVE` permission in the form. | `approve_payroll`, `HRMS_PAYROLL.xml` | PRESERVE |
| BR-PAY-020 | Reverse has no status check and ignores the reason parameter; marks details `REVERSED`. | `reverse_payroll` | FIX |
| BR-PAY-021 | Payslip YTD fields are placeholders (0). | `get_payslip` | FIX |
| BR-PAY-022 | Pay register CSV written via `UTL_FILE` to directory `PAYROLL_OUTPUT`. | `generate_pay_register` | PRESERVE (format) |
| BR-PAY-023 | Form default shows `OPEN` periods newest first. | `HRMS_PAYROLL.xml` | PRESERVE |

## 5. Performance (`PKG_PERFORMANCE`, `HRMS_PERFORMANCE`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-PRF-001 | Cycle lifecycle `DRAFT → OPEN → CLOSED`; open requires `DRAFT` (`-20401`); close has no status check. DDL also allows `IN_PROGRESS`, `CALIBRATION` (unused). | `PKG_PERFORMANCE`, `04_performance_tables.sql` | PRESERVE; FIX close check |
| BR-PRF-002 | `generate_reviews_for_cycle` creates an `ANNUAL` review in `NOT_STARTED` for each active employee **that has a manager**; existing reviews (duplicate key) are skipped. | `PKG_PERFORMANCE.generate_reviews_for_cycle` | PRESERVE |
| BR-PRF-003 | Self-assessment is allowed from `NOT_STARTED` or `SELF_REVIEW` and moves the review to `MANAGER_REVIEW` (`-20402` otherwise); manager notified. | `PKG_PERFORMANCE` | PRESERVE |
| BR-PRF-004 | Manager rating 1.0–5.0 (`-20403`, DB check). Labels: ≥4.5 Exceptional, ≥3.5 Exceeds Expectations, ≥2.5 Meets Expectations, ≥1.5 Needs Improvement, else Unsatisfactory. Review → `COMPLETED` (no status check). | `PKG_PERFORMANCE` | PRESERVE; FIX status check |
| BR-PRF-005 | Employee acknowledgement `COMPLETED → ACKNOWLEDGED`; silently does nothing in any other status. | `PKG_PERFORMANCE` | DECIDE |
| BR-PRF-006 | Goals have weight, progress and target date; categories `BUSINESS, DEVELOPMENT, LEADERSHIP, INNOVATION, COMPLIANCE`. | `PKG_PERFORMANCE`, DDL | PRESERVE |
| BR-PRF-007 | Form shows `OPEN`/`DRAFT` cycles, newest year first. | `HRMS_PERFORMANCE.xml` | PRESERVE |

## 6. Notifications, audit, common (`PKG_NOTIFICATION`, `PKG_AUDIT`, `PKG_COMMON`, `PKG_VALIDATION`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-CMN-001 | Notifications are queued (autonomous transaction) and sent by a job every 5 minutes, 50 per batch, by priority then age, via SMTP host hard-coded in the package. | `PKG_NOTIFICATION` | PRESERVE async; FIX config |
| BR-CMN-002 | Failed sends set `FAILED` and increment retry count; retries while count < 3. | `PKG_NOTIFICATION` | PRESERVE |
| BR-CMN-003 | Audit and error logging use autonomous transactions and swallow their own errors. | `PKG_AUDIT`, `PKG_COMMON` | DECIDE (outbox pattern) |
| BR-CMN-004 | `AUDIT_LOG.ACTION_TYPE` check allows only `INSERT/UPDATE/DELETE`; the leave status trigger writes `STATUS_CHANGE`, which is rejected and silently dropped. | `trg_audit.sql`, `04_performance_tables.sql` | FIX |
| BR-CMN-005 | Audit purge keeps 365 days. | `PKG_AUDIT.purge_old_records` | PRESERVE |
| BR-CMN-006 | Audit triggers: `TRG_SALARY_AUDIT` and `TRG_DEPARTMENT_AUDIT` (insert/update/delete), `TRG_LEAVE_REQUEST_AUDIT` (status update). | `trg_audit.sql` | PRESERVE (JPA listeners / events) |
| BR-CMN-007 | System parameters read from `SYSTEM_PARAMETERS`; only `EDITABLE_FLAG='Y'` rows can be changed (`-20900`). | `PKG_COMMON` | PRESERVE |
| BR-CMN-008 | Fiscal year starts 1 October (Oct–Dec belong to next FY); quarters follow. | `PKG_COMMON` | PRESERVE |
| BR-CMN-009 | Phone: 10 or 11 digits after stripping formatting. SSN: 9 digits, no all-zero group. | `HRMS_VALIDATION_LIB`, `PKG_VALIDATION` | PRESERVE |
| BR-CMN-010 | Date display `MM/DD/YYYY`, datetime `MM/DD/YYYY HH24:MI:SS`. | `HRMS_COMMON_LIB` | PRESERVE (UI locale) |
| BR-CMN-011 | Form error mapping: 40401 → "No changes to save.", 40501 → "Record is locked by another user…", 40202 suppressed. | `HRMS_EMPLOYEE.xml` ON-ERROR | PRESERVE (409 for lock conflicts) |

## 7. Reporting & integration (`PKG_REPORTING`, `PKG_INTEGRATION`)

| ID | Rule | Source | Disposition |
|---|---|---|---|
| BR-RPT-001 | Reports: headcount, compensation (compa-ratio vs grade midpoint, median), turnover, new hires, leave utilisation, payroll summary, EEO. Active employees only unless stated. | `PKG_REPORTING` | PRESERVE |
| BR-RPT-002 | `refresh_reporting_tables` is an empty placeholder. | `PKG_REPORTING` | DECIDE |
| BR-INT-001 | GL journal file: pipe-delimited `D|cost centre|GL account|debit|credit|description|RUN-<id>` lines and a `T|<count>` trailer; earnings debit, everything else credit. | `PKG_INTEGRATION` | PRESERVE byte-for-byte |
| BR-INT-002 | Benefits file: fixed-width ADP-style export of employees and dependents. | `PKG_INTEGRATION` | PRESERVE byte-for-byte |
| BR-INT-003 | Time & attendance CSV import is a stub (TODO, no DB updates). | `PKG_INTEGRATION` | DECIDE |
| BR-INT-004 | Directory/org sync is a placeholder. | `PKG_INTEGRATION` | DECIDE |

---

## Open questions for the business owner

1. BR-SEC-005, BR-EMP-021, BR-EMP-031, BR-LV-012: the export shows behaviour that cannot work in production (no password check, rehire blocked, invalid trigger, auto-approve fails). Is the production code different? Please provide a fresh export from the live schema.
2. BR-EMP-022: should the hire-date limit be 90 or 180 days?
3. BR-EMP-026: should hires through the UI always run the full hire process (salary, history, notifications)?
4. BR-LV-010: which year should a leave request spanning 31 December be charged to?
5. BR-PAY-007 / BR-PAY-016: are mid-period hires prorated, and is the YTD double count real in production payslips?
6. Which DBMS_SCHEDULER jobs run, and at what times?
