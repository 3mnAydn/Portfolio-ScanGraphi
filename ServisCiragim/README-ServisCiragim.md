# 🔧 ServisCıragım — Automotive Service & Spare Parts SaaS Platform

> **Status:** Under active development · MVP completed · Pre-beta  
> **Version:** v0.9-beta

---

## 📖 About the Project

ServisCıragım is a multi-tenant SaaS management platform designed for automotive service stations, spare parts dealers, and hybrid businesses.

The platform offers vehicle service tracking, mechanic/staff management, stock control, customer records, and e-invoice/e-archive integration under a single roof.

---

## ✅ Completed Features

### 🏗️ Infrastructure & Architecture
*   **Multi-tenant Architecture** — Each company works securely within its own data thanks to the isolation layer.
*   **Advanced Authentication** — JWT infrastructure with a short-lived Access token and Refresh token system.
*   **Session and Token Management** — A token blacklist structure for secure logout operations and a versioning system that terminates all active sessions upon a password change.
*   **Role-Based Access Control (RBAC)** — Different authorization levels such as system administrator, company manager, office staff, mechanic, and network manager.
*   **Granular Permission System** — Fine-tuned, JSON-based modular authorization infrastructure.
*   **Rate Limiting** — Request limiting mechanism to prevent abuse.
*   **Modular Company Types** — Workflows dynamically filtered according to service, spare part, or hybrid business types.
*   **Audit Log** — IP and user-based recording of critical system operations.
*   **Database Versioning** — Structural database updates via migration tools.
*   **Modern Standards** — PSR-4 standard autoloading architecture.

### 👥 User & Company Management
*   User registration, login, and secure logout processes.
*   Multi-company-based account management (A user can operate in multiple companies).
*   Advanced company profile and approval/rejection workflows via the platform administrator.
*   Staff approval and request mechanisms.

### 🔧 Service Management
*   **Vehicle Intake & Tracking** — Status tracking of vehicles taken into service (Waiting, Waiting for Parts, Completed, Cancelled, etc.).
*   **Staff (Mechanic) Management** — Task assignment, system invitation via email, and profile management.
*   **Work Log System** — Detailed tracking of which staff member performed what action on which vehicle.
*   **Vehicle Owner CRM** — Basic customer record and history management.
*   **Vehicle Delivery Workflow** — System automation applied when the process is completed.
*   **Malfunction & Part Requests** — Request forms supported by a dynamic data structure.

### 🏪 Stock & Product Management
*   Comprehensive product management system.
*   Category module (engine, electronics, brakes, suspension, etc. categorization).
*   VAT rate and concurrent stock quantity tracking.

### 💰 Invoice & Payment
*   Invoice creation, editing, and payment record tracking.
*   Company-specific dynamic invoice line catalog management.
*   **E-invoice & E-archive Integration** — External integrator API connections:
    *   Invoice and archive submission.
    *   Invoice type detection and taxpayer inquiry.
    *   Serial number management and cancellation processes.
*   Bulk invoice creation and draft preparation services.

### 🏢 Hierarchy & Dealer Network
*   Main dealer and sub-branch (franchise) management hierarchy.
*   Ability for network managers to view consolidated sub-dealer data.
*   Automated dealer registration forms.

### 📣 Communication & Support
*   Support ticket management system and central office communication channels.
*   In-system announcement mechanism.
*   Multi-driver supported dynamic email infrastructure (SMTP / API).

### 🖥️ Platform Admin Panel
*   Central management of all registered companies and subscriptions on the system.
*   Toggling platform-wide features (feature toggle).
*   Control of user support, password reset, and communication processes.

### 🌐 Front End (Client)
*   High-performance, framework-independent Vanilla JS SPA (Single Page Application) architecture.
*   **Customized UI Components** — Modals, select boxes, and data tables.
*   Dynamic menu structure shaped according to user authorization.
*   Comprehensive dashboard and interactive module screens.
*   Auto-generated Swagger API documentation for developers.

### 🌍 General Site Pages
*   Corporate web interface (Home, pricing, features, integrations, contact, registration).
*   Legal texts (KVKK/GDPR, Privacy Policy, Terms of Use).

---

## 🗃️ Database Architecture Overview

**DataBase:** MySQL 8+ / InnoDB · Karakter seti: `utf8mb4`

**Core Design Decisions:**
*   **Isolation:** Each table is structurally strictly bound to its respective tenant, preventing data leakage at the query level.
*   **Data Integrity:** "Soft Delete" logic is used to prevent the deletion of critical data.
*   **State Management:** Workflow processes are standardized with enumerated (ENUM) data types.
*   **Flexibility:** JSON data types are used for data blocks requiring dynamic content (part lists, descriptions) to provide flexibility.
*   **Hierarchy:** Dealer and sub-branch relationships are established with self-referencing configurations between tables.

---

## 🚧 To-Do (Future Vision)

### Short-Term
*   **Test Coverage** — Unit Test integrations for authentication, permissions, and invoice workflows.
*   **Frontend Optimization** — Minimizing source files using build tools (Vite/Webpack) for the production environment.
*   **Two-Factor Authentication (2FA)** — OTP or TOTP integration for critical operations.
*   **Email Verification** — Adding an email verification step to user registration processes.

### Medium-Term
*   **Virtual POS Integration** — Establishing live API connections for payment systems (iyzico, Stripe, etc.).
*   **Subscription Automation** — Tracking, notifying, and renewing automation for expired subscriptions.
*   **Advanced Analytics** — Detailed reporting system for businesses, including revenue, staff performance, and service statistics.
*   **File Management** — Secure cloud storage integration for vehicle photos and service documents.

### Long-Term
*   **Multi-Language Support (i18n)** — Making the system suitable for international use.
*   **Accounting Integrations** — Data synchronization with widely used ERP and accounting software.
*   **Mobile Application** — Developing a native mobile application using the existing API infrastructure.

---

## 🔐 Security Policies

*   Passwords are encrypted using modern hashing methods.
*   Short-lived tokens are preferred in session management, and instant cancellation lists (blacklists) are activated to prevent unauthorized access.
*   Prepared statements are used in database interactions, providing full protection against SQL Injections.
*   Tenant data is completely isolated from each other with strict filtering rules at the application layer.

---

## 📦 Core Dependencies

| Package | Purpose |
| :--- | :--- |
| `php-jwt` | Secure token encode/decode operations |
| `phpmailer` | Corporate email delivery infrastructure |
| `phpdotenv` | Secure management of environment variables |
| `phinx` | Database schema and version (migration) management |

---

## 🤝 Contributing

This project reflects the architectural foundation of a commercial venture developed under ScanGraphi. While it is closed to open-source contributions, feedback on code standards and architectural design is always welcome.

---

## 📄 License

All rights of this project are reserved. Unauthorized copying, use, or distribution of the contents is prohibited.

---

<p align="center">
  <em>ServisCıragım — Every service, every dealer, one platform.</em>
</p>
