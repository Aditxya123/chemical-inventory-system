# AGC LAB.CO System Understanding

## Project summary

AGC LAB.CO is a chemical inventory management system built with:

- Node.js + Express backend
- MySQL database
- Plain HTML/CSS/JavaScript frontend
- Nodemailer for optional low-stock email alerts
- `localStorage` for browser-side session and manual wanted rows

The main business flow is:

1. A user signs up or logs in.
2. The user views the chemical inventory.
3. The user adds, edits, deletes, uses, or refills chemicals.
4. Each use/refill creates a log entry.
5. Low stock chemicals appear in reports and may trigger an email alert.

## Main modules

- `server.js`
  Backend server, database bootstrap, API routes, transaction handling, and low-stock alert logic.
- `public/js/main.js`
  Frontend page behavior, API calls, modal handling, filtering, reporting, printing, and `localStorage` use.
- `lab.sql`
  MySQL schema and sample data for `users`, `chemicals`, and `logs`.

## ER Diagram

This version is arranged more like the example image, using entity boxes plus relationship labels.

```mermaid
flowchart LR
    U["USER<br/>- id (PK)<br/>- name<br/>- email (UNIQUE)<br/>- password_hash<br/>- role<br/>- reset_code<br/>- reset_expires<br/>- created_at"]

    C["CHEMICAL<br/>- id (PK)<br/>- name<br/>- formula<br/>- category<br/>- unit<br/>- quantity<br/>- max_quantity<br/>- low_stock_quantity<br/>- hazard_info<br/>- room_no<br/>- expiry_date<br/>- created_at<br/>- updated_at"]

    L["LOG / USAGE_RECORD<br/>- id (PK)<br/>- chemical_id (FK)<br/>- action<br/>- amount<br/>- date<br/>- user<br/>- purpose<br/>- class_name"]

    W["MANUAL_WANTED_ITEM<br/>- chemical<br/>- quantity<br/>- price<br/>- signature<br/>- stored in localStorage"]

    R1{"manages"}
    R2{"has transactions"}
    R3{"extends report"}

    U -->|"1"| R1
    R1 -->|"M (logical app usage)"| C

    C -->|"1"| R2
    R2 -->|"M"| L

    W -->|"M"| R3
    R3 -->|"1 report view"| L
```

Notes:

- `logs.chemical_id` references `chemicals.id`.
- `users` is used for authentication, but `logs.user` stores a text name, not `users.id`.
- The database has 3 real tables: `users`, `chemicals`, and `logs`.
- Manual wanted items are report-only browser data kept in `localStorage`, not in MySQL.
- If you want strict database ER modeling, `USER -> CHEMICAL` is not a physical foreign-key relationship; it is a functional/system relationship only.

## Use Case Diagram

```mermaid
flowchart LR
    User([Staff User])
    Mailer([Email Service])

    UC1((Sign up))
    UC2((Log in))
    UC3((Reset password))
    UC4((View inventory))
    UC5((Add chemical))
    UC6((Edit chemical))
    UC7((Delete chemical))
    UC8((Use chemical))
    UC9((Refill chemical))
    UC10((View logs report))
    UC11((View low stock report))
    UC12((View inventory stock report))
    UC13((Print reports))
    UC14((Add manual wanted item))
    UC15((Receive low stock email))

    User --> UC1
    User --> UC2
    User --> UC3
    User --> UC4
    User --> UC5
    User --> UC6
    User --> UC7
    User --> UC8
    User --> UC9
    User --> UC10
    User --> UC11
    User --> UC12
    User --> UC13
    User --> UC14

    UC8 --> UC15
    UC9 --> UC15
    Mailer --> UC15
```

## Sequence Diagram: Login

```mermaid
sequenceDiagram
    actor User
    participant Browser as Browser UI
    participant Frontend as main.js
    participant Server as Express API
    participant DB as MySQL

    User->>Browser: Enter email and password
    Browser->>Frontend: Submit login form
    Frontend->>Server: POST /api/login
    Server->>DB: SELECT user by email
    DB-->>Server: User row
    Server->>Server: Verify password hash
    Server-->>Frontend: Authenticated user JSON
    Frontend->>Frontend: Save agc_user in localStorage
    Frontend-->>Browser: Redirect to /inventory
```

## Sequence Diagram: Use / Refill Chemical

