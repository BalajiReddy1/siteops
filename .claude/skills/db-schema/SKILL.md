---
name: db-schema
description: >
  Provides the complete SiteOps database schema reference. Load when generating MySQL DDL,
  JPA entities, DTOs, or any code that maps to database columns. Contains exact table
  names, column names, types, constraints, and FK relationships.
user-invocable: false
---

# SiteOps — Complete DB Schema Reference

## Column Naming Conventions

- PKs: `id BIGINT AUTO_INCREMENT PRIMARY KEY`
- Timestamps: `created_at DATETIME DEFAULT CURRENT_TIMESTAMP`, `updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`
- Soft deletes: `deleted_at DATETIME DEFAULT NULL` (contractors, workers, sites, invoices)
- Money: `DECIMAL(12,2)` — always INR rupees, never paise
- Enums: MySQL `ENUM(...)` matching Java enum names exactly
- FK naming: `fk_<child_table>_<parent_table>`
- Index naming: `idx_<table>_<column(s)>`
- Charset: `utf8mb4 COLLATE utf8mb4_unicode_ci` everywhere

---

## Table: contractors
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| name | VARCHAR(200) NOT NULL | Company name |
| phone | VARCHAR(15) UNIQUE NOT NULL | Primary login credential |
| email | VARCHAR(320) | Optional |
| gst_number | VARCHAR(20) | |
| address_line1 | VARCHAR(200) | |
| address_line2 | VARCHAR(200) | |
| city | VARCHAR(100) | |
| state | VARCHAR(100) | |
| pincode | VARCHAR(10) | |
| subscription_tier | ENUM('FREE','PRO','ENTERPRISE') DEFAULT 'FREE' | |
| preferred_language | ENUM('en','hi','mr') DEFAULT 'en' | |
| logo_file_id | BIGINT NULL | FK → files.id |
| created_at | DATETIME | |
| updated_at | DATETIME | |
| deleted_at | DATETIME DEFAULT NULL | Soft delete |

## Table: users
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT NOT NULL | FK → contractors.id |
| email | VARCHAR(320) UNIQUE | Optional |
| phone | VARCHAR(15) | Login phone |
| full_name | VARCHAR(200) NOT NULL | |
| password_hash | VARCHAR(255) NOT NULL | BCrypt |
| role | ENUM('CONTRACTOR_ADMIN','SITE_SUPERVISOR','VIEWER') DEFAULT 'CONTRACTOR_ADMIN' | |
| is_active | BOOLEAN DEFAULT TRUE | |
| last_login_at | DATETIME | |
| created_at | DATETIME | |
| updated_at | DATETIME | |

## Table: sites
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT NOT NULL | FK → contractors.id |
| name | VARCHAR(200) NOT NULL | |
| code | VARCHAR(50) | Short identifier |
| address_line1 | VARCHAR(200) | |
| city | VARCHAR(100) | |
| state | VARCHAR(100) | |
| status | ENUM('ACTIVE','ON_HOLD','COMPLETED','CANCELLED') DEFAULT 'ACTIVE' | |
| start_date | DATE | |
| expected_end_date | DATE | |
| actual_end_date | DATE | |
| builder_name | VARCHAR(200) | |
| builder_phone | VARCHAR(15) | |
| share_token | VARCHAR(64) UNIQUE | For public progress view |
| notes | TEXT | |
| created_at | DATETIME | |
| updated_at | DATETIME | |
| deleted_at | DATETIME DEFAULT NULL | |

## Table: workers
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT NOT NULL | FK → contractors.id |
| name | VARCHAR(200) NOT NULL | |
| phone | VARCHAR(15) | |
| skill_type | ENUM('CIVIL','TILES','WATERPROOFING','FINISHING','ELECTRICIAN','PLUMBER','HELPER','OTHER') | |
| default_daily_rate | DECIMAL(12,2) NOT NULL | Default rate across sites |
| id_proof_type | VARCHAR(50) | Aadhaar, PAN, etc. |
| id_proof_number | VARCHAR(100) | |
| bank_account_number | VARCHAR(30) | |
| bank_ifsc | VARCHAR(20) | |
| is_active | BOOLEAN DEFAULT TRUE | |
| created_at | DATETIME | |
| updated_at | DATETIME | |
| deleted_at | DATETIME DEFAULT NULL | |

