-- ===========================================================
-- STUDENT ENROLLMENTS CONCURRENCY CONTROL DEMONSTRATION
-- Parts A (Deadlock), B (MVCC), and C (Locking vs MVCC)
-- ===========================================================

-- Drop table if already exists
DROP TABLE IF EXISTS StudentEnrollments;

-- Create table
CREATE TABLE StudentEnrollments (
    student_id      INT PRIMARY KEY,
    student_name    VARCHAR(100) NOT NULL,
    course_id       VARCHAR(10) NOT NULL,
    enrollment_date DATE NOT NULL
);

-- Insert initial sample data
INSERT INTO StudentEnrollments (student_id, student_name, course_id, enrollment_date)
VALUES
(1, 'Ashish',  'CSE101', '2024-06-01'),
(2, 'Smaran',  'CSE102', '2024-06-01'),
(3, 'Vaibhav', 'CSE103', '2024-06-01');

-- ===========================================================
-- PART A: DEADLOCK SIMULATION
-- Open TWO sessions (User A and User B) and run as below:
-- ===========================================================

-- USER A
-- Session 1
START TRANSACTION;
UPDATE StudentEnrollments
SET enrollment_date = '2024-07-01'
WHERE student_id = 1;

-- Now try to update row 2 (this will wait if User B has locked it)
UPDATE StudentEnrollments
SET enrollment_date = '2024-07-02'
WHERE student_id = 2;
-- Do not COMMIT yet

-- USER B
-- Session 2
START TRANSACTION;
UPDATE StudentEnrollments
SET enrollment_date = '2024-08-01'
WHERE student_id = 2;

-- Now try to update row 1 (conflict with User A)
UPDATE StudentEnrollments
SET enrollment_date = '2024-08-02'
WHERE student_id = 1;
-- Deadlock occurs here: DB automatically rolls back one transaction

-- ===========================================================
-- PART B: MVCC DEMONSTRATION
-- Show how readers see old snapshot while writers update
-- ===========================================================

-- USER A
-- Session 1
START TRANSACTION;
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Output: Ashish, CSE101, 2024-06-01
-- Keep session open, DO NOT COMMIT yet

-- USER B
-- Session 2
START TRANSACTION;
UPDATE StudentEnrollments
SET enrollment_date = '2024-07-10'
WHERE student_id = 1;
COMMIT;

-- USER A (still open)
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Still sees old value 2024-06-01 due to snapshot isolation

-- USER A (after commit)
COMMIT;
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Now sees updated value 2024-07-10

-- ===========================================================
-- PART C: LOCKING vs MVCC
-- Compare behavior with and without MVCC
-- ===========================================================

-- CASE 1: Locking (using SELECT FOR UPDATE)
-- USER A
START TRANSACTION;
SELECT * FROM StudentEnrollments
WHERE student_id = 1
FOR UPDATE;
-- Row locked

-- USER B (run now)
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- BLOCKED until User A commits

-- USER A
COMMIT;

-- CASE 2: MVCC (normal SELECT)
-- USER A
START TRANSACTION;
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Reads snapshot value

-- USER B
UPDATE StudentEnrollments
SET enrollment_date = '2024-07-15'
WHERE student_id = 1;
COMMIT;

-- USER A (still open, before commit)
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Still sees old value

-- USER A (after commit)
COMMIT;
SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- Now sees 2024-07-15
