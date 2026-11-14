# MediQueue DSA — Technical Specification & Academic Documentation

This document contains the complete technical architecture, database schemas, REST API documentation, algorithmic time & space complexity analysis, automated testing report, and full source code listings for all six manually implemented Data Structures and Algorithms.

---

## 1. System Architecture

MediQueue DSA implements an asynchronous, decoupled three-tier architecture:

```
┌────────────────────────────────────────────────────────┐
│             Presentation Tier (Frontend)               │
│   React 19 + TypeScript + Tailwind CSS v4 + Vite       │
│   Responsive Dashboard • DSA Live State Visualizers   │
└──────────────────────────┬─────────────────────────────┘
                           │ JSON over REST / Bearer JWT
                           ▼
┌────────────────────────────────────────────────────────┐
│             Application Tier (Backend)                 │
│   Python 3.11 + FastAPI Asynchronous Web Gateway       │
│   Input Validation (Pydantic) • Bcrypt • PyJWT Auth    │
└──────────────┬──────────────────────────┬──────────────┘
               │                          │
    Synchronized In-Memory                │ Persistent CRUD
               ▼                          ▼
┌──────────────────────────────┐ ┌──────────────────────┐
│  Six Custom DSA Engines      │ │ MongoDB Database     │
│  - FIFO Queue (DLL)          │ │ - patients           │
│  - Binary Max-Heap           │ │ - doctors            │
│  - Separate Chaining Hash    │ │ - appointments       │
│  - Singly Linked List        │ │ - waiting_queue      │
│  - Merge Sort (Manual)       │ │ - emergency_queue    │
│  - Insertion Sort (Manual)   │ │ - medical_history    │
│                              │ │ - users              │
└──────────────────────────────┘ └──────────────────────┘
```

### Architectural Principles:
1. **MongoDB is the Persistent Source of Truth:** Any demographic update, appointment booking, or medical history entry is written durably to MongoDB.
2. **In-Memory Algorithmic Acceleration:** For real-time outpatient queuing and emergency priority scheduling, active queues are maintained in specialized memory structures (`FIFOQueue`, `EmergencyPriorityQueue`, and `PatientHashTable`) synchronized during application lifespan.
3. **No Direct Built-In Shortcuts:** Built-in Python functions like `sorted()`, `list.sort()`, or `heapq` are not used in place of the custom DSA requirements.
4. **Idempotent Sorting:** Sorting algorithms operate on deep defensive copies of dataset records, guaranteeing that persistent storage order is never unintentionally altered.

---

## 2. Database Schema & Data Models

### 2.1 Collection: `patients`
Stores registered patient demographic records.
- **Index:** `patient_id` (Unique)
- **Fields:**
  - `patient_id` (`string`): Unique identifier (e.g. `P-1001`).
  - `name` (`string`): Full name.
  - `age` (`integer`): Patient age.
  - `gender` (`string`): Female, Male, or Other.
  - `contact` (`string`): Primary phone number.
  - `blood_group` (`string`): ABO/Rh blood type (e.g. `O+`, `A-`).
  - `address` (`string`): Physical street address.
  - `emergency_contact` (`string`): Next-of-kin contact number.
  - `registration_date` (`string`): ISO date (`YYYY-MM-DD`).
  - `is_deleted` (`boolean`): Soft-delete flag (default: `False`).

### 2.2 Collection: `doctors`
Stores physician staff profiles and schedules.
- **Index:** `doctor_id` (Unique)
- **Fields:**
  - `doctor_id` (`string`): Unique identifier (e.g. `DOC-01`).
  - `name` (`string`): Doctor's name (e.g. `Dr. Gregory House`).
  - `specialization` (`string`): Medical specialty.
  - `department` (`string`): Hospital division.
  - `contact` (`string`): Contact number.
  - `email` (`string`): Staff email address.
  - `room_number` (`string`): Consultation clinic office.
  - `available_days` (`list[string]`): Roster days (e.g. `["Monday", "Tuesday"]`).
  - `shift_hours` (`string`): Working shift (e.g. `09:00 - 17:00`).
  - `is_available` (`boolean`): Active on-duty status.

### 2.3 Collection: `appointments`
Tracks scheduled physician consultations with conflict validation.
- **Index:** `appointment_id` (Unique)
- **Compound Check:** `(doctor_id, appointment_date, appointment_time)` prevents double-booking.
- **Fields:**
  - `appointment_id` (`string`): Unique identifier (e.g. `APT-101`).
  - `patient_id` (`string`): Reference to `patients.patient_id`.
  - `patient_name` (`string`): Denormalized patient name.
  - `doctor_id` (`string`): Reference to `doctors.doctor_id`.
  - `doctor_name` (`string`): Denormalized doctor name.
  - `appointment_date` (`string`): Date (`YYYY-MM-DD`).
  - `appointment_time` (`string`): Time slot (`HH:MM`).
  - `reason` (`string`): Clinical reason for visit.
  - `status` (`string`): `Scheduled`, `Completed`, or `Cancelled`.
  - `created_at` (`string`): ISO timestamp.

### 2.4 Collection: `waiting_queue`
Persists regular outpatient waiting room states.
- **Fields:** `queue_id`, `patient_id`, `patient_name`, `doctor_id`, `doctor_name`, `visit_reason`, `enqueued_at`, `status` (`waiting` or `completed`).

### 2.5 Collection: `emergency_queue`
Persists emergency admissions for triage simulation.
- **Fields:** `emergency_id`, `patient_id`, `patient_name`, `chief_complaint`, `priority_score` (1-5), `vital_signs`, `triage_nurse`, `sequence`, `status`.

