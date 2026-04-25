# AutoTrack Pro — Vehicle Import/Export Operations Platform

A full-stack internal management system built for a Japan-based international vehicle trading and export company. The platform handles the entire vehicle lifecycle — from sourcing at Japanese auction houses through ocean freight, customs clearance, and final delivery across multiple destination markets.

---

## Business Problem Solved

International used-vehicle trading involves a dense chain of operations across multiple countries and time zones: auction purchasing in Japan, domestic port transport, container loading, ocean shipping, customs clearance, destination-country expenses, and final sales. Managing all of this through spreadsheets and disconnected tools leads to data inconsistencies, delayed invoicing, and loss of visibility.

This platform centralises every stage into a single web application with real-time data, role-scoped access, and automated Excel/PDF report generation — eliminating manual handoffs between operations teams in Japan, Africa, and the Middle East.

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend Framework** | PHP — CodeIgniter 2.x (custom-extended) |
| **Database** | MySQL |
| **File Storage** | AWS S3 (images, documents, media) |
| **SMS Notifications** | Twilio API |
| **Reporting** | PHPExcel (Excel) + custom PDF generation |
| **Authentication** | Session-based + Two-Factor Auth (2FA) |
| **Frontend** | jQuery + AJAX, Bootstrap |
| **CLI Automation** | CodeIgniter CLI cron jobs |

---

## Architecture Decisions

### Custom Base Classes
Rather than using vanilla CodeIgniter controllers, the application extends two project-specific base classes:

- **`Jans_Controller`** — handles session validation, super-admin detection, global menu loading, and per-request access-rights resolution before any controller action fires.
- **`Jans_Model`** — provides shared query helpers (paginated gets, soft-delete aware inserts/updates) used consistently across all 30+ models.

This keeps controllers thin and avoids repeating auth boilerplate.

### Granular Role-Based Access Control (RBAC)
Permissions are stored at the individual *function* level (not just controller level). Each user can hold multiple roles. Each role maps to a set of `(menu_id, permission_id)` pairs where `permission_id` represents one of: **view (1), add (2), edit (3), delete (4)**. This allows operations like "can view freight records but not edit them" with no code changes — purely configuration.

### Multi-Country Architecture
Destination markets (e.g., Southern Africa, Middle East) each have country-scoped data and cost configurations. Users can be restricted to specific countries, meaning a Namibia operations team member sees only Namibia freight, expense, and sales data while a super-admin sees the global picture.

### Hybrid MVC + JSON API
The application serves both traditional server-rendered views (for internal dashboards) and JSON endpoints (consumed by AJAX on the same dashboards and by a companion mobile API controller). No separate API server was needed — the same CodeIgniter routing handles both.

---

## Key Modules

### Vehicle Registry (`cars`)
Central record for every vehicle in the pipeline. Stores:
- Chassis/VIN number with duplicate and format validation
- Make, model, year, colour, grade, transmission, drive type, body type
- Auction source, purchase price, FOB price
- Interior/exterior condition grades
- Multi-image upload (stored on AWS S3)
- Status flags: `is_display`, `is_delivery`, `is_auction_sheet`
- Third-party loader, seats bracket, CKD/truck classification

### Auction Management (`auctions`, `engine_auctions`)
Tracks Japanese auction houses by country, auction event records, and per-car auction results. Separate sub-module for engine and parts auctions.

### Sales & Weekly Batch (`sales`)
- Weekly batch management: group vehicles into named weekly sets for coordinated shipping
- Bill of Lading (BL) master records and per-BL container assignment
- Sales records linked to BL shipments

### Transportation & Carriers (`transportations`)
- Carrier master (transport companies, drivers)
- Carrier transport jobs: per-vehicle assignments with rated amounts
- BL detail view: vessel name, ship date, ETA, container breakdown
- Rate admin tools with automatic amount recalculation

### Japan Domestic Transport (`japan_transports`)
Tracks vehicles being transported within Japan from auction yard to export port, including city-to-city routing and vehicle category classification.

### Shipping Invoices (`shipping`)
Generates shipping invoices per BL. Supports multiple invoice types (proforma, commercial, supplementary). Outputs to PDF and Excel via pre-built templates.