```mermaid
sequenceDiagram
    actor User
    participant Browser as Browser UI
    participant Frontend as main.js
    participant Server as Express API
    participant DB as MySQL
    participant Mailer as Nodemailer

    User->>Browser: Open Use or Refill modal
    User->>Browser: Submit amount and details
    Browser->>Frontend: Submit transaction form
    Frontend->>Server: POST /api/chemicals/:id/transaction
    Server->>DB: BEGIN TRANSACTION
    Server->>DB: SELECT chemical FOR UPDATE
    DB-->>Server: Current chemical row
    Server->>Server: Validate stock rules
    Server->>DB: UPDATE chemicals.quantity
    Server->>DB: INSERT INTO logs
    Server->>DB: COMMIT
    Server->>DB: SELECT updated chemical
    DB-->>Server: Updated chemical row
    Server->>Server: Check low stock threshold
    alt Low stock and SMTP configured
        Server->>Mailer: sendLowStockEmail()
        Mailer-->>Server: Sent or ignored
    end
    Server-->>Frontend: { ok, chemical, isLowStock }
    Frontend->>Frontend: Refresh inventory
    Frontend-->>Browser: Show alert if low stock
```

## Sequence Diagram: Load Reports

```mermaid
sequenceDiagram
    actor User
    participant Browser as Browser UI
    participant Frontend as main.js
    participant Server as Express API
    participant DB as MySQL
    participant Storage as localStorage

    User->>Browser: Open logs/reports page
    Browser->>Frontend: DOMContentLoaded
    Frontend->>Server: GET /api/logs
    Server->>DB: SELECT logs JOIN chemicals
    DB-->>Server: Report rows
    Server-->>Frontend: Logs JSON

    Frontend->>Server: GET /api/reports/low-stock
    Server->>DB: SELECT chemicals below threshold
    DB-->>Server: Low stock rows
    Server-->>Frontend: Low stock JSON

    Frontend->>Storage: Read manual wanted rows
    Storage-->>Frontend: Manual rows
    Frontend-->>Browser: Render report tables and totals
```

## Class Diagram

This project is not written with ES6 classes, so the class diagram below is a conceptual design diagram showing the main system objects and responsibilities.

```mermaid
classDiagram
    class AppServer {
        +initDb()
        +start()
    }

    class AuthController {
        +signup(req, res)
        +login(req, res)
        +forgotPassword(req, res)
        +resetPassword(req, res)
        +hashPassword(password)
        +verifyPassword(password, stored)
    }

    class ChemicalController {
        +listChemicals(req, res)
        +createChemical(req, res)
        +updateChemical(req, res)
        +deleteChemical(req, res)
        +validateChemical(body)
        +isDuplicateChemicalName(name, excludeId)
    }

    class TransactionController {
        +createTransaction(req, res)
        +sendLowStockEmail(chemical)
    }

    class ReportController {
        +listLogs(req, res)
        +lowStockReport(req, res)
    }

    class Chemical {
        +id
        +name
        +formula
        +category
        +unit
        +quantity
        +max_quantity
        +low_stock_quantity
        +hazard_info
        +room_no
        +expiry_date
    }

    class Log {
        +id
        +chemical_id
        +action
        +amount
        +date
        +user
        +purpose
        +class_name
    }

    class User {
        +id
        +name
        +email
        +password_hash
        +role
        +reset_code
        +reset_expires
    }

    class FrontendApp {
        +init()
        +api(url, options)
        +loadInventory()
        +submitChemical(event)
        +submitTransaction(event)
        +loadLogsReport()
        +loadLowStockReport()
        +loadInventoryStockReport()
    }

    class ReportPrinter {
        +printReportCard(cardId, title)
    }

    class BrowserStorage {
        +getStoredUser()
        +getManualWantedRows()
        +saveManualWantedRows(rows)
    }

    AppServer --> AuthController
    AppServer --> ChemicalController
    AppServer --> TransactionController
    AppServer --> ReportController
    ChemicalController --> Chemical
    TransactionController --> Chemical
    TransactionController --> Log
    AuthController --> User
    FrontendApp --> BrowserStorage
    FrontendApp --> ReportPrinter
```

## Simple system architecture

```mermaid
flowchart TD
    A[User Browser] --> B[HTML Pages]
    B --> C[public/js/main.js]
    C --> D[Express Server]
    D --> E[MySQL Database]
    D --> F[SMTP Email Service]
    C --> G[localStorage]
```

## Important observations

- Authentication is session-less on the server side; the frontend stores the logged-in user in `localStorage`.
- Authorization is minimal. A logged-in browser user can call inventory APIs, but there is no server-side session or token validation.
- `logs.user` is stored as plain text, so log history is not strongly linked to a real `users.id`.
- The transaction route is the most important business flow because it updates stock and creates log records in one database transaction.
- Low stock reporting combines database data with browser-only manual rows.
