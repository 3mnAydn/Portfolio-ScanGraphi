# 🚛 Tow Truck & Dealer Management Platform

> A full-stack B2B SaaS platform built for a logistics company operating in Turkey. The system manages tow truck dispatch, dealer networks, package sales, invoicing, and multi-role user authentication — and is currently running live in production.

---

## 📌 Project Overview

This platform was developed as a **custom enterprise solution** for a real-world business operating in the roadside assistance and automotive logistics sector. It handles the complete operational lifecycle: from dealer onboarding and package sales to tow truck dispatch, payment processing, and automated e-invoice generation.

## ✨ Key Features

### 🏢 Multi-Role Authentication System
- **Admin Panel** — Full control over the platform (users, dealers, trucks, financials)
- **Dealer Panel** — Dealers manage their own sales, packages, and payment requests
- **Driver / Truck Panel** — Field operators receive dispatch notifications and update job status
- **Executive Portal** — High-level reporting and oversight dashboard

### 📦 Package & Sales Management
- Configurable service packages with pricing and commission structures
- Per-dealer package toggling and commission overrides
- Sold packages tracking with date filtering and export
- Balance-based sales flow (dealer sells from pre-loaded balance)

### 💳 Payment & Financial Operations
- Garanti Bank 3D Secure POS integration (card payments)
- Dealer payment requests with receipt upload and admin approval workflow
- Supplier payment management with upload and tracking
- Monthly and yearly financial account summaries
- Commission calculation and balance reconciliation

### 🧾 E-Invoice Integration
- Automated e-invoice generation via **Nilvera** integration
- PDF invoice creation and download
- Bulk invoice creation from sold packages (date-based batch processing)
- Invoice CRUD with admin controls

### 🚨 Incident (İhbar) Management
- Tow truck dispatch request creation and tracking
- Geo-location based assignment
- File/photo upload support per incident
- Dealer-level stats per incident type

### 📋 Contract Management
- Digital contract creation, update, and deletion
- PDF contract generation with signature support
- Contract export (Excel, PDF, ZIP)
- Agreement-to-contract migration tooling

### 📢 Announcements & Support
- Admin-to-dealer announcement broadcasting
- Read tracking per dealer
- Support ticket submission and admin reply system

### 📊 Reporting & Analytics
- Portal statistics dashboard
- Dealer activity reports (no-sales detection)
- Balance list enrichment and cross-referencing
- Visit tracking and logging

### 🔐 Security & Infrastructure
- Rate limiting middleware (`gate-rate-limit.php`)
- CSRF protection and session hardening
- reCAPTCHA on public-facing forms
- Role-based access control enforced at the API layer
- `.htaccess` based route protection

---

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | PHP (vanilla, procedural + OOP patterns) |
| **Frontend** | HTML5, CSS3, Vanilla JavaScript |
| **Database** | MySQL / MariaDB |
| **Payment Gateway** | Garanti Bank 3D Secure API |
| **E-Invoice** | Nilvera API |
| **SMS** | NetGSM API |
| **PDF Generation** | Custom PHP PDF writer (FPDF-based) |
| **Session Management** | Custom PHP session layer with security hardening |
| **Hosting** | Shared/VPS Linux hosting with Apache + `.htaccess` routing |

---

## 🔒 Security Notice

This repository is a **development/backup snapshot** intended for portfolio purposes only.

The following items are **intentionally excluded** from this repository:
- Database credentials and connection strings
- Live API keys (Garanti Bank, Nilvera, NetGSM, reCAPTCHA)
- Session secrets and encryption keys
- Production server configuration files
- Any personally identifiable data (PII) or customer data

Do **not** attempt to connect this codebase to any live database or payment gateway without proper configuration and security review.

---

## 🚀 What I Built

This project was developed **end-to-end** by me as a freelance/contract developer. My responsibilities included:

### 🚨 Incident (İhbar) Management System *(personally designed & built)*
One of the core modules I designed and implemented from scratch is the full incident dispatch and tracking system:
- Incident creation with geo-location data and categorization
- Real-time assignment flow to available tow truck drivers
- File/photo upload per incident for documentation
- Status tracking across admin, dealer, and driver panels
- Per-dealer statistical reporting on incident activity
- Tow truck payment upload and approval per resolved incident

### 🔌 External API Integrations *(personally researched, integrated & maintained)*
All third-party service integrations were implemented by me, including:
- **Garanti Bank 3D Secure POS** — Full payment flow: session init, 3D redirect, async callback handling, and refund processing
- **Nilvera E-Invoice API** — Draft creation, submission, and status tracking for Turkish official e-invoice (e-Fatura) infrastructure
- **NetGSM SMS API** — Transactional SMS notifications (e.g. payment confirmations, admin alerts)
- **Google reCAPTCHA** — Bot protection on public-facing forms

### 🏗️ Other Core Responsibilities
- **System Architecture** — Designing the multi-role, multi-panel structure with a shared API layer
- **Backend Development** — Building 100+ PHP API endpoints covering all business logic
- **Frontend Development** — Building all admin, dealer, driver, and truck panels using HTML/CSS/JS
- **Financial Logic** — Designing the commission, balance, and reconciliation calculation engine
- **PDF Generation** — Building custom PDF generators for invoices, contracts, and receipts
- **Security Hardening** — Implementing rate limiting, session security, CSRF protection, and RBAC
- **Database Design** — Designing and maintaining the full relational schema
- **Deployment & DevOps** — Managing hosting, `.htaccess` rules, and production migrations

---

## 📈 Scale & Status

- ✅ **Live in production** — actively used by real users daily
- 👥 Multi-dealer network (dealers, sub-users, drivers, admin staff)
- 🧾 Integrated with Turkey's official e-invoice infrastructure
- 💰 Real financial transactions processed through integrated payment gateway

---

## 📄 License

This codebase is shared for **portfolio and demonstration purposes only**.  
All business logic, design patterns, and implementation details are the intellectual property of the original developer.  
Unauthorized commercial use or redistribution is prohibited.

---

*Built with ❤️ — A production-grade enterprise platform from scratch.*
