# MediQueue DSA — Hospital Management System Using Data Structures and Algorithms

A full-stack, enterprise-grade Hospital Management System that demonstrates how **manual Data Structures and Algorithms (DSA)** solve critical operational challenges in clinical workflows: outpatient queuing, emergency trauma triage, instant medical record indexing, longitudinal clinical history tracking, and patient demographic analytics.

Built with **React.js, TypeScript, Tailwind CSS, Vite**, a **Python FastAPI** backend, **MongoDB** persistent storage (with an automated in-memory MongoDB-compatible fallback for zero-dependency local execution), and **six custom-coded DSA algorithms** implemented from first principles without relying on built-in sorting or dictionaries where custom DSA is required.

---

## 1. Project Overview & Architectural Flow

MediQueue DSA bridges theoretical computer science with real-world healthcare operations. The system enforces strict architectural layers:

```
[ React + TypeScript Frontend (Vite) ]
               │  REST API (JSON over HTTP / JWT Auth)
               ▼
   [ FastAPI Backend Gateway ]
        │                 │
        │ In-Memory Sync  │ Persistent DB Operations
        ▼                 ▼
[ 6 Manual DSA Engines ]   [ MongoDB Database Collections ]
  - FIFO Queue               - patients
  - Binary Max-Heap          - doctors
  - Separate Chaining Hash   - appointments
  - Singly Linked List       - waiting_queue
  - Merge Sort               - emergency_queue
  - Insertion Sort           - medical_history
                             - users
```

### The Seven-Step Integration Lifecycle:
1. **User Action:** A user performs an action on the React frontend (e.g., booking an appointment, enqueuing a patient, triage admission, record search, or sorting).
2. **REST Invocation:** The frontend dispatches an asynchronous HTTP request with optional Bearer JWT credentials to FastAPI endpoints.
3. **Validation & Conflict Checks:** FastAPI and Pydantic models validate input payloads and run business rules (such as validating that a doctor is not double-booked at the same date and time).
4. **Persistent MongoDB Operations:** The database layer executes persistent CRUD operations against MongoDB collections (`patients`, `doctors`, `appointments`, etc.). MongoDB remains the absolute **source of truth**.
5. **DSA Synchronization & Processing:** Custom in-memory DSA structures are updated or traversed (e.g., node enqueue in FIFO queue, `heapify_up` in binary max-heap, collision resolution in hash table, chronological pointer traversal in linked list).
6. **Algorithmic Diagnostics:** The backend captures operational metrics (comparisons, step counts, recursion depth, pointer traversals, execution time in milliseconds).
7. **Reactive UI State:** The frontend updates state, displaying visual node pipelines, binary heap tree levels, bucket chains, and diagnostic cards.

---

## 2. Technology Stack

| Layer | Technology | Details |
| :--- | :--- | :--- |
| **Frontend Framework** | React 19 + TypeScript | Component-driven UI with strong static typing |
| **Styling & Icons** | Tailwind CSS v4 + Lucide Icons | Responsive healthcare design, custom badges, dark/light accents |
| **Frontend Build Tool** | Vite | Ultra-fast HMR and optimized production bundling |
| **Backend Framework** | Python 3.11+ / FastAPI | High-performance asynchronous REST API architecture |
| **Persistent Database** | MongoDB | Document database for durable patient, doctor, and history storage |
| **DB Client & Fallback** | Motor (async) + MongoMock Engine | Connects to live MongoDB or falls back to MongoMock if offline |
| **Authentication** | JWT (JSON Web Tokens) + Bcrypt | Secure password hashing and token-based RBAC |
| **DSA Implementations** | Manual Python Algorithms | Hand-coded from scratch without built-in sorting/lookup shortcuts |

---

## 3. The Six Mandatory DSA Implementations

### A. FIFO Queue (Doubly-Linked Nodes) — Outpatient Waiting Room
- **Use Case:** Manages non-critical outpatient consultations in First-In, First-Out sequence.
- **Implementation:** Custom `FIFOQueue` with `QueueNode` containing `data`, `next`, and `prev` pointers.
- **Operations:** `enqueue` (Tail O(1)), `dequeue` (Head O(1)), `peek` (O(1)), `is_empty`, `size`, `display` (O(n)).
- **Why O(1):** Dual head and tail pointers avoid list shifting overhead ($O(n)$ in naive arrays).

