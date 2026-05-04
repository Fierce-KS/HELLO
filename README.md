# UniGo Repository Analysis & Viva Guide

This document breaks down the entire PL/SQL implementation of the UniGo system. It is designed to help you completely master the codebase for your final presentation and viva evaluation.

---

## 1. Stored Procedures
Stored procedures are used for complex transactions that require multiple steps, validations, and potential rollbacks.

### A. `book_ride` (Located in `C1_booking.sql`)
*   **Purpose:** Handles the entire logic of a student booking a seat in a ride group.
*   **How it works:** 
    1. It first attempts to lock the group row using `SELECT ... FOR UPDATE NOWAIT` to prevent race conditions.
    2. It creates a `SAVEPOINT`.
    3. It validates if the group has space and if the luggage fits.
    4. If the group destination is an airport/railway station, it enforces PriorityGo (trust score >= 90).
    5. It inserts the student into `Group_Members` and creates a `Payments` record.
    6. If any validation fails, it issues a `ROLLBACK TO SAVEPOINT`. Otherwise, it issues a `COMMIT`.
*   **Key Concept Demonstrated:** Transaction Management (ACID properties), Pessimistic Concurrency Control.

### B. `find_travel_buddies` (Located in `B1_views_and_proc.sql`)
*   **Purpose:** The core matching engine. Finds existing groups that a student can join.
*   **How it works:** Uses an **Explicit Cursor** (`c_matches`) with a `UNION ALL` statement to find both *Direct* matches (groups going to the exact same destination) and *En-Route* matches (groups passing through the student's destination).
*   **Key Concept Demonstrated:** Explicit Cursors, Complex joins, Subqueries.

### C. `cancel_booking` & `complete_ride` (Located in `C1_booking.sql`)
*   **Purpose:** Lifecycle management. 
    *   `cancel_booking` deletes a student from `Group_Members` (which fires a penalty trigger) and refunds their payment.
    *   `complete_ride` changes a group's status to 'COMPLETED' (which fires a reward trigger).

### D. `log_trust_change` (Located in `D2_function.sql`)
*   **Purpose:** A utility procedure called by triggers. It immutably inserts a record into the `Trust_Audit_Log` detailing exactly why a student's trust score changed.

---

## 2. Functions
Functions are designed to perform a calculation and return a specific value.

### A. `calculate_travel_risk` (Located in `D2_function.sql`)
*   **Purpose:** Evaluates if a student has enough time to catch their connecting flight/train.
*   **How it works:** It takes the student's available connection time and the destination ID. It looks up the `avg_travel_hours` for that destination, adds a 30% safety buffer (`buffer_factor = 1.3`), and compares it.
*   **Returns:** `'SAFE'`, `'RISKY'`, or `'CRITICAL'`.

### B. `check_priority_eligibility` (Located in `D2_function.sql`)
*   **Purpose:** Checks if a student has high enough trust to use the PriorityGo feature.
*   **Returns:** `'ELIGIBLE'` (if trust >= 90), else `'NOT_ELIGIBLE: XX'`.

---

## 3. Database Triggers
Triggers automatically execute in response to specific database events (INSERT, UPDATE, DELETE). They enforce business rules at the database level.

### A. `trg_seat_sync` (Located in `D1_triggers.sql`)
*   **Purpose:** Automatically updates the `seats_filled` and `total_luggage` columns in the `Ride_Groups` table.
*   **When it fires:** `AFTER INSERT OR DELETE ON Group_Members`.
*   **Special Logic:** It is built as a **Compound Trigger** to avoid the infamous `ORA-04091 Mutating Table` error in Oracle. It waits until the statement is finished before recalculating the totals from scratch.

### B. `trg_trust_on_completion` (Located in `D1_triggers.sql`)
*   **Purpose:** Rewards good behavior.
*   **When it fires:** `AFTER UPDATE OF status ON Ride_Groups`.
*   **Special Logic:** When the status changes to `'COMPLETED'`, it uses a Cursor to loop through every member of the group, adds +10 to their `trust_score`, increments their `total_rides`, and calls `log_trust_change`.

### C. `trg_trust_on_cancellation` (Located in `D1_triggers.sql`)
*   **Purpose:** Penalizes bad behavior (last-minute dropouts).
*   **When it fires:** `AFTER DELETE ON Group_Members`.
*   **Special Logic:** Deducts 15 points from the student's trust score (preventing it from dropping below 0) and calls `log_trust_change`.

### D. `trg_immutable_audit` (Located in `D1_triggers.sql`)
*   **Purpose:** Ensures the `Trust_Audit_Log` is completely tamper-proof.
*   **When it fires:** `BEFORE UPDATE OR DELETE ON Trust_Audit_Log`.
*   **Special Logic:** It instantly raises an `APPLICATION_ERROR`. No one, not even the Database Administrator, can alter or delete a trust audit log once it is written.

---

## 4. Cursors
Cursors are used to handle multi-row result sets by iterating through them one row at a time.

### A. `c_matches` (Inside `find_travel_buddies`)
*   **Type:** Explicit Cursor.
*   **Why used:** The matching engine might find 0, 1, or 50 potential ride groups for a student. A standard `SELECT INTO` would crash if it found multiple rows. The cursor allows the procedure to fetch and output each match line by line.

### B. `c_members` (Inside `trg_trust_on_completion`)
*   **Type:** Explicit Cursor.
*   **Why used:** When a ride finishes, the trigger must reward *every* student in that group. The cursor fetches all the `student_id`s tied to that `group_id` and processes the reward in a `FOR rec IN c_members LOOP`.

---

## 5. Potential Viva Questions & Answers

**Q1: What is the difference between your Functions and your Stored Procedures?**
*Answer:* Functions are strictly used to compute and return a value (like calculating the risk buffer or checking eligibility) and do not modify the database. Stored Procedures perform actions, manage transactions (`COMMIT`/`ROLLBACK`), and manipulate data (like booking a ride).

**Q2: How did you handle Concurrency when two students try to book the last seat at the exact same time?**
*Answer:* I used Pessimistic Locking. In the `book_ride` procedure, my first step is `SELECT ... FOR UPDATE NOWAIT`. This places an exclusive lock on the `Ride_Groups` row. If Student B tries to book while Student A's transaction is processing, the database instantly throws an `ORA-00054` error, completely preventing a double-booking.

**Q3: What are ACID properties and how did you implement them?**
*Answer:* 
*   **Atomicity:** I used `SAVEPOINT`. If luggage capacity fails during a booking, it rolls back to the savepoint so no partial inserts happen.
*   **Consistency:** Implemented via `CHECK` constraints (e.g., trust cannot be negative) and database triggers.
*   **Isolation:** Achieved using the `FOR UPDATE NOWAIT` row lock.
*   **Durability:** Ensured by the `COMMIT` command, which permanently writes the confirmed booking to disk.

**Q4: I see you used a Compound Trigger for `trg_seat_sync`. Why?**
*Answer:* In Oracle, if a row-level trigger tries to query the very table that caused the trigger to fire, it throws a "Mutating Table Error" (`ORA-04091`). Because I need to count the total members in `Group_Members` after inserting a new member, I used a Compound Trigger to wait until the `AFTER STATEMENT` phase, where the table is stable and safe to query.

**Q5: Why did you use Views instead of just querying the tables directly?**
*Answer:* Views provide abstraction and security. For example, my `vw_available_rides` view joins `Ride_Groups`, `Vehicles`, and `Destinations` while mathematically filtering out full groups and past dates. This means the frontend or matching engine can just do a simple `SELECT * FROM vw_available_rides` instead of writing a massive 4-table join every time.

**Q6: How does the system handle "En-Route" matching?**
*Answer:* I created a self-referencing `Routes` table that links a `from_dest_id` to a `to_dest_id`. My `vw_enroute_matches` view uses an `EXISTS` subquery to check if a forming ride group is passing through the student's desired destination as a layover.

**Q7: Explain how the Trust Audit Log is immutable.**
*Answer:* I wrote a trigger (`trg_immutable_audit`) that fires `BEFORE UPDATE OR DELETE` on the `Trust_Audit_Log` table. It simply raises an application error. This means the table is strictly append-only; past records can never be modified.
