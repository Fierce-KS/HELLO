# UniGo Technical Requirements Document (TRD)

Project: UniGo - Student Ride-Pooling and InterCity Travel Coordination System

Course: UCS310 Database Management Systems

Institute: Thapar Institute of Engineering and Technology, Patiala

Academic Year: 2025-26

Date: 29 Apr 2026

Target DBMS: Oracle Database with PL/SQL

## 1. Purpose

This document defines the technical requirements for UniGo, a DBMS-centered system that coordinates student ride-pooling with strict database-level enforcement of business rules, concurrency control, and auditability. It is intended for database design, implementation, testing, and demonstration in a DBMS course setting.

## 2. Scope

In scope:

- Relational database schema for 9 core entities and their relationships.
- Integrity constraints, triggers, views, procedures, functions, and cursors in PL/SQL.
- Matching logic based on route compatibility, capacity, and trust thresholds.
- Transaction and concurrency control for group booking.
- Minimal frontend for demonstration and data entry only.

Out of scope:

- Full-featured mobile app or production-grade web UI.
- External services such as payment gateways, maps, or notifications.
- Non-relational databases, ORMs, or auto-SQL tools.
- Advanced route optimization beyond the defined Routes table.

## 3. Stakeholders and User Roles

- Student: creates trip requests, joins ride groups, and pays fare.
- Admin: manages destinations, routes, and vehicles.
- System (automated): runs matching logic, validations, and safety checks.
- Instructors and evaluators: review schema, queries, and PL/SQL correctness.

## 4. Assumptions and Constraints

- Oracle Database is the target DBMS, and PL/SQL is mandatory.
- All critical business rules are enforced in the database layer via constraints and triggers.
- The system is relational-only. No NoSQL stores or file-based persistence.
- Concurrency control must use row-level locks with `SELECT FOR UPDATE NOWAIT`.
- All modules are designed as independent units with ownership set to TBD.

## 5. Modular Architecture (Team-Oriented)

| Module | Purpose | Key DB Objects | Inputs | Outputs | Dependencies | Ownership |
| --- | --- | --- | --- | --- | --- | --- |
| User Management | Student registration, profile, trust score maintenance | `Students`, `Trust_Audit_Log`, `trg_update_trust` | student profile data | student records, trust events | none | TBD |
| Matching Engine | Find compatible ride groups | `vw_available_rides`, `Routes`, `find_travel_buddies` | `request_id`, destination, time window | candidate groups | User Management, Admin Data | TBD |
| Group Booking | Atomically join group with capacity validation | `Ride_Groups`, `Group_Members`, `group_lock_in` | `group_id`, `student_id`, luggage count | membership record, seat updates | Matching Engine | TBD |
| Reputation System | Immutable trust audit log | `Trust_Audit_Log`, `trg_update_trust` | ride completion and cancellation events | audit rows | User Management | TBD |
| Safety Validator | Validate connection time buffer | `calculate_travel_risk`, `Destinations` | `destination_id`, connection time | risk flag or warning | Matching Engine | TBD |
| Admin Reference Data | Manage destinations, routes, vehicles | `Destinations`, `Routes`, `Vehicles` | admin updates | reference records | none | TBD |
| Frontend (Minimal) | Demo-only UI for required flows | Consumes views and procedures | form inputs | screen output | all backend modules | TBD |

## 6. Functional Requirements

FR-01 The system shall store students with a unique email and a materialized trust score.

FR-02 The system shall allow a student to create a trip request with a destination and optional connection requirement.

FR-03 The system shall precompute availability using `vw_available_rides` with seats and luggage remaining.

FR-04 The system shall match rides by exact destination or route compatibility where `Routes.is_enroute = TRUE`.

FR-05 The system shall block group joins if seat capacity or luggage capacity would be exceeded.

FR-06 The system shall enforce a trust threshold of at least 90 when a trip request has `has_connection = TRUE`.