### Freight Management (`freights`)
- Per-chassis duty and freight cost tracking
- Country-specific freight rate configuration
- Excel export: chassis-level duty and freight breakdown

### Port Charges / Duty Invoices (`duty`)
Invoice generation for destination-port charges, grouped by BL and container. Supports multiple invoice variants.

### Direct Expenses (`expenses`, `dubai`)
Post-arrival direct expense entry against BL/container, with country-specific rate auto-fill. Dubai module extends this with market-specific expense categories (damage charges, standing guarantees, exit certificates, passport collection tracking).

### Inventory / Stock (`stocks`)
- Opening stock management per country
- Inventory reports with country/date filtering
- Stock cost list, ageing report, on-the-way and invoiced stock views
- Excel and PDF export for all inventory variants

### Inspection & Customs (`inspection`)
Bill-of-entry management for customs clearance. Links inspection records to BL numbers with RORO/container type differentiation.

### Vehicle Reservations (`reservations`, `dubai_reserve`)
Customer vehicle reservation workflow with deposit tracking and reservation status management.

### Price Management (`sale_price_settings`, `price_discounting`)
- Sale price setting per vehicle with country-specific pricing
- Discounting rules configuration
- Mobile API endpoint (`SalePriceApi`) for fetching live sale price lists with pagination and search

### User & Role Management (`users`)
- Role creation with display names and country/city restriction flags
- Per-role permission assignment (view/add/edit/delete per menu function)
- User profile management
- Two-factor authentication toggle
- Forced logout endpoint for session management

### Reporting (`system_reports`, `stock_reports`, `sales_reports`)
Automated Excel report generation covering: sold stock, Dubai sales, vessel arrived pack reports, stock cost lists, transport vehicle reports, weekly batch summaries, and ageing analysis.

### SMS Notifications (`sms`, `sms_api`)
Group-based SMS broadcast via Twilio. Supports contact group management and bulk sends tied to operational events.

### AWS S3 Integration (`aws` model)
Centralised S3 upload/download abstraction supporting images (JPEG, PNG), documents (PDF, XLSX), and media (MP4, audio). Content-type detection is handled automatically by file extension.

### Database Synchronisation (`synchronization`, `cron_job`)
Multi-instance sync: a configurable flag controls whether the sync is allowed to run. A CLI-only `cron_job` controller triggers synchronisation on a scheduled basis, copying records between instances safely.

### Chassis Code Management (`chessis_code`)
VIN/chassis code validation rules with exception handling for non-standard chassis formats. Supports marking chassis as "not-sure" with audit trail.

---

## Database Schema Summary

The schema is structured around the vehicle (car) as the central entity, with the following key table groups:

```
── Vehicle Core
   car_records            Primary vehicle record (chassis, specs, status flags)
   car_images             S3 image references per car
   car_maker / car_color / car_grade / car_body_type / car_transmission  (lookup masters)

── Auction
   auction_master         Auction house / location
   car_auctions           Per-car auction event and result

── Logistics
   weekly_master          Weekly batch groupings
   bl_master              Bill of Lading records
   bl_containers          Container assignments per BL
   carrier_master         Transport carrier companies
   carrier_transport_master / carrier_transportation_detail  Per-job carrier billing

── Financial
   freight_records        Freight cost per chassis
   expense_types_master   Expense category definitions
   direct_expenses        BL-level expense entries (per country)
   shipping_invoices      Generated shipping invoice records
   duty_invoices          Port charge invoice records

── Sales
   sale_records           Vehicle sale events
   sale_price_settings    Country-scoped sale pricing

── Stock
   opening_stock          Per-country opening balance records
   stock_expense_management  Stock-level cost tracking

── Users & Permissions
   admin_menus_new        Menu/function registry (controller_name, menu_function_name)
   roles_master           Role definitions
   role_permissions        (role_id, menu_id, permission_id) mapping
   users_master            User accounts

── Location Masters
   countries / cities / ports / yards
   shipping_companies / forwarding_companies / vessels
```

Soft-delete is used throughout (`delete_state = 0/1`) to preserve audit history.

---

## API Structure Overview

A lightweight JSON API layer is exposed via dedicated controller methods and named routes:

```
POST  /ws/locations/vessels          List vessels
POST  /ws/locations/vessel_add       Add vessel
POST  /ws/locations/vessel_update    Update vessel
POST  /ws/locations/vessel_delete    Delete vessel

GET   /chassis_manufacture_year/:code  Decode chassis manufacturing year
GET   /jjp_ssl                         SSL/connection health check

POST  /getSalePriceList               Paginated sale price list (mobile app)
POST  /ccid                            Toggle car delivery status
POST  /change_car_is_display           Toggle car display status
POST  /get_ckd_info                    CKD/truck classification lookup

POST  /gsc                             Find similar chassis numbers
POST  /gctl                            Chassis transaction log

GET   /dbs/:env                        Database status check
GET   /inv_pdf_downloads/:type/:fmt/:id  Invoice PDF/Excel download
```

All endpoints return JSON `{success: "...", data: [...]}` or `{error: "..."}`. Authentication is session-cookie based for web; `app_user_id` POST parameter for mobile consumers.

---

## Challenges Solved

**1. Chassis uniqueness across a distributed operation**
Duplicate chassis numbers are a real risk when multiple staff are entering vehicles simultaneously. The system validates chassis at insert time with a `callback_check_chassis_add` rule and provides a similar-chassis lookup tool (`/gsc`) so operators can detect near-duplicates before committing.

**2. Multi-country financial isolation**
Each destination market has different freight rates, expense categories, and port charge structures. Rather than hard-coding per-country logic, a country-scoped rate table drives invoice generation. Adding a new market requires configuration, not code changes.

**3. Granular access without complexity**
A flat permission model would make it impossible to give a finance user read-only access to freight while allowing full access to expenses. The `(role, menu_function, permission_type)` three-dimensional RBAC model covers this with zero custom code per user.

**4. Excel reporting at scale**
Operations teams need to export large vehicle lists with calculated columns for ageing, pricing, and costs. PHPExcel is used with streaming writers to avoid PHP memory limits on large datasets.

**5. Multi-instance data consistency**
The company operates staging and production databases. A controlled synchronisation module — gated by a configuration flag and triggerable only from CLI — keeps them consistent without risking accidental overwrites.

**6. AWS S3 file management**
Vehicle records can accumulate dozens of images, PDFs, and audio files. A unified `aws` model wraps the S3 SDK with content-type detection, private ACL enforcement, and optional source-file cleanup after upload, keeping file handling consistent across all modules.

---

## Setup (Local Development)

```bash
# 1. Clone into your web server document root
git clone <repo-url> /path/to/webroot/autotrack

# 2. Install PHP dependencies
composer install

# 3. Configure database
cp jans_dev/config/database.php.example jans_dev/config/database.php
# Edit database.php with your local MySQL credentials

# 4. Configure constants (AWS keys, Twilio credentials, etc.)
cp jans_dev/config/constants.php.example jans_dev/config/constants.php

# 5. Import the database schema
mysql -u root -p autotrack < schema/autotrack.sql

# 6. Point your vhost to the project root — mod_rewrite must be enabled
# The .htaccess rewrites all requests through index.php
```

> **Note:** All credentials (AWS keys, SMS API tokens, DB passwords) are defined as constants in `constants.php` and `constants1.php`, which are excluded from version control via `.gitignore`.

---

## Project Structure

```
/
├── jans_core/          CodeIgniter core (framework files, unmodified)
├── jans_dev/
│   ├── config/         Environment config, routes, DB settings, autoload
│   ├── controllers/    ~35 controllers — one per business domain
│   ├── core/           Jans_Controller.php, Jans_Model.php (base classes)
│   ├── helpers/        Custom helpers (menu_helper, etc.)
│   ├── models/         ~40 models — data access layer
│   ├── views/          Server-rendered HTML views (Bootstrap + jQuery)
│   └── third_party/    PHPExcel
├── downloads/          Server-side Excel file output directory
├── uploads/            Temporary upload staging area
└── vendor/             Composer dependencies (AWS SDK)
```

---

## Author

Built and maintained as the lead backend developer on a multi-developer team. Responsible for system architecture, RBAC design, AWS integration, report engine, API layer, and multi-country financial modules.

**Kashif Umar**
[linkedin.com/in/kashif-umar](https://www.linkedin.com/in/kashif-umar/)

---

> This documentation and architecture is authored by Kashif Umar. Unauthorized use or reproduction is not permitted.