### 2.6 Collection: `medical_history`
Stores longitudinal consultation records reconstructed into a Singly Linked List.
- **Index:** `visit_id` (Unique)
- **Fields:** `visit_id`, `patient_id`, `date`, `doctor_id`, `doctor_name`, `diagnosis`, `prescription`, `notes`, `next_appointment`.

### 2.7 Collection: `users`
System staff accounts with bcrypt password hashes.
- **Index:** `username` (Unique)
- **Fields:** `username`, `hashed_password`, `name`, `role` (`admin`, `doctor`, `nurse`, `staff`).

---

## 3. REST API Endpoint Reference

| Method | Endpoint | Description | Status Codes |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/health` | Service health check and DSA module list | `200` |
| `POST` | `/api/auth/login` | User login returning JWT Bearer token | `200`, `401` |
| `GET` | `/api/auth/me` | Current authenticated user profile | `200`, `401` |
| `GET` | `/api/dashboard/stats` | Dynamic KPI counts, active heads, recent appts | `200` |
| `GET` | `/api/patients` | List active patient records | `200` |
| `POST` | `/api/patients` | Register new patient & sync Hash Table | `201`, `400` |
| `GET` | `/api/patients/{id}` | Lookup patient (Hash Table first, DB fallback) | `200`, `404` |
| `PUT` | `/api/patients/{id}` | Update patient details & update Hash Table | `200`, `404` |
| `DELETE` | `/api/patients/{id}` | Soft delete patient & remove from Hash Table | `200`, `404` |
| `GET` | `/api/doctors` | List doctors in hospital directory | `200` |
| `POST` | `/api/doctors` | Add new doctor profile | `201` |
| `PUT` | `/api/doctors/{id}` | Update doctor profile or toggle availability | `200`, `404` |
| `DELETE` | `/api/doctors/{id}` | Remove doctor profile | `200`, `404` |
| `GET` | `/api/appointments` | List appointments sorted by date | `200` |
| `POST` | `/api/appointments` | Book appointment with conflict check | `201`, `404`, `409` |
| `PUT` | `/api/appointments/{id}`| Reschedule or update appointment | `200`, `404`, `409` |
| `DELETE` | `/api/appointments/{id}`| Cancel appointment | `200`, `404` |
| `GET` | `/api/queue` | View active FIFO Waiting Queue roster | `200` |
| `POST` | `/api/queue/join` | Enqueue patient to rear of FIFO queue | `201`, `400`, `404` |
| `POST` | `/api/queue/serve-next` | Dequeue front patient from FIFO queue | `200`, `404` |
| `GET` | `/api/emergency` | View Priority Ranked Emergency Heap | `200` |
| `POST` | `/api/emergency/admit` | Insert emergency patient into Max-Heap | `201`, `404` |
| `POST` | `/api/emergency/dispatch-highest-priority` | Extract max priority patient from Heap | `200`, `404` |
| `GET` | `/api/search/hash-lookup/{id}` | O(1) patient lookup with search step trace | `200`, `404` |
| `GET` | `/api/search/hash-table-diagnostics` | View all bucket arrays & collision chains | `200` |
| `GET` | `/api/medical-history/{patient_id}` | Reconstruct & traverse Singly Linked List | `200`, `404` |
| `POST` | `/api/medical-history` | Append or prepend consultation node | `201`, `404` |
| `DELETE` | `/api/medical-history/{visit_id}` | Delete visit node & rewire list pointers | `200`, `404` |
| `GET` | `/api/sorting/patients` | Execute Merge Sort or Insertion Sort | `200`, `400` |
| `GET` | `/api/sorting/benchmark-comparison` | Side-by-side empirical benchmark | `200` |
| `GET` | `/api/dsa-lab/complexity-matrix` | Complete theoretical DSA complexity matrix | `200` |
| `POST` | `/api/dsa-lab/queue/enqueue` | Interactive Queue sandbox enqueue | `200` |
| `POST` | `/api/dsa-lab/queue/dequeue` | Interactive Queue sandbox dequeue | `200` |
| `POST` | `/api/dsa-lab/priority-queue/insert` | Interactive Max-Heap insertion | `200` |
| `POST` | `/api/dsa-lab/priority-queue/extract-max` | Interactive Max-Heap extraction | `200` |
| `POST` | `/api/dsa-lab/hash-table/insert` | Interactive Hash Table key-value insert | `200` |
| `GET` | `/api/dsa-lab/hash-table/search/{key}` | Interactive Hash Table search step trace | `200` |
| `POST` | `/api/dsa-lab/linked-list/append` | Interactive Linked List tail append | `200` |
| `POST` | `/api/dsa-lab/linked-list/prepend` | Interactive Linked List head prepend | `200` |
| `POST` | `/api/dsa-lab/sorting/merge-sort` | Interactive Merge Sort sandbox | `200` |
| `POST` | `/api/dsa-lab/sorting/insertion-sort` | Interactive Insertion Sort sandbox | `200` |

---

## 4. Time & Space Complexity Master Matrix

| Data Structure / Algorithm | Practical Hospital Application | Time: Best | Time: Average | Time: Worst | Space: Auxiliary | Structural Properties |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **FIFO Queue (Doubly Linked Nodes)** | Regular Outpatient Waiting Room Queue | $O(1)$ | $O(1)$ | $O(1)$ | $O(n)$ | First-In, First-Out; head & tail pointers guarantee pure $O(1)$ enqueue and dequeue without list shifts. |
| **Priority Queue (Binary Max-Heap)** | Emergency Trauma Triage Simulation | $O(1)$ peek | $O(\log n)$ | $O(\log n)$ | $O(n)$ | Complete binary tree in array; parents $\ge$ children; FIFO tie-breaking preserves arrival order on equal priorities. |
| **Hash Table (Separate Chaining)** | Instant Patient Record Lookup by Patient ID | $O(1)$ | $O(1)$ | $O(n)$ | $O(n + k)$ | Horner's polynomial rolling hash (prime multiplier 31); linked collision chains; auto-rehash at load factor $> 0.75$. |
| **Singly Linked List** | Chronological Patient Medical History Chain | $O(1)$ front/back | $O(n)$ traversal | $O(n)$ delete | $O(n)$ | Dynamic node pointers (`next`); natural temporal sequence without preallocated array constraints. |
| **Merge Sort** | Deterministic Sorting of Patient Registries | $O(n \log n)$ | $O(n \log n)$ | $O(n \log n)$ | $O(n)$ | Divide-and-conquer; strictly manual merging; stable relative order preservation. |
| **Insertion Sort** | Adaptive In-Place Sorting of Small / Streaming Batches | $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | In-place shifting of sorted prefix; stable; minimal overhead on nearly-sorted cohorts. |

---

## 5. Automated Testing Report

All test suites were executed using `pytest` on Python 3.11:

```
============================= test session starts =============================
platform win32 -- Python 3.11.4, pytest-9.2.0, pluggy-1.7.0
rootdir: C:\Users\hjiya\OneDrive\Desktop\MediQueue\backend
plugins: anyio-4.15.1
collected 23 items