### B. Priority Queue (Binary Max-Heap) — Emergency Trauma Triage Simulation
- **Use Case:** Prioritizes emergency cases by clinical urgency (Score 1 to 5, where 5 is resuscitation).
- **Academic Disclaimer:** *Educational simulation of heap algorithms; not an FDA/MDR-approved clinical triage system.*
- **Implementation:** Array-backed binary tree using parent-child index math: `parent(i) = (i-1)//2`, `left(i) = 2i+1`, `right(i) = 2i+2`.
- **Operations:** `insert` (O(log n)), `extract_max` (O(log n)), `peek` (O(1)), `heapify_up`, `heapify_down`, `build_heap` (O(n) Floyd algorithm).
- **Tie-Breaking Rule:** Maintains arrival order for equal priority scores using an auto-incrementing sequence counter.

### C. Separate Chaining Hash Table — Patient Record Search & Indexing
- **Use Case:** High-speed lookup of patient demographic documents using patient IDs (e.g. `P-1001`).
- **Implementation:** Fixed/dynamic bucket array where each bucket holds a singly-linked list of `HashNode` objects.
- **Hash Function:** Horner’s polynomial rolling hash: $h = \left(\sum s[i] \cdot 31^{n-1-i}\right) \pmod{\text{capacity}}$.
- **Collision Resolution:** Appends colliding nodes to bucket linked lists; automatic rehashing when load factor $\alpha > 0.75$.
- **Complexity:** Average $O(1)$ search, insert, delete; worst-case $O(n)$ under pathological collision.

### D. Singly Linked List — Chronological Medical History
- **Use Case:** Maintains a patient’s historical clinic consultations, diagnoses, and prescriptions.
- **Implementation:** `MedicalVisitNode` with pointers to the next sequential clinical visit.
- **Operations:** `append` (Tail O(1)), `insert_at_beginning` (Head O(1)), `search` by ID/keyword (O(n)), `delete` (O(n)), `traversal` (O(n)).
- **Persistence:** Reconstructed dynamically from MongoDB visit documents.

### E. Manual Merge Sort — Stable Patient Sorting
- **Use Case:** Deterministic sorting of hospital patient registries by `name`, `age`, or `registration_date`.
- **Implementation:** Pure divide-and-conquer (`_merge_sort_recursive` and `_merge`). Built-in `sorted()` and `list.sort()` are strictly prohibited.
- **Complexity:** Best, Average, and Worst-Case $O(n \log n)$; Space $O(n)$.
- **Stability:** Stable (preserves relative arrival order of equal keys).

### F. Manual Insertion Sort — Adaptive In-Place Sorting
- **Use Case:** Incremental sorting of incoming streams or small/nearly sorted patient cohorts.
- **Implementation:** In-place shifting of sorted prefix.
- **Complexity:** Best-case $O(n)$ (already sorted); Average/Worst-case $O(n^2)$; Space $O(1)$ in-place.
- **Stability:** Stable.

---

## 4. Setup and Execution Instructions

### Prerequisites
- **Python:** 3.11 or higher
- **Node.js:** v18 or higher (Node v20+ recommended)
- **MongoDB (Optional):** Local MongoDB daemon or MongoDB Atlas connection string. If MongoDB is not running, the application automatically activates its embedded MongoMock-Async engine so everything runs out-of-the-box.

---

### Step 1: Clone and Navigate to Repository
```bash
git clone <repository-url>
cd MediQueue
```

---

### Step 2: Backend Setup
```bash
# Navigate to backend directory
cd backend

# Create virtual environment (optional but recommended)
python -m venv venv

# Activate virtual environment
# Windows:
venv\Scripts\activate
# Linux/macOS:
# source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run backend test suite (Unit tests + Integration tests)
python -m pytest tests -v

# Launch FastAPI server
python run.py
```
*Backend server runs at:* `http://127.0.0.1:8000`
*Interactive Swagger API Docs:* `http://127.0.0.1:8000/docs`

---

### Step 3: Frontend Setup
Open a new terminal window:
```bash
# Navigate to frontend directory
cd frontend

# Install npm dependencies
npm install

# Build for production verification
npm run build

# Start Vite development server
npm run dev
```
*Frontend web application runs at:* `http://localhost:5173`

---

## 5. Seed Accounts & Pre-Configured Credentials

The database seeds automatically on startup with synthetic clinical data and test accounts:

| Role | Username | Password | Access Capabilities |
| :--- | :--- | :--- | :--- |
| **Administrator** | `admin` | `admin123` | Full administrative control, user management, audit logs |
| **Physician** | `doctor` | `doctor123` | Consultation, prescription authoring, appointments |
| **Triage Nurse** | `nurse` | `nurse123` | Emergency admission, waiting room queue control |
| **Guest Clinician** | *Anonymous* | *None* | Read-only browsing and interactive testing |

---

## 6. Directory Structure

```
MediQueue/
├── README.md                          # Project documentation and setup guide
├── DOCUMENTATION.md                   # Full academic architecture, schema & code listings
├── VIVA_QUESTIONS.md                  # Comprehensive viva voce preparation guide
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                    # FastAPI application gateway & lifespan
│   │   ├── config.py                  # Environment settings & JWT configuration
│   │   ├── database.py                # Dual MongoDB / MongoMock layer & DSA singletons
│   │   ├── auth.py                    # Bcrypt password hashing & JWT token verification
│   │   ├── seed_data.py               # Synthetic hospital data seeder
│   │   ├── dsa/                       # Hand-coded Data Structures & Algorithms
│   │   │   ├── __init__.py
│   │   │   ├── queue.py               # A. FIFO Queue (Doubly Linked Nodes)
│   │   │   ├── priority_queue.py      # B. Binary Max-Heap Priority Queue
│   │   │   ├── hash_table.py          # C. Separate Chaining Hash Table
│   │   │   ├── linked_list.py         # D. Singly Linked List (Medical History)
│   │   │   ├── merge_sort.py          # E. Manual Merge Sort
│   │   │   └── insertion_sort.py      # F. Manual Insertion Sort
│   │   ├── models/                    # Pydantic schemas
│   │   │   ├── patient.py
│   │   │   ├── doctor.py
│   │   │   ├── appointment.py
│   │   │   ├── queue_item.py
│   │   │   ├── emergency.py
│   │   │   ├── medical_history.py
│   │   │   └── user.py
│   │   └── routers/                   # Modular REST API endpoints
│   │       ├── auth.py
│   │       ├── dashboard.py
│   │       ├── patients.py
│   │       ├── doctors.py
│   │       ├── appointments.py
│   │       ├── queue.py
│   │       ├── emergency.py
│   │       ├── search.py
│   │       ├── medical_history.py
│   │       ├── sorting.py
│   │       └── dsa_lab.py
│   ├── tests/                         # Pytest automated test suites
│   │   ├── test_dsa_queue.py
│   │   ├── test_dsa_priority_queue.py
│   │   ├── test_dsa_hash_table.py
│   │   ├── test_dsa_linked_list.py
│   │   ├── test_dsa_sorting.py
│   │   └── test_api_integration.py
│   ├── requirements.txt
│   └── run.py
├── frontend/
│   ├── src/
│   │   ├── components/                # Modular UI widgets
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   ├── StatCard.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Badge.tsx
│   │   │   └── EmptyState.tsx
│   │   ├── pages/                     # Full-featured application views
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Patients.tsx
│   │   │   ├── Doctors.tsx
│   │   │   ├── Appointments.tsx
│   │   │   ├── WaitingQueue.tsx
│   │   │   ├── EmergencyTriage.tsx
│   │   │   ├── PatientSearch.tsx
│   │   │   ├── MedicalHistory.tsx
│   │   │   ├── SortingAnalytics.tsx
│   │   │   ├── DsaLab.tsx
│   │   │   └── Login.tsx
│   │   ├── services/
│   │   │   └── api.ts                 # Typed fetch client
│   │   ├── types/
│   │   │   └── index.ts               # TypeScript interfaces
│   │   ├── App.tsx                    # Root container
│   │   ├── main.tsx                   # React DOM entry
│   │   └── index.css                  # Tailwind styles
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
```

---

## 7. Testing & Verification Summary

The project includes an extensive test suite executed via `pytest`:
- **DSA Unit Tests:** 13 unit tests covering FIFO queue operations, priority queue max-heap invariants, tie-breaking FIFO order, Floyd's build_heap, separate chaining collision resolution, singly linked list pointers, and manual merge/insertion sort edge cases.
- **API Integration Tests:** 10 end-to-end integration tests verifying FastAPI endpoints, conflict detection on doctor scheduling, CRUD cycles, search tracing, and laboratory sandboxes.
- **Test Result:** `23 passed in 1.71s` (100% pass rate).