FR-07 The system shall prevent double booking by locking the ride group row with `SELECT FOR UPDATE NOWAIT`.

FR-08 The system shall use SAVEPOINT and partial rollback when a group join fails mid-transaction.

FR-09 The system shall write immutable trust changes to `Trust_Audit_Log` via trigger.

FR-10 The system shall compute travel risk using `calculate_travel_risk` based on destination travel time and connection buffer.

FR-11 The system shall prevent duplicate membership in a ride group via a composite primary key.

FR-12 The system shall allow admins to create and update destinations, routes, and vehicles.

FR-13 The system shall store payment status per group member.

FR-14 The system shall support a query to list students with no cancellations using `NOT EXISTS` on the audit log.

FR-15 The system shall provide a minimal UI to create students, submit requests, view matches, and join groups.

## 7. Data Model Summary

### 7.1 Entities and Key Attributes

- Students (`student_id` PK, `name`, `email` UNIQUE, `trust_score`, `total_rides`)
- Destinations (`destination_id` PK, `city_name` UNIQUE, `is_transit_hub`, `avg_travel_hours`)
- Routes (`route_id` PK, `from_dest_id` FK, `to_dest_id` FK, `is_enroute`, `distance`)
- Vehicles (`vehicle_id` PK, `vehicle_type`, `seat_capacity`, `max_large_bags`)
- Trip_Requests (`request_id` PK, `student_id` FK, `destination_id` FK, `has_connection`)
- Ride_Groups (`group_id` PK, `vehicle_id` FK, `destination_id` FK, `seats_filled`, `total_luggage`, `status`)
- Group_Members (`group_id` FK, `student_id` FK, composite PK, `payment_status`, `member_status`, `joined_at`)
- Payments (`payment_id` PK, `student_id` FK, `group_id` FK, `amount`, `status`)
- Trust_Audit_Log (`audit_id` PK, `student_id` FK, `old_score`, `new_score`, `reason`, `timestamp`)

### 7.2 Relationships

- Student creates Trip_Requests (1:N)
- Trip_Requests target Destinations (N:1)
- Ride_Groups target Destinations (N:1)
- Ride_Groups use Vehicles (N:1)
- Ride_Groups contain Group_Members (1:N)
- Students belong to Ride_Groups through Group_Members (M:N)
- Students generate Trust_Audit_Log entries (1:N)
- Destinations are connected via Routes (M:N)

### 7.3 Normalization Requirements

- All tables must be in 1NF, 2NF, and 3NF.
- Composite keys must not introduce partial dependencies.
- No transitive dependencies for non-key attributes.
- Target BCNF where practical for all 9 tables.

## 8. Business Rules and Constraints

- `Students.email` must be unique and non-null.
- `Destinations.city_name` must be unique and non-null.
- `Ride_Groups.seats_filled` must be between 0 and `Vehicles.seat_capacity`.
- `Ride_Groups.total_luggage` must be between 0 and `Vehicles.max_large_bags`.
- `Group_Members.member_status` must be one of `JOINED`, `CANCELLED`, `COMPLETED`.
- `Trip_Requests.has_connection = TRUE` requires `Students.trust_score >= 90`.
- Route compatibility allows matching when `Routes.is_enroute = TRUE` or destination matches exactly.
- `Trust_Audit_Log` entries are append-only and cannot be updated or deleted.

## 9. Database Objects and PL/SQL Design

### 9.1 View

`vw_available_rides` provides available groups with remaining seat and luggage capacity.

```sql
CREATE OR REPLACE VIEW vw_available_rides AS
SELECT
  rg.group_id,
  rg.destination_id,
  v.seat_capacity - rg.seats_filled AS seats_available,
  v.max_large_bags - rg.total_luggage AS luggage_available,
  rg.status
FROM Ride_Groups rg
JOIN Vehicles v ON v.vehicle_id = rg.vehicle_id
WHERE rg.status = 'OPEN';
```

### 9.2 Procedures