tests/test_api_integration.py::test_health_endpoint PASSED               [  4%]
tests/test_api_integration.py::test_dashboard_stats PASSED               [  8%]
tests/test_api_integration.py::test_patient_crud_and_hash_table_sync PASSED [ 13%]
tests/test_api_integration.py::test_doctor_crud PASSED                   [ 17%]
tests/test_api_integration.py::test_appointment_booking_and_conflict_validation PASSED [ 21%]
tests/test_api_integration.py::test_waiting_queue_fifo_flow PASSED       [ 26%]
tests/test_api_integration.py::test_emergency_priority_heap_flow PASSED  [ 30%]
tests/test_api_integration.py::test_medical_history_linked_list PASSED   [ 34%]
tests/test_api_integration.py::test_sorting_endpoints PASSED             [ 39%]
tests/test_api_integration.py::test_dsa_lab_endpoints PASSED             [ 43%]
tests/test_dsa_hash_table.py::test_hash_table_crud PASSED                [ 47%]
tests/test_dsa_hash_table.py::test_hash_table_collision_handling PASSED  [ 52%]
tests/test_dsa_linked_list.py::test_linked_list_operations PASSED        [ 56%]
tests/test_dsa_priority_queue.py::test_priority_queue_ordering PASSED    [ 60%]
tests/test_dsa_priority_queue.py::test_priority_queue_equal_priority_arrival_order PASSED [ 65%]
tests/test_dsa_priority_queue.py::test_priority_queue_build_heap PASSED  [ 69%]
tests/test_dsa_queue.py::test_fifo_queue_operations PASSED               [ 73%]
tests/test_dsa_sorting.py::test_manual_merge_sort_by_name PASSED         [ 78%]
tests/test_dsa_sorting.py::test_manual_merge_sort_by_age PASSED          [ 82%]
tests/test_dsa_sorting.py::test_manual_merge_sort_by_registration_date PASSED [ 86%]
tests/test_dsa_sorting.py::test_manual_insertion_sort_by_name PASSED     [ 91%]
tests/test_dsa_sorting.py::test_manual_insertion_sort_by_age PASSED      [ 95%]
tests/test_dsa_sorting.py::test_sorting_empty_and_single PASSED          [100%]

======================= 23 passed in 1.71s =======================
```

---

## 6. Complete Source Code Listings for All Six DSA Implementations

The following source code listings are extracted directly from `backend/app/dsa/`.

### 6.1 Data Structure A: FIFO Queue (`backend/app/dsa/queue.py`)

```python
"""
Queue Implementation for MediQueue DSA.
Data Structure: FIFO (First-In, First-Out) Queue using a Doubly-Linked Node Architecture.
Features:
- O(1) Enqueue (tail insertion)
- O(1) Dequeue (head removal)
- O(1) Peek / Front
- Size tracking and empty state checks
- Traversal / Display for visualization
"""

from typing import Any, Optional, List, Dict


class QueueNode:
    """A single node inside the FIFO Queue."""
    def __init__(self, data: Any):
        self.data: Any = data
        self.next: Optional['QueueNode'] = None
        self.prev: Optional['QueueNode'] = None

    def to_dict(self) -> Dict[str, Any]:
        if isinstance(self.data, dict):
            return self.data
        if hasattr(self.data, "to_dict"):
            return self.data.to_dict()
        return {"data": self.data}