## Table: site_assignments
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| site_id | BIGINT NOT NULL | FK → sites.id |
| worker_id | BIGINT NOT NULL | FK → workers.id |
| daily_rate | DECIMAL(12,2) NOT NULL | Rate AT THIS SITE |
| assigned_date | DATE | |
| unassigned_date | DATE DEFAULT NULL | NULL = still assigned |
| assigned_by | BIGINT | FK → users.id |
| UNIQUE | (site_id, worker_id, unassigned_date) | One active assignment per worker per site |

## Table: attendance_records
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| site_id | BIGINT NOT NULL | FK → sites.id |
| worker_id | BIGINT NOT NULL | FK → workers.id |
| attendance_date | DATE NOT NULL | |
| status | ENUM('PRESENT','ABSENT','HALF_DAY') NOT NULL | |
| daily_rate_applied | DECIMAL(12,2) NOT NULL | Snapshot of rate on that day |
| amount_earned | DECIMAL(12,2) NOT NULL | rate, rate/2, or 0 |
| marked_by | BIGINT | FK → users.id |
| notes | VARCHAR(500) | |
| created_at | DATETIME | |
| updated_at | DATETIME | |
| UNIQUE | (site_id, worker_id, attendance_date) | One record per worker per day per site |

## Table: wage_withdrawals
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| worker_id | BIGINT NOT NULL | FK → workers.id |
| site_id | BIGINT NOT NULL | FK → sites.id (withdrawal from which site balance) |
| amount | DECIMAL(12,2) NOT NULL | |
| withdrawal_date | DATE NOT NULL | |
| recorded_by | BIGINT | FK → users.id |
| notes | VARCHAR(500) | |
| created_at | DATETIME | |
| updated_at | DATETIME | |

## Table: files
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| owner_type | VARCHAR(50) | 'contractor','progress_update','invoice','report_job' |
| owner_id | BIGINT | Polymorphic FK |
| filename | VARCHAR(255) | Display name |
| original_filename | VARCHAR(255) | As uploaded |
| content_type | VARCHAR(120) | MIME type |
| storage_key | VARCHAR(512) | MinIO object key |
| size_bytes | BIGINT | |
| checksum_sha256 | CHAR(64) | |
| uploaded_by | BIGINT | FK → users.id |
| created_at | DATETIME | |
| updated_at | DATETIME | |

## Table: invoices
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| site_id | BIGINT NOT NULL | FK → sites.id |
| contractor_id | BIGINT NOT NULL | FK → contractors.id |
| invoice_number | VARCHAR(50) UNIQUE NOT NULL | Auto: INV-2024-0001 |
| builder_name | VARCHAR(200) | |
| builder_phone | VARCHAR(15) | |
| builder_email | VARCHAR(320) | |
| line_items | JSON NOT NULL | [{description, quantity, unit, unit_price, amount}] |
| subtotal | DECIMAL(12,2) | |
| gst_rate | DECIMAL(5,2) DEFAULT 18.00 | |
| gst_amount | DECIMAL(12,2) | |
| total_amount | DECIMAL(12,2) NOT NULL | |
| amount_paid | DECIMAL(12,2) DEFAULT 0.00 | |
| status | ENUM('DRAFT','SENT','PARTIALLY_PAID','PAID','OVERDUE','CANCELLED') DEFAULT 'DRAFT' | |
| sent_at | DATETIME | |
| due_date | DATE | |
| razorpay_payment_link_id | VARCHAR(100) | |
| razorpay_payment_link_url | VARCHAR(500) | |
| pdf_file_id | BIGINT NULL | FK → files.id |
| notes | TEXT | |
| created_at | DATETIME | |
| updated_at | DATETIME | |
| deleted_at | DATETIME DEFAULT NULL | |

## Table: invoice_payments
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| invoice_id | BIGINT NOT NULL | FK → invoices.id |
| amount | DECIMAL(12,2) NOT NULL | |
| payment_date | DATE NOT NULL | |
| payment_mode | ENUM('CASH','UPI','BANK_TRANSFER','CHEQUE','RAZORPAY') NOT NULL | |
| reference_number | VARCHAR(100) | UTR or cheque number |
| razorpay_payment_id | VARCHAR(100) | |
| recorded_by | BIGINT | FK → users.id |
| notes | VARCHAR(500) | |
| created_at | DATETIME | |

