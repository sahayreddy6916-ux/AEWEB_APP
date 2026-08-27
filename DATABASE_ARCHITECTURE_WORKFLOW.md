# Autocrat ERP - Database Architecture, API & Workflow Reference

This document provides an end-to-end technical reference for the database layer, REST API endpoints, client services, and application workflows in the **Autocrat ERP Work Order Planning** module.

---

## 1. High-Level System Architecture

The application transitions from hardcoded frontend mock objects to a full relational database model with an Express REST API backend and an asynchronous client fetch layer.

```
┌─────────────────────────────────────────────────────────────┐
│                   React 19 Frontend (Vite)                  │
│   WorkOrderList.tsx  |  CardPlanning.tsx  |  CardRouting    │
└──────────────────────────────┬──────────────────────────────┘
                               │ async fetch calls
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Vite Development Server Proxy               │
│               vite.config.ts (/api -> :3001)                │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP REST Requests
                               ▼
┌─────────────────────────────────────────────────────────────┐
│              Express REST API Server (Port 3001)            │
│                  server/index.js Controllers                │
└──────────────────────────────┬──────────────────────────────┘
                               │ SQLite SQL Queries
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                     SQLite Database Engine                  │
│               server/RDBMS.js  -->  autocrat_erp.RDBMS            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Database Schemas (`server/RDBMS.js`)

The database engine uses **SQLite** with Write-Ahead Logging (`WAL`) mode for low latency and transactional safety. Tables auto-seed default master records if empty on startup.

### 2.1 `fg_items` (Finished Goods Catalog Master)
Stores the master product catalog available for work order planning.

| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `code` | `TEXT` | `PRIMARY KEY` | Unique Finished Good code (e.g. `FG-8092-A1`) |
| `name` | `TEXT` | `NOT NULL` | Full part description |
| `type` | `TEXT` | `NOT NULL` | Category (e.g. Manufactured Parts, Sub-Assembly) |
| `drawing_rev` | `TEXT` | `NOT NULL` | CAD drawing revision version (e.g. `Rev 3.2`) |
| `default_unit` | `TEXT` | `NOT NULL` | Standard manufacturing unit (e.g. `PCS`, `SET`) |
| `default_order_type` | `TEXT` | `NOT NULL` | Default order type (e.g. `For Order (Sales)`) |
| `default_qty` | `INTEGER` | `NOT NULL` | Standard batch production size |

---

### 2.2 `planners` (Planner Team Master)
Stores authorized production planners and coordinator profiles.

| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | `PRIMARY KEY` | Planner identifier (e.g. `P1`) |
| `name` | `TEXT` | `NOT NULL` | Full name and title (e.g. `Alex Chen (Senior Planner)`) |

---

### 2.3 `work_orders` (Work Orders Master Ledger)
Stores all primary production work orders.

| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `work_order_no` | `TEXT` | `PRIMARY KEY` | Work Order Number (e.g. `WO-684-SW`) |
| `create_date` | `TEXT` | `NOT NULL` | Date record was created (`DD-MM-YYYY`) |
| `fg_item_code` | `TEXT` | `NOT NULL` | Foreign key reference to `fg_items.code` |
| `fg_item_name` | `TEXT` | `NOT NULL` | Part name snapshot |
| `item_type` | `TEXT` | `NOT NULL` | Material category |
| `drawing_rev` | `TEXT` | `NOT NULL` | Applicable drawing revision |
| `unit` | `TEXT` | `NOT NULL` | Factory location unit (`Unit-1`, `Unit-2`, `Unit-3`) |
| `order_type` | `TEXT` | `NOT NULL` | Sales Order, Internal Order, Sample, etc. |
| `ordered_qty` | `INTEGER` | `NOT NULL` | Customer ordered quantity |
| `planned_qty` | `INTEGER` | `NOT NULL` | Target shop-floor batch quantity |
| `work_order_date` | `TEXT` | `NOT NULL` | Start/issuance date |
| `proj_comp_date` | `TEXT` | `NOT NULL` | Target completion deadline |
| `planner` | `TEXT` | `NULLABLE` | Assigned planner name |
| `remarks` | `TEXT` | `NULLABLE` | Production directives and notes |
| `status` | `TEXT` | `DEFAULT 'Draft'` | `Draft` \| `Open` \| `Released` \| `Cancelled` |
| `priority` | `TEXT` | `DEFAULT 'Normal'` | `Low` \| `Normal` \| `High` \| `Critical` |

---

### 2.4 `routing_steps` (Manufacturing Sequence Operations)
Stores individual routing steps and machine assignments per work order or item template.

| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | `PRIMARY KEY` | Routing step ID (e.g. `r1_a1` or `route_17192837`) |
| `work_order_no` | `TEXT` | `NULLABLE` | Specific Work Order number (if customized) |
| `fg_item_code` | `TEXT` | `NULLABLE` | Fallback FG Item Code template |
| `sequence` | `INTEGER` | `NOT NULL` | Operation order (10, 20, 30, 40...) |
| `operation` | `TEXT` | `NOT NULL` | Name of operation (e.g. `Laser Cutting`, `CNC Bending`) |
| `work_center` | `TEXT` | `NOT NULL` | Shop floor work station (e.g. `WC-LASER-01`) |
| `machine_id` | `TEXT` | `NOT NULL` | Machine identifier (e.g. `L-CUT-3000`) |
| `cycle_time_minutes` | `REAL` | `NOT NULL` | Cycle duration in minutes per unit |
| `assigned_operator` | `TEXT` | `NULLABLE` | Designated operator name |

---

### 2.5 `allocated_materials` (Bill of Materials & Inventory Allocation)
Stores raw component requirements and inventory allocation statuses.

| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | `PRIMARY KEY` | Material entry ID |
| `work_order_no` | `TEXT` | `NULLABLE` | Work Order reference |
| `fg_item_code` | `TEXT` | `NULLABLE` | FG Item Code reference |
| `item_code` | `TEXT` | `NOT NULL` | Raw component code (e.g. `RM-STL-4x8-01`) |
| `description` | `TEXT` | `NOT NULL` | Material description |
| `required_qty` | `INTEGER` | `NOT NULL` | Calculated required quantity |
| `available_qty` | `INTEGER` | `NOT NULL` | Current warehouse stock |
| `unit` | `TEXT` | `NOT NULL` | Unit of measure (`PCS`, `SHT`, `KG`, `MTR`) |

       CREATE TABLE IF NOT EXISTS allocated_materials (
        id              TEXT PRIMARY KEY,
        work_order_no   TEXT,
        fg_item_code    TEXT,
        item_code       TEXT NOT NULL,
        description     TEXT NOT NULL,
        required_qty    NUMERIC NOT NULL DEFAULT 0,
        available_qty   NUMERIC NOT NULL DEFAULT 0,
        allocated_qty   NUMERIC NOT NULL DEFAULT 0,
        unit            TEXT NOT NULL
      );
---

## 3. REST API Specifications (`server/index.js`)

All endpoints respond with standard JSON payloads.

| HTTP Method | Route | Description | Query / Body Parameters | Response |
| :--- | :--- | :--- | :--- | :--- |
| **GET** | `/api/fg-items` | Fetch all Finished Goods catalog items | None | `Array<FGItem>` |
| **GET** | `/api/planners` | Fetch active planner list | None | `Array<{ id, name }>` |
| **GET** | `/api/work-orders` | Fetch work orders for dashboard table | None | `Array<WorkOrderListItem>` (Ordered by WO No DESC) |
| **GET** | `/api/work-orders/:woNo` | Fetch single Work Order details | `woNo` (URL Param) | Single `WorkOrderState` object |
| **POST** | `/api/work-orders` | Insert a new Work Order record | JSON `WorkOrderState` body | `{ message, workOrderNo }` |
| **PUT** | `/api/work-orders/:woNo` | Update existing Work Order fields | `woNo` (URL Param) + JSON body | `{ message, woNo }` |
| **DELETE** | `/api/work-orders/:woNo` | Delete Work Order + related routings/materials | `woNo` (URL Param) | `{ message, woNo }` |
| **GET** | `/api/routing` | Fetch routing steps for WO or FG Item | `?workOrderNo=...` or `?fgItemCode=...` | `Array<RoutingStep>` |
| **POST** | `/api/routing` | Add a new routing operation step | JSON `RoutingStep` body | `{ message, id }` |
| **DELETE** | `/api/routing/:id` | Delete routing step | `id` (URL Param) | `{ message }` |
| **GET** | `/api/materials` | Fetch BOM material allocations | `?workOrderNo=...` or `?fgItemCode=...` | `Array<AllocatedMaterial>` |

---

## 4. Client API Layer (`src/api.ts`)

The frontend uses helper functions in `src/api.ts` to perform non-blocking HTTP fetch requests:

- `fetchFGItemsRDBMS()` $\rightarrow$ Queries `/api/fg-items`
- `fetchPlannersRDBMS()` $\rightarrow$ Queries `/api/planners`
- `fetchWorkOrdersRDBMS()` $\rightarrow$ Queries `/api/work-orders`
- `fetchWorkOrderDetailsRDBMS(woNo)` $\rightarrow$ Queries `/api/work-orders/:woNo`
- `saveWorkOrderRDBMS(wo, isNew)` $\rightarrow$ Sends `POST` or `PUT` request to `/api/work-orders`
- `deleteWorkOrderRDBMS(woNo)` $\rightarrow$ Sends `DELETE` request to `/api/work-orders/:woNo`
- `fetchRoutingStepsRDBMS(woNo, itemCode)` $\rightarrow$ Queries `/api/routing`
- `addRoutingStepRDBMS(step, woNo, itemCode)` $\rightarrow$ Sends `POST` to `/api/routing`
- `deleteRoutingStepRDBMS(id)` $\rightarrow$ Sends `DELETE` to `/api/routing/:id`
---

## 5. End-to-End Application Workflows

### Workflow 1: Dashboard Loading & Real-Time Filtering
1. User loads the main dashboard view (`WorkOrderList.tsx`).
2. Component triggers `fetchWorkOrdersRDBMS()` inside `useEffect`.
3. Express server executes `SELECT` query on `work_orders` table and returns JSON array.
4. User types in search box or adjusts filters (Unit / Status). Table filters instantly in memory.
5. User clicks **Delete icon**: UI sends `DELETE /api/work-orders/:woNo`. Server removes record from SQLite RDBMS and updates UI table.

---

### Workflow 2: Initiating a New Work Order & ERP Handshake (SO Load)
1. User clicks **"Add New"** button. Frontend initializes empty `WorkOrderState` with auto-generated ID (e.g. `WO-686-342`).
2. User selects **Order Type** (e.g. `For Order (Sales)`) and selects **FG Item Code** (e.g. `FG-8092-A1`) from the RDBMS-populated autocomplete dropdown.
3. User clicks **"Load Order (RDBMS)"**:
   - Component queries RDBMS catalog for `FG-8092-A1`.
   - Auto-populates `fgItemName`, `itemType`, `drawingRev`, `orderedQty`, `plannedQty`, and default `planner`.
   - Executes `saveWorkOrderRDBMS(updatedWO, true)` to insert the initial draft record into SQLite table `work_orders`.
   - Toast notification alerts: *"Database Handshake Succeeded"*.

---

### Workflow 3: Master Planning Updates & Instant RDBMS Persistence
1. In Stage 1 (`CardPlanning.tsx`), user modifies quantities, dates, location unit, or remarks.
2. User clicks on **Status Pill** (`Draft` $\rightarrow$ `Open` $\rightarrow$ `Released` $\rightarrow$ `Cancelled`) or **Priority Pill** (`Low` $\rightarrow$ `Normal` $\rightarrow$ `High` $\rightarrow$ `Critical`).
3. State cycles and immediately invokes `saveWorkOrderRDBMS(updatedWO, false)`.
4. Express server executes SQL `UPDATE work_orders SET status = ?, priority = ? WHERE work_order_no = ?`.
5. Database updates immediately.

---

### Workflow 4: Stage 2 Routing Sequence Customization
1. User clicks **"Proceed to Routing Stage Planning"**.
2. Stage 2 (`CardRouting.tsx`) queries `/api/routing` and `/api/materials` for `workOrderNo` / `fgItemCode`.
3. Operations list renders in chronological sequence order (10, 20, 30...).
4. User clicks **"Add Sequence"**, fills form (Operation, Work Center, Machine ID, Cycle Time), and submits:
   - Component updates local React array for zero lag.
   - Component calls `addRoutingStepRDBMS(...)` $\rightarrow$ `POST /api/routing`.
   - Server inserts row into `routing_steps` SQLite table.
5. User clicks **Trash Icon** on a routing step:
   - Component removes step from UI and calls `deleteRoutingStepRDBMS(id)` $\rightarrow$ `DELETE /api/routing/:id`.

---

### Workflow 5: Material Deficit Check & Work Order Release
1. Stage 2 dynamically scales BOM required quantities based on `plannedQty` from Stage 1. 
2. If stock is missing: Pill flags **"Partial Shortage"**. User clicks **"Request Safety Stock"** to approve store transfer.
3. Once stock requirements are met, user clicks **"Release Work Order to Shop Floor"**:
   - Work order status updates to `"Released"`.
   - Database record is updated via `saveWorkOrderRDBMS`.
   - Routing traveler dispatch notice is confirmed.