class FIFOQueue:
    """
    Custom FIFO Queue implementation with head and tail pointers.
    Guarantees pure O(1) enqueue and O(1) dequeue time complexity.
    """
    def __init__(self):
        self._head: Optional[QueueNode] = None
        self._tail: Optional[QueueNode] = None
        self._size: int = 0

    def enqueue(self, item: Any) -> int:
        """
        Appends an element to the rear of the queue.
        Time Complexity: O(1)
        """
        new_node = QueueNode(item)
        if self.is_empty():
            self._head = new_node
            self._tail = new_node
        else:
            new_node.prev = self._tail
            if self._tail:
                self._tail.next = new_node
            self._tail = new_node
        self._size += 1
        return self._size

    def dequeue(self) -> Optional[Any]:
        """
        Removes and returns the front element from the queue.
        Time Complexity: O(1)
        """
        if self.is_empty():
            return None
        
        dequeued_node = self._head
        self._head = self._head.next
        if self._head:
            self._head.prev = None
        else:
            self._tail = None
            
        self._size -= 1
        return dequeued_node.data if dequeued_node else None

    def peek(self) -> Optional[Any]:
        """
        Inspects the front element without removing it.
        Time Complexity: O(1)
        """
        if self.is_empty():
            return None
        return self._head.data if self._head else None

    def is_empty(self) -> bool:
        """Checks whether the queue contains any elements. O(1)."""
        return self._size == 0

    def size(self) -> int:
        """Returns the current number of elements in the queue. O(1)."""
        return self._size

    def display(self) -> List[Any]:
        """
        Traverses the queue from front to rear and returns elements as a list.
        Time Complexity: O(n)
        """
        result = []
        current = self._head
        position = 1
        while current is not None:
            node_data = current.data
            if isinstance(node_data, dict):
                node_data_copy = dict(node_data)
                node_data_copy["queue_position"] = position
                result.append(node_data_copy)
            else:
                result.append(node_data)
            current = current.next
            position += 1
        return result

    def clear(self) -> None:
        """Clears all elements from the queue."""
        self._head = None
        self._tail = None
        self._size = 0
```

---

### 6.2 Data Structure B: Priority Queue using Binary Max-Heap (`backend/app/dsa/priority_queue.py`)

```python
"""
Priority Queue Implementation using Binary Max-Heap for MediQueue DSA.
Used for: Emergency Priority Patient Simulation.

ACADEMIC DISCLAIMER:
This priority queue is an illustrative academic simulation of priority scheduling
using a Binary Max-Heap data structure. It is NOT a medically validated clinical
triage algorithm (e.g., ESI or Manchester Triage System) and must not be used
for real-world clinical decision-making.

Algorithm & Design:
- Binary Max-Heap stored as an array (list).
- Parent-child index relationships:
    parent(i) = (i - 1) // 2
    left(i)   = 2 * i + 1
    right(i)  = 2 * i + 2
- Elements have:
    - priority: integer (higher number = higher urgency)
    - sequence: integer auto-increment arrival counter (tie-breaker for FIFO stability)
    - data: patient/triage payload
- Order comparison:
    item_A > item_B iff:
      (item_A.priority > item_B.priority) or
      (item_A.priority == item_B.priority and item_A.sequence < item_B.sequence)
- Time Complexity:
    - insert: O(log n)
    - extract_max: O(log n)
    - peek: O(1)
    - build_heap: O(n) Floyd's bottom-up construction
"""

from typing import Any, Dict, List, Optional
import time


class HeapElement:
    """Represents a node in the priority queue with priority and arrival sequence."""
    def __init__(self, priority: int, sequence: int, data: Any):
        self.priority: int = priority
        self.sequence: int = sequence  # Lower sequence number means earlier arrival
        self.data: Any = data
        self.created_at: float = time.time()

    def __lt__(self, other: 'HeapElement') -> bool:
        if self.priority != other.priority:
            return self.priority < other.priority
        return self.sequence > other.sequence

    def __gt__(self, other: 'HeapElement') -> bool:
        if self.priority != other.priority:
            return self.priority > other.priority
        return self.sequence < other.sequence

    def __eq__(self, other: 'HeapElement') -> bool:
        return self.priority == other.priority and self.sequence == other.sequence

    def to_dict(self) -> Dict[str, Any]:
        return {
            "priority": self.priority,
            "sequence": self.sequence,
            "data": self.data
        }