`find_travel_buddies` returns candidate groups for a request, using view and route logic.

```sql
PROCEDURE find_travel_buddies(
  p_request_id IN NUMBER,
  p_results OUT SYS_REFCURSOR
);
```

`group_lock_in` performs an atomic join with capacity checks and locking.

```sql
PROCEDURE group_lock_in(
  p_group_id IN NUMBER,
  p_student_id IN NUMBER,
  p_luggage_count IN NUMBER,
  p_payment_status IN VARCHAR2
);
```

### 9.3 Function

`calculate_travel_risk` validates the time buffer for connecting travel.

```sql
FUNCTION calculate_travel_risk(
  p_destination_id IN NUMBER,
  p_connection_minutes IN NUMBER
) RETURN VARCHAR2;
```

Return values can be `SAFE`, `RISKY`, or `BLOCKED` based on buffer comparison.

### 9.4 Triggers

`trg_update_trust` writes to `Trust_Audit_Log` when a member completes or cancels.

```sql
CREATE OR REPLACE TRIGGER trg_update_trust
AFTER UPDATE OF member_status ON Group_Members
FOR EACH ROW
WHEN (NEW.member_status IN ('CANCELLED','COMPLETED'))
BEGIN
  -- Insert audit log and update Students.trust_score
END;
```

An additional trigger can enforce PriorityGo trust threshold at insert time.

```sql
CREATE OR REPLACE TRIGGER trg_prioritygo_trust
BEFORE INSERT ON Trip_Requests
FOR EACH ROW
BEGIN
  -- Validate trust_score when has_connection = TRUE
END;
```

### 9.5 Cursor Usage

`find_travel_buddies` must demonstrate cursor usage when iterating candidates.

```sql
CURSOR cur_candidate_rides IS
  SELECT * FROM vw_available_rides WHERE destination_id = :dest_id;
```

## 10. Transaction and Concurrency Requirements

All group joins must be atomic and protected by row-level locks. The recommended pattern is:

```sql
BEGIN
  SELECT * FROM Ride_Groups
  WHERE group_id = :p_group_id
  FOR UPDATE NOWAIT;

  SAVEPOINT member_add;

  INSERT INTO Group_Members (group_id, student_id, payment_status, member_status, joined_at)
  VALUES (:p_group_id, :p_student_id, :p_payment_status, 'JOINED', SYSTIMESTAMP);

  UPDATE Ride_Groups
  SET seats_filled = seats_filled + 1,
      total_luggage = total_luggage + :p_luggage
  WHERE group_id = :p_group_id;

  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK TO member_add;
    RAISE;
END;
```

## 11. Minimal Frontend Requirements

The frontend is a demo-only interface to exercise DBMS features.

- Student registration form connected to `Students`.
- Trip request form connected to `Trip_Requests`.
- Results page that displays `vw_available_rides` and route-compatible matches.
- Join group action that calls `group_lock_in`.
- Admin screens for destinations, routes, and vehicles.
- Audit view that reads `Trust_Audit_Log`.

No styling or advanced UX features are required beyond form input and list output.

## 12. Security and Access Control

- Use least-privilege roles for Student and Admin operations.
- Students interact via stored procedures and views, not direct table DML.
- Admin operations are restricted to reference data tables.
- Audit log tables are read-only for non-admin users.

## 13. Verification Checklist

- Schema includes all 9 entities with correct PK and FK constraints.
- `vw_available_rides` returns expected availability values.
- `find_travel_buddies` returns matches for destination and enroute routes.
- `group_lock_in` prevents overbooking under concurrency.
- Trust audit entries are created on cancellation and completion events.
- `calculate_travel_risk` warns or blocks insufficient connection buffers.
- All business rules are enforced without application-side logic.

## 14. Deliverables

- DDL scripts for tables, constraints, and indexes.
- DML scripts for seed data.
- PL/SQL scripts for views, procedures, functions, triggers, and cursors.
- This TRD document.
