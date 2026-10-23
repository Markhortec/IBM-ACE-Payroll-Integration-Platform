# 🚀 Enterprise Payroll Integration Platform (IBM ACE 12 + IBM MQ + SQL Server)

A production-grade, highly resilient Enterprise Payroll Integration Platform built using **IBM App Connect Enterprise (ACE v12)**, **IBM MQ**, and **Microsoft SQL Server**.

This platform receives high-volume payroll requests via REST, validates requests, ensures idempotency, decouples processing using IBM MQ queues, calls external REST microservices (Tax & Bank validation), aggregates results per employee, persists transaction audit trails in SQL Server, handles technical/business errors gracefully, and exposes live status querying.

---

## 🏛️ Architecture & Flow Overview

[ POST /payroll ] ──> Gateway Flow ──(MQ Put)──> PAYROLL.REQUEST
│
(MQ Get)
▼
[ Final DB Log ] <── Persistent Storage <── Worker Processing Flow
├──> Call Tax API (Mock)
└──> Call Bank API (Mock)


1. **API Gateway (`Payroll_API_Gateway.msgflow`)**:
   - Accepts REST JSON payload via HTTP POST `/payroll`.
   - Performs **Idempotency Check** via SQL database to prevent duplicate requests.
   - Puts payload into MQ queue `PAYROLL.REQUEST`.
   - Returns immediate Async `202 ACCEPTED` response to caller.

2. **Worker Core (`Payroll_Worker.msgflow`)**:
   - Reads from `PAYROLL.REQUEST`.
   - Iterates through employees, performs DB validation (Active vs Inactive).
   - Calls **Tax Microservice** via HTTP REST to calculate dynamic tax slabs.
   - Calls **Bank Validation Microservice** via HTTP REST.
   - Saves final status (`SUCCESS`, `FAILED_INACTIVE`, `FAILED_BAD_BANK`) to `PAYROLL_TRANSACTION`.
   - Puts final response JSON to `PAYROLL.RESPONSE`.

3. **Global Error Handler & DLQ (`Subflow_GlobalErrorHandler.subflow`)**:
   - Technical exceptions (DB connectivity, HTTP timeouts) are caught and routed to `PAYROLL.ERROR`.
   - MQ Queue Backout Threshold (`BOTHRESH(3)`) routes unhandled poison messages to `PAYROLL.DLQ`.

4. **Audit Logging (`Subflow_AuditLogger.subflow`)**:
   - Asynchronous audit logger writes execution logs to `C:\PayrollAudit\payroll_audit.log`.

5. **Status Query Service (`Payroll_Status_Query.msgflow`)**:
   - Exposes HTTP GET `/payroll/status?requestId=...` to query live aggregated transaction metrics from SQL Server.

---

## 🛠️ Tech Stack & Prerequisites

- **Integration Engine:** IBM App Connect Enterprise (ACE) v12
- **Messaging Middleware:** IBM MQ v9.x (Docker / Containerized)
- **Database:** Microsoft SQL Server (ODBC DSN: `PAYROLLDB`)
- **Protocols:** HTTP REST (JSON), MQ, ESQL, SQL/ODBC

---

## 📥 Queues Configured (IBM MQ)

| Queue Name | Type | Description |
| :--- | :--- | :--- |
| `PAYROLL.REQUEST` | Local | Primary decoupling queue (`BOTHRESH(3)`, `BOQNAME('PAYROLL.DLQ')`) |
| `PAYROLL.RESPONSE` | Local | Asynchronous result destination |
| `PAYROLL.ERROR` | Local | Technical error routing target |
| `PAYROLL.RETRY` | Local | Transient retry staging |
| `PAYROLL.DLQ` | Local | Dead Letter Queue for poison messages |

---

## 🗄️ Database Schema

### `EMPLOYEE` Table
```sql
CREATE TABLE EMPLOYEE (
    employee_id VARCHAR(20) PRIMARY KEY,
    company_id VARCHAR(20),
    name VARCHAR(100),
    status VARCHAR(20) -- ACTIVE / INACTIVE
);


PAYROLL_TRANSACTION Table

CREATE TABLE PAYROLL_TRANSACTION (
    request_id VARCHAR(50),
    company_id VARCHAR(20),
    employee_id VARCHAR(20),
    gross_salary DECIMAL(18,2),
    tax DECIMAL(18,2),
    net_salary DECIMAL(18,2),
    status VARCHAR(100), -- SUCCESS / FAILED_INACTIVE / FAILED_BAD_BANK
    created_at DATETIME DEFAULT GETDATE()
);


🚀 API Endpoints
Method	Endpoint	Description
POST	http://localhost:7800/payroll	Main Payroll Submission Endpoint
GET	http://localhost:7800/payroll/status?requestId=PR-2026-00001	Live Query Status Endpoint
POST	http://localhost:7800/tax/calculate	Mock Tax Service
POST	http://localhost:7800/bank/account/validate	Mock Bank Service