class EmergencyPriorityQueue:
    """
    Binary Max-Heap Priority Queue.
    Includes explicit heapify_up, heapify_down, insert, extract_max, peek, and build_heap.
    """
    def __init__(self):
        self._heap: List[HeapElement] = []
        self._sequence_counter: int = 0

    @staticmethod
    def _parent(i: int) -> int:
        return (i - 1) // 2

    @staticmethod
    def _left_child(i: int) -> int:
        return 2 * i + 1

    @staticmethod
    def _right_child(i: int) -> int:
        return 2 * i + 2

    def heapify_up(self, index: int) -> None:
        """
        Restores max-heap property upwards from given index.
        Time Complexity: O(log n)
        """
        while index > 0:
            parent_idx = self._parent(index)
            if self._heap[index] > self._heap[parent_idx]:
                self._heap[index], self._heap[parent_idx] = self._heap[parent_idx], self._heap[index]
                index = parent_idx
            else:
                break

    def heapify_down(self, index: int) -> None:
        """
        Restores max-heap property downwards from given index.
        Time Complexity: O(log n)
        """
        size = len(self._heap)
        while True:
            largest = index
            left = self._left_child(index)
            right = self._right_child(index)

            if left < size and self._heap[left] > self._heap[largest]:
                largest = left

            if right < size and self._heap[right] > self._heap[largest]:
                largest = right

            if largest != index:
                self._heap[index], self._heap[largest] = self._heap[largest], self._heap[index]
                index = largest
            else:
                break

    def insert(self, data: Any, priority: int) -> int:
        """
        Inserts a new item with given priority into the heap.
        Time Complexity: O(log n)
        """
        self._sequence_counter += 1
        new_element = HeapElement(priority=priority, sequence=self._sequence_counter, data=data)
        self._heap.append(new_element)
        self.heapify_up(len(self._heap) - 1)
        return len(self._heap)

    def extract_max(self) -> Optional[Dict[str, Any]]:
        """
        Removes and returns the element with the highest priority.
        Maintains arrival order for equal priorities.
        Time Complexity: O(log n)
        """
        if self.is_empty():
            return None

        max_elem = self._heap[0]
        last_elem = self._heap.pop()

        if len(self._heap) > 0:
            self._heap[0] = last_elem
            self.heapify_down(0)

        return {
            "priority": max_elem.priority,
            "sequence": max_elem.sequence,
            "data": max_elem.data
        }

    def peek(self) -> Optional[Dict[str, Any]]:
        """
        Inspects the maximum element without removal.
        Time Complexity: O(1)
        """
        if self.is_empty():
            return None
        max_elem = self._heap[0]
        return {
            "priority": max_elem.priority,
            "sequence": max_elem.sequence,
            "data": max_elem.data
        }

    def build_heap(self, items: List[Dict[str, Any]]) -> None:
        """
        Constructs a max-heap from an unsorted list in O(n) time using Floyd's algorithm.
        Each item is expected to have 'data' and 'priority' keys.
        """
        self._heap = []
        for item in items:
            self._sequence_counter += 1
            priority = item.get("priority", 1)
            data = item.get("data", item)
            seq = item.get("sequence", self._sequence_counter)
            self._heap.append(HeapElement(priority=priority, sequence=seq, data=data))

        for i in range((len(self._heap) // 2) - 1, -1, -1):
            self.heapify_down(i)

    def is_empty(self) -> bool:
        return len(self._heap) == 0

    def size(self) -> int:
        return len(self._heap)

    def display(self) -> List[Dict[str, Any]]:
        """
        Returns a copy of elements sorted by priority order (without mutating heap).
        Demonstrates the exact extraction order.
        """
        temp_queue = EmergencyPriorityQueue()
        temp_queue._heap = list(self._heap)
        temp_queue._sequence_counter = self._sequence_counter

        ordered_list = []
        rank = 1
        while not temp_queue.is_empty():
            item = temp_queue.extract_max()
            if item:
                item["triage_rank"] = rank
                ordered_list.append(item)
                rank += 1
        return ordered_list

    def get_raw_tree(self) -> List[Dict[str, Any]]:
        """Returns internal array representation of the binary heap."""
        return [
            {
                "index": i,
                "priority": elem.priority,
                "sequence": elem.sequence,
                "data": elem.data,
                "parent_index": self._parent(i) if i > 0 else None,
                "left_index": self._left_child(i) if self._left_child(i) < len(self._heap) else None,
                "right_index": self._right_child(i) if self._right_child(i) < len(self._heap) else None,
            }
            for i, elem in enumerate(self._heap)
        ]

    def clear(self) -> None:
        self._heap = []
        self._sequence_counter = 0
```

---

### 6.3 Data Structure C: Separate Chaining Hash Table (`backend/app/dsa/hash_table.py`)

```python
"""
Hash Table Implementation using Separate Chaining for MediQueue DSA.
Used for: Patient Record Lookup & Indexing by Patient ID.

Features:
- Custom polynomial rolling hash / Horner's rule hash function.
- Separate chaining collision resolution (singly linked bucket chains).
- Manual insert, search, delete, display, and resize/rehash.
- Collision counter and chain length diagnostics for academic demonstration.
- Average Case Complexity: O(1) lookup, insertion, and deletion.
- Worst Case Complexity: O(n) when all keys hash to the same bucket.
"""

from typing import Any, Dict, List, Optional, Tuple


class HashNode:
    """A node in a separate-chaining bucket collision list."""
    def __init__(self, key: str, value: Any):
        self.key: str = key
        self.value: Any = value
        self.next: Optional['HashNode'] = None


class PatientHashTable:
    """
    Fixed or dynamic size Hash Table with Separate Chaining.
    Bucket array where each bucket holds a linked list of HashNodes.
    """
    def __init__(self, initial_capacity: int = 17):
        self._capacity: int = max(7, initial_capacity)
        self._buckets: List[Optional[HashNode]] = [None] * self._capacity
        self._size: int = 0
        self._total_collisions: int = 0

    def hash_function(self, key: str) -> int:
        """
        Computes the bucket index using polynomial rolling hash (Horner's method).
        hash = (sum(char_code * 31^(len - 1 - i))) % capacity
        Time Complexity: O(k) where k is key length.
        """
        hash_val = 0
        prime = 31
        for char in str(key):
            hash_val = (hash_val * prime + ord(char)) % self._capacity
        return hash_val

    def insert(self, key: str, value: Any) -> Tuple[int, bool]:
        """
        Inserts or updates a key-value pair.
        Returns (bucket_index, was_collision).
        Time Complexity: Average O(1), Worst O(n).
        """
        index = self.hash_function(key)
        head = self._buckets[index]
        was_collision = False

        if head is None:
            self._buckets[index] = HashNode(key, value)
            self._size += 1
            return index, False

        current = head
        while current is not None:
            if current.key == key:
                current.value = value
                return index, False
            if current.next is None:
                break
            current = current.next

        current.next = HashNode(key, value)
        self._size += 1
        self._total_collisions += 1
        was_collision = True

        if self.load_factor() > 0.75:
            self._rehash(self._capacity * 2 + 1)

        return index, was_collision

    def search(self, key: str) -> Tuple[Optional[Any], Dict[str, Any]]:
        """
        Searches for a patient by key.
        Returns (value, search_trace) with diagnostic steps for the DSA Lab.
        Time Complexity: Average O(1), Worst O(n).
        """
        index = self.hash_function(key)
        steps = 0
        current = self._buckets[index]
        chain_traversed = []

        while current is not None:
            steps += 1
            chain_traversed.append(current.key)
            if current.key == key:
                return current.value, {
                    "found": True,
                    "bucket_index": index,
                    "steps_taken": steps,
                    "chain_keys_checked": chain_traversed,
                    "time_complexity_observed": f"{steps} hop(s)"
                }
            current = current.next

        return None, {
            "found": False,
            "bucket_index": index,
            "steps_taken": steps,
            "chain_keys_checked": chain_traversed,
            "time_complexity_observed": f"{steps} hop(s)"
        }

    def delete(self, key: str) -> bool:
        """
        Removes a key-value pair from the hash table.
        Returns True if found and deleted, False otherwise.
        Time Complexity: Average O(1), Worst O(n).
        """
        index = self.hash_function(key)
        current = self._buckets[index]
        prev = None

        while current is not None:
            if current.key == key:
                if prev is None:
                    self._buckets[index] = current.next
                else:
                    prev.next = current.next
                self._size -= 1
                return True
            prev = current
            current = current.next

        return False

    def load_factor(self) -> float:
        return self._size / self._capacity if self._capacity > 0 else 0.0

    def _rehash(self, new_capacity: int) -> None:
        old_buckets = self._buckets
        self._capacity = new_capacity
        self._buckets = [None] * new_capacity
        self._size = 0
        self._total_collisions = 0

        for head in old_buckets:
            current = head
            while current is not None:
                self.insert(current.key, current.value)
                current = current.next

    def display(self) -> List[Dict[str, Any]]:
        table_state = []
        for i in range(self._capacity):
            chain = []
            current = self._buckets[i]
            while current is not None:
                chain.append({
                    "key": current.key,
                    "value": current.value
                })
                current = current.next
            table_state.append({
                "bucket_index": i,
                "chain_length": len(chain),
                "is_collision": len(chain) > 1,
                "entries": chain
            })
        return table_state

    def get_all_records(self) -> List[Any]:
        records = []
        for head in self._buckets:
            current = head
            while current is not None:
                records.append(current.value)
                current = current.next
        return records

    def statistics(self) -> Dict[str, Any]:
        occupied_buckets = sum(1 for b in self._buckets if b is not None)
        max_chain = max((self._get_chain_length(b) for b in self._buckets), default=0)
        return {
            "capacity": self._capacity,
            "total_elements": self._size,
            "occupied_buckets": occupied_buckets,
            "empty_buckets": self._capacity - occupied_buckets,
            "load_factor": round(self.load_factor(), 4),
            "max_chain_length": max_chain,
            "total_collisions": self._total_collisions
        }

    @staticmethod
    def _get_chain_length(node: Optional[HashNode]) -> int:
        count = 0
        curr = node
        while curr:
            count += 1
            curr = curr.next
        return count

    def clear(self) -> None:
        self._buckets = [None] * self._capacity
        self._size = 0
        self._total_collisions = 0
```

---

### 6.4 Data Structure D: Singly Linked List (`backend/app/dsa/linked_list.py`)

```python
"""
Singly Linked List Implementation for MediQueue DSA.
Used for: Patient Medical History & Consultation Records.

Features:
- MedicalVisitNode holding comprehensive clinical visit details.
- Singly linked traversal with 'next' pointers.
- Operations:
    - append (add to tail) - O(1) with tail pointer
    - insert_at_beginning (prepend) - O(1)
    - search (by visit_id or diagnosis keyword) - O(n)
    - delete (by visit_id) - O(n)
    - to_list / traversal (chronological visit chain) - O(n)
    - size and is_empty - O(1)
- Reconstructed dynamically from MongoDB persistent medical records.
"""

from typing import Any, Dict, List, Optional


class MedicalVisitNode:
    """A single medical visit event node in the chronological linked list."""
    def __init__(
        self,
        visit_id: str,
        patient_id: str,
        date: str,
        doctor_id: str,
        doctor_name: str,
        diagnosis: str,
        prescription: str,
        notes: str,
        next_appointment: Optional[str] = None
    ):
        self.visit_id: str = visit_id
        self.patient_id: str = patient_id
        self.date: str = date
        self.doctor_id: str = doctor_id
        self.doctor_name: str = doctor_name
        self.diagnosis: str = diagnosis
        self.prescription: str = prescription
        self.notes: str = notes
        self.next_appointment: Optional[str] = next_appointment
        self.next: Optional['MedicalVisitNode'] = None

    def to_dict(self) -> Dict[str, Any]:
        return {
            "visit_id": self.visit_id,
            "patient_id": self.patient_id,
            "date": self.date,
            "doctor_id": self.doctor_id,
            "doctor_name": self.doctor_name,
            "diagnosis": self.diagnosis,
            "prescription": self.prescription,
            "notes": self.notes,
            "next_appointment": self.next_appointment
        }


class MedicalHistoryLinkedList:
    """
    Singly Linked List managing chronological medical consultation records.
    Maintains head and tail pointers for efficient front and back insertions.
    """
    def __init__(self, patient_id: str = ""):
        self.patient_id: str = patient_id
        self._head: Optional[MedicalVisitNode] = None
        self._tail: Optional[MedicalVisitNode] = None
        self._size: int = 0

    def insert_at_beginning(self, node_data: Dict[str, Any]) -> None:
        """
        Inserts a new visit at the head of the list (e.g. most recent first).
        Time Complexity: O(1)
        """
        new_node = MedicalVisitNode(
            visit_id=node_data.get("visit_id", ""),
            patient_id=node_data.get("patient_id", self.patient_id),
            date=node_data.get("date", ""),
            doctor_id=node_data.get("doctor_id", ""),
            doctor_name=node_data.get("doctor_name", ""),
            diagnosis=node_data.get("diagnosis", ""),
            prescription=node_data.get("prescription", ""),
            notes=node_data.get("notes", ""),
            next_appointment=node_data.get("next_appointment")
        )

        new_node.next = self._head
        self._head = new_node
        if self._tail is None:
            self._tail = new_node
        self._size += 1

    def append(self, node_data: Dict[str, Any]) -> None:
        """
        Appends a visit to the tail of the list (chronological progression).
        Time Complexity: O(1) with tail pointer.
        """
        new_node = MedicalVisitNode(
            visit_id=node_data.get("visit_id", ""),
            patient_id=node_data.get("patient_id", self.patient_id),
            date=node_data.get("date", ""),
            doctor_id=node_data.get("doctor_id", ""),
            doctor_name=node_data.get("doctor_name", ""),
            diagnosis=node_data.get("diagnosis", ""),
            prescription=node_data.get("prescription", ""),
            notes=node_data.get("notes", ""),
            next_appointment=node_data.get("next_appointment")
        )

        if self._head is None:
            self._head = new_node
            self._tail = new_node
        else:
            if self._tail:
                self._tail.next = new_node
            self._tail = new_node
        self._size += 1

    def search(self, visit_id: str) -> Optional[Dict[str, Any]]:
        """
        Linear search across linked nodes matching visit_id.
        Time Complexity: O(n)
        """
        current = self._head
        index = 0
        while current is not None:
            if current.visit_id == visit_id:
                result = current.to_dict()
                result["linked_list_position"] = index
                return result
            current = current.next
            index += 1
        return None

    def search_by_keyword(self, keyword: str) -> List[Dict[str, Any]]:
        matches = []
        kw = keyword.lower()
        current = self._head
        index = 0
        while current is not None:
            if kw in current.diagnosis.lower() or kw in current.notes.lower() or kw in current.prescription.lower():
                res = current.to_dict()
                res["linked_list_position"] = index
                matches.append(res)
            current = current.next
            index += 1
        return matches

    def delete(self, visit_id: str) -> bool:
        """
        Deletes a node with matching visit_id.
        Time Complexity: O(n)
        """
        current = self._head
        prev: Optional[MedicalVisitNode] = None

        while current is not None:
            if current.visit_id == visit_id:
                if prev is None:
                    self._head = current.next
                    if self._head is None:
                        self._tail = None
                else:
                    prev.next = current.next
                    if current == self._tail:
                        self._tail = prev
                self._size -= 1
                return True
            prev = current
            current = current.next

        return False

    def traversal(self) -> List[Dict[str, Any]]:
        """
        Iterates over the linked list from head to tail and returns list of visits.
        Time Complexity: O(n)
        """
        visits = []
        current = self._head
        index = 0
        while current is not None:
            v_dict = current.to_dict()
            v_dict["sequence_index"] = index
            v_dict["has_next"] = current.next is not None
            visits.append(v_dict)
            current = current.next
            index += 1
        return visits

    def size(self) -> int:
        return self._size

    def is_empty(self) -> bool:
        return self._size == 0

    @classmethod
    def from_list(cls, patient_id: str, records: List[Dict[str, Any]]) -> 'MedicalHistoryLinkedList':
        history = cls(patient_id=patient_id)
        for rec in records:
            history.append(rec)
        return history
```

---

### 6.5 Algorithm E: Manual Merge Sort (`backend/app/dsa/merge_sort.py`)

```python
"""
Manual Merge Sort Implementation for MediQueue DSA.
Used for: Sorting Patient Records by Name, Age, or Registration Date.

Algorithm: Divide and Conquer
1. Divide: Split the unsorted list into two approximately equal sublists.
2. Conquer: Recursively sort both sublists until sublist size is 1 or 0 (base case).
3. Combine: Merge the two sorted sublists back into one sorted list.

Guarantees:
- Stable sorting: preserves relative order of duplicate keys.
- Time Complexity:
    - Best Case: O(n log n)
    - Average Case: O(n log n)
    - Worst Case: O(n log n)
- Space Complexity: O(n) auxiliary space for sublist merging.

NOTE: Python's built-in sorted() and list.sort() are STRICTLY AVOIDED.
"""

from typing import Any, Callable, Dict, List, Tuple
import time


class MergeSortDiagnostics:
    """Tracks metrics during the merge sort execution for academic demonstration."""
    def __init__(self):
        self.comparisons: int = 0
        self.merge_operations: int = 0
        self.recursion_depth: int = 0
        self.max_depth: int = 0


def _extract_sort_key(record: Dict[str, Any], field: str) -> Any:
    """Safely extracts field value from dictionary for comparison."""
    val = record.get(field)
    if val is None:
        return ""
    if isinstance(val, (int, float)):
        return val
    if field == "age":
        try:
            return int(val)
        except (ValueError, TypeError):
            return 0
    if field == "name":
        return str(val).lower()
    return str(val)


def _merge(
    left: List[Dict[str, Any]],
    right: List[Dict[str, Any]],
    field: str,
    reverse: bool,
    metrics: MergeSortDiagnostics
) -> List[Dict[str, Any]]:
    """Combines two pre-sorted lists into a single sorted list."""
    merged: List[Dict[str, Any]] = []
    i = 0
    j = 0
    metrics.merge_operations += 1

    while i < len(left) and j < len(right):
        metrics.comparisons += 1
        key_left = _extract_sort_key(left[i], field)
        key_right = _extract_sort_key(right[j], field)

        if not reverse:
            condition = (key_left <= key_right)
        else:
            condition = (key_left >= key_right)

        if condition:
            merged.append(left[i])
            i += 1
        else:
            merged.append(right[j])
            j += 1

    while i < len(left):
        merged.append(left[i])
        i += 1

    while j < len(right):
        merged.append(right[j])
        j += 1

    return merged


def _merge_sort_recursive(
    items: List[Dict[str, Any]],
    field: str,
    reverse: bool,
    metrics: MergeSortDiagnostics,
    current_depth: int
) -> List[Dict[str, Any]]:
    if current_depth > metrics.max_depth:
        metrics.max_depth = current_depth

    if len(items) <= 1:
        return items

    mid = len(items) // 2
    left_half = items[:mid]
    right_half = items[mid:]

    sorted_left = _merge_sort_recursive(left_half, field, reverse, metrics, current_depth + 1)
    sorted_right = _merge_sort_recursive(right_half, field, reverse, metrics, current_depth + 1)

    return _merge(sorted_left, sorted_right, field, reverse, metrics)


def manual_merge_sort(
    records: List[Dict[str, Any]],
    field: str = "name",
    reverse: bool = False
) -> Tuple[List[Dict[str, Any]], Dict[str, Any]]:
    records_copy = [dict(r) for r in records]
    metrics = MergeSortDiagnostics()

    start_time = time.perf_counter()
    sorted_records = _merge_sort_recursive(records_copy, field, reverse, metrics, 1)
    end_time = time.perf_counter()

    duration_ms = (end_time - start_time) * 1000.0

    diagnostics = {
        "algorithm": "Merge Sort",
        "field": field,
        "direction": "descending" if reverse else "ascending",
        "element_count": len(records),
        "comparisons": metrics.comparisons,
        "merge_operations": metrics.merge_operations,
        "max_recursion_depth": metrics.max_depth,
        "execution_time_ms": round(duration_ms, 4),
        "time_complexity_theoretical": "O(n log n)",
        "space_complexity_theoretical": "O(n)",
        "stability": "Stable"
    }

    return sorted_records, diagnostics
```

---

### 6.6 Algorithm F: Manual Insertion Sort (`backend/app/dsa/insertion_sort.py`)

```python
"""
Manual Insertion Sort Implementation for MediQueue DSA.
Used for: Sorting Patient Records by Name, Age, or Registration Date.

Algorithm:
1. Divide list conceptually into a sorted prefix and unsorted suffix.
2. Pick next element (the 'key') from the unsorted portion.
3. Shift elements of the sorted portion that are greater than key to one position right.
4. Place key at its correct position.

Guarantees:
- In-place sorting (O(1) auxiliary space).
- Stable sorting.
- Time Complexity:
    - Best Case (Already Sorted): O(n) with 0 shifts and n-1 comparisons.
    - Average Case: O(n^2)
    - Worst Case (Reverse Sorted): O(n^2) with ~n^2/2 shifts and comparisons.
- Ideal for small arrays or incrementally incoming streams.

NOTE: Built-in sorted() and list.sort() are strictly avoided.
"""

from typing import Any, Dict, List, Tuple
import time


def _extract_sort_key(record: Dict[str, Any], field: str) -> Any:
    """Safely extracts field value from dictionary for comparison."""
    val = record.get(field)
    if val is None:
        return ""
    if isinstance(val, (int, float)):
        return val
    if field == "age":
        try:
            return int(val)
        except (ValueError, TypeError):
            return 0
    if field == "name":
        return str(val).lower()
    return str(val)


def manual_insertion_sort(
    records: List[Dict[str, Any]],
    field: str = "name",
    reverse: bool = False
) -> Tuple[List[Dict[str, Any]], Dict[str, Any]]:
    items = [dict(r) for r in records]
    n = len(items)

    comparisons = 0
    shifts = 0

    start_time = time.perf_counter()

    for i in range(1, n):
        key_record = items[i]
        key_val = _extract_sort_key(key_record, field)
        j = i - 1

        while j >= 0:
            comparisons += 1
            current_val = _extract_sort_key(items[j], field)

            if not reverse:
                should_shift = current_val > key_val
            else:
                should_shift = current_val < key_val

            if should_shift:
                items[j + 1] = items[j]
                shifts += 1
                j -= 1
            else:
                break

        items[j + 1] = key_record

    end_time = time.perf_counter()
    duration_ms = (end_time - start_time) * 1000.0

    diagnostics = {
        "algorithm": "Insertion Sort",
        "field": field,
        "direction": "descending" if reverse else "ascending",
        "element_count": n,
        "comparisons": comparisons,
        "shifts": shifts,
        "execution_time_ms": round(duration_ms, 4),
        "best_case_complexity": "O(n)",
        "average_worst_complexity": "O(n^2)",
        "space_complexity_theoretical": "O(1)",
        "stability": "Stable"
    }

    return items, diagnostics
```