## Table: progress_updates
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| site_id | BIGINT NOT NULL | FK → sites.id |
| description | TEXT NOT NULL | |
| milestone_label | VARCHAR(200) | e.g. "Foundation complete" |
| reported_by | BIGINT | FK → users.id |
| reported_at | DATETIME DEFAULT CURRENT_TIMESTAMP | |
| created_at | DATETIME | |
| updated_at | DATETIME | |

## Table: attachments
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| owner_type | VARCHAR(50) | 'progress_update' |
| owner_id | BIGINT | |
| file_id | BIGINT NOT NULL | FK → files.id |
| display_order | INT DEFAULT 0 | |
| caption | VARCHAR(500) | |
| created_at | DATETIME | |

## Table: notifications
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT | FK → contractors.id |
| recipient_type | ENUM('WORKER','BUILDER','CONTRACTOR') NOT NULL | |
| recipient_id | BIGINT NOT NULL | |
| recipient_phone | VARCHAR(15) NOT NULL | E.164 format |
| channel | ENUM('WHATSAPP','SMS') DEFAULT 'WHATSAPP' | |
| template_name | VARCHAR(100) NOT NULL | Meta approved template name |
| message_body | TEXT | Rendered message for log |
| status | ENUM('PENDING','SENT','DELIVERED','FAILED') DEFAULT 'PENDING' | |
| external_message_id | VARCHAR(200) | Meta message ID |
| sent_at | DATETIME | |
| error_message | TEXT | |
| created_at | DATETIME | |

## Table: audit_logs
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT | FK → contractors.id |
| actor_id | BIGINT | FK → users.id |
| action | ENUM('CREATE','UPDATE','DELETE','LOGIN','LOGOUT') NOT NULL | |
| entity_type | VARCHAR(50) NOT NULL | Java class name or 'Invoice', 'Worker', etc. |
| entity_id | BIGINT NOT NULL | |
| change_set | JSON | {before: {}, after: {}} |
| ip_address | VARCHAR(45) | IPv4 or IPv6 |
| created_at | DATETIME DEFAULT CURRENT_TIMESTAMP | Immutable — no updated_at |

## Table: report_jobs
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT AUTO_INCREMENT PK | |
| contractor_id | BIGINT | FK → contractors.id |
| report_type | VARCHAR(50) | ATTENDANCE, INVOICE, SITE_SUMMARY |
| status | VARCHAR(30) DEFAULT 'PENDING' | PENDING, GENERATING, READY, FAILED |
| reference_id | BIGINT | siteId or invoiceId |
| parameters | JSON | {month, year, siteId, etc.} |
| file_id | BIGINT NULL | FK → files.id (null until READY) |
| error_message | TEXT | |
| created_at | DATETIME | |
| completed_at | DATETIME | |

---

## Required Indexes

```sql
-- Attendance: primary query pattern
INDEX idx_attendance_site_date (site_id, attendance_date)
INDEX idx_attendance_worker_date (worker_id, attendance_date)

-- Wages
INDEX idx_withdrawals_worker_site (worker_id, site_id)

-- Invoices
INDEX idx_invoices_contractor_status (contractor_id, status)
INDEX idx_invoices_due_date (due_date)
INDEX idx_invoices_number (invoice_number)

-- Files
INDEX idx_files_owner (owner_type, owner_id)

-- Notifications
INDEX idx_notifications_status (status)
INDEX idx_notifications_contractor (contractor_id)

-- Audit
INDEX idx_audit_entity (entity_type, entity_id)
INDEX idx_audit_actor (actor_id)

-- General FK indexes
INDEX idx_users_contractor (contractor_id)
INDEX idx_sites_contractor (contractor_id)
INDEX idx_workers_contractor (contractor_id)
INDEX idx_assignments_site (site_id)
INDEX idx_assignments_worker (worker_id)
INDEX idx_attendance_site (site_id)
INDEX idx_progress_site (site_id)
```
