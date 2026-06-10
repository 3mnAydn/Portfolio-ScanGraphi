# KAYA Assistance

An end-to-end digital platform developed for 24/7 roadside assistance and support services.

KAYA Assistance is a monolithic web application comprising a corporate website, multi-role management panels, and a REST-based PHP API layer. It centralizes package sales via a dealer network, contract management, payment flows, incident reporting, and accounting integrations into a single system.

---

## Overview

| Field | Description |
|------|----------|
| **Project** | KAYA Assistance |
| **Industry** | Roadside Assistance / Towing Services |
| **Architecture** | PHP REST API + Vanilla JS Panels + Relational Database |
| **Frontend** | Modern CSS Framework, Responsive Design |

The platform hosts both the customer-facing corporate site and operational management panels on the same codebase. Post-sale contract PDF generation, commission calculations, supplier payments, and external system integrations operate securely on this infrastructure.

---

## Modules and Panels

### Corporate Website
- Landing page, services, contact, and request forms.
- Promotion of 24/7 roadside assistance and towing services.
- Application flows for dealers and suppliers.

### Management Panels

| Panel | User Role | Core Functions |
|-------|----------------|----------------|
| **Admin** | Administrator | Contract search, sales, incident management, accounting, user management |
| **Dealer** | Authorized Dealer | Package sales, payments, support requests, contract tracking |
| **Personnel** | Field Staff | Operational tracking and procedures |
| **Supplier** | Tow Truck / Driver | Service records, payment requests |
| **Executive** | Top Management | Balance matching, advanced reporting |

---

## Key Features

### Package Sales and Contract Management
- Package sales via the dealer panel (credit card or account balance).
- Automated contract number generation and PDF archiving.
- Contract search, update, deletion, and bulk export (Excel/PDF).

### Payment Infrastructure
- **Virtual POS** integration (3D Secure and direct provision).
- Atomic transaction recording post-payment approval.
- POS transaction history, dealer payment approvals, and commission tracking.

### Accounting and Finance
- Balance sheets and automated commission calculations.
- Monthly and annual account reporting.
- Dealer and supplier payment request/approval workflows.

### Security Layer
- Session-based authentication with role-based access control (RBAC).
- Rate limiting mechanism.
- Bot protection via Captcha integration.
- Location verification and IP-based access control.
- Login attempt logging and robust SQL injection protection via prepared statements.

---

## External API Integrations

### E-Invoice / E-Archive Integration (SOAP)
The post-sale e-invoice process is fully integrated with an external billing service provider using a SOAP architecture.

**Capabilities:**
- Automatic dispatch of invoice creation requests upon successful package sale.
- Response status tracking securely stored in the database.
- Real-time invoice status display in the Admin Panel.
- Dynamic column mapping for various schema types (E-Archive, E-Invoice).

### Incident (Damage/Breakdown) Management Integration
Designed to work synchronously with contract data for the core roadside assistance operations.

**Capabilities:**
- Contract-based incident entry flow with automated allowance calculations (accident, breakdown, tire, fuel).
- Centralized incident tracking covering location data, contact info, and media attachments.
- Secure API bridge for transmitting incident data to external operational/CRM systems.
- Strict role-based isolation for incident reporting.

### Additional Integrations
- **SMS Gateway:** Automated notifications.
- **Data Export Bridge:** XML formatted sales/commission data presentation for external systems (API Key authenticated).
- **IP Geolocation:** Enhanced security and audit logging.

---

## Technology Stack

| Layer | Technology |
|--------|-----------|
| Backend | PHP 8.x (Strict Types) |
| Database | Relational Database (PDO) |
| Frontend | HTML5, Vanilla JavaScript, CSS |
| Document Engine | PDF Generation Library |
| Security | Captcha, Rate Limiting, Secure Sessions |

---

License
This project is the proprietary property of KAYA Assistance. Unauthorized use, copying, or distribution of the source code is strictly prohibited.
