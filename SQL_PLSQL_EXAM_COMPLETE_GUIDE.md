# Complete SQL/PL/SQL Exam Preparation Guide
**Last Updated:** Exam Day Ready

---

## TABLE OF CONTENTS
1. [SQL Basics & Syntax](#sql-basics--syntax)
2. [8 Main Query Patterns](#8-main-query-patterns)
3. [7 Edge Case Patterns](#7-edge-case-patterns)
4. [PL/SQL Complete Reference](#plsql-complete-reference)
5. [JDBC Complete Reference](#jdbc-complete-reference)
6. [MongoDB Reference](#mongodb-reference)
7. [Expected Exam Questions](#expected-exam-questions)
8. [Common Mistakes to Avoid](#common-mistakes-to-avoid)

---

# SQL BASICS & SYNTAX

## Data Types in Oracle

```sql
NUMBER(p,s)        -- p=precision (total digits), s=scale (decimals)
                    -- NUMBER(10,2) = 12345678.90
VARCHAR2(n)        -- Variable-length string, max n chars
                    -- VARCHAR2(50) = up to 50 chars
CHAR(n)            -- Fixed-length string, pads with spaces
DATE                -- Stores date + time (second precision)
                    -- Default format: DD-MON-YY HH:MI:SS
TIMESTAMP          -- More precise than DATE (microseconds)
CLOB                -- Character Large Object (large text)
BLOB                -- Binary Large Object (images, files)
```

## Creating Tables

```sql
CREATE TABLE Consumer (
    Cons_ID   NUMBER PRIMARY KEY,
    Cons_Name VARCHAR2(100),
    City      VARCHAR2(50),
    Meter_No  VARCHAR2(20) UNIQUE
);

CREATE TABLE Bill (
    Bill_ID    NUMBER PRIMARY KEY,
    Cons_ID    NUMBER REFERENCES Consumer(Cons_ID),
    Bill_Month VARCHAR2(20),
    Units_Used NUMBER(10,2),
    Amount     NUMBER(10,2),
    Status     VARCHAR2(10) DEFAULT 'UNPAID'
               CHECK (Status IN ('PAID','UNPAID'))
);
```

## Basic CRUD Operations

```sql
-- CREATE (INSERT)
INSERT INTO Consumer VALUES (1, 'Tarun', 'Madurai', 'MTR001');
INSERT INTO Consumer (Cons_ID, Cons_Name) VALUES (2, 'Arjun');

-- READ (SELECT)
SELECT * FROM Consumer;
SELECT Cons_ID, Cons_Name FROM Consumer WHERE City = 'Madurai';
SELECT COUNT(*) FROM Consumer;
SELECT DISTINCT City FROM Consumer;

-- UPDATE
UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = 5;
UPDATE Consumer SET City = 'Chennai' WHERE Cons_ID = 1;

-- DELETE
DELETE FROM Bill WHERE Bill_ID = 999;
DELETE FROM Consumer WHERE Cons_ID = 1;

-- TRUNCATE (faster delete, cannot rollback)
TRUNCATE TABLE Bill;
```

## Commit and Rollback

```sql
-- Auto-commit mode (default)
INSERT INTO Bill VALUES (1, 1, 'JAN-2024', 300, 1200, 'UNPAID');
-- Automatically committed

-- Manual commit control
SET AUTOCOMMIT OFF;
INSERT INTO Bill VALUES (1, 1, 'JAN-2024', 300, 1200, 'UNPAID');
COMMIT;   -- Now permanent

-- Rollback
UPDATE Bill SET Amount = 9999 WHERE Bill_ID = 5;
ROLLBACK; -- Undo the update

-- Savepoint (partial rollback)
SAVEPOINT before_update;
UPDATE Bill SET Amount = 9999 WHERE Bill_ID = 5;
ROLLBACK TO before_update;  -- Undo only update, not prior inserts
```

## Date Functions

```sql
SYSDATE              -- Current date + time
TRUNC(SYSDATE)       -- Today at midnight (00:00:00)
TRUNC(d, 'MM')       -- First day of month
TRUNC(d, 'YYYY')     -- First day of year

-- Arithmetic with dates
SYSDATE + 7          -- 7 days from now
SYSDATE - 30         -- 30 days ago
TO_DATE('2024-01-01', 'YYYY-MM-DD')  -- String to date

-- Comparison
WHERE Bill_Date > SYSDATE - 30
WHERE Bill_Date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'
WHERE TRUNC(Bill_Date) = TRUNC(SYSDATE)
```

## NULL Handling

```sql
NVL(col, default_value)        -- Replace NULL with value
SELECT NVL(City, 'No City') FROM Consumer;

COALESCE(col1, col2, col3)     -- Return first non-null
SELECT COALESCE(mobile, email, 'No Contact') FROM Customer;

IS NULL                         -- Check for NULL
WHERE City IS NULL

IS NOT NULL                     -- Check for not NULL
WHERE Phone IS NOT NULL
```

## String Functions

```sql
UPPER(str)           -- Convert to uppercase
LOWER(str)           -- Convert to lowercase
SUBSTR(str, pos, len) -- Extract substring
LENGTH(str)          -- String length
CONCAT(str1, str2)   -- Concatenate (or use ||)
RPAD(str, len, pad)  -- Right-pad string
LPAD(str, len, pad)  -- Left-pad string
TRIM(str)            -- Remove leading/trailing spaces
REPLACE(str, old, new) -- Replace substring

-- Examples
SELECT UPPER(Cons_Name) FROM Consumer;
SELECT SUBSTR(Cons_Name, 1, 3) FROM Consumer;
SELECT Length(Cons_Name) FROM Consumer;
SELECT Cons_Name || ' from ' || City FROM Consumer;
```

## Math Functions

```sql
ROUND(num, decimals)  -- Round to N decimals
TRUNC(num, decimals)  -- Truncate to N decimals
ABS(num)              -- Absolute value
CEIL(num)             -- Ceiling
FLOOR(num)            -- Floor
MOD(num1, num2)       -- Modulo (remainder)
POWER(num, exp)       -- Power
SQRT(num)             -- Square root

-- Examples
SELECT ROUND(1234.567, 2);  -- 1234.57
SELECT TRUNC(1234.567, 2);  -- 1234.56
SELECT ABS(-100);           -- 100
```

---

# 8 MAIN QUERY PATTERNS

## PATTERN 1: NOT EXISTS (Things with no related record)

**Question type:** "Show flights with no reservations", "Cars not rented", "Consumers with no bills"

```sql
-- Template
SELECT a.*
FROM TableA a
WHERE NOT EXISTS (
    SELECT 1
    FROM TableB b
    WHERE b.fk_id = a.pk_id
);

-- Example: Cars never rented
SELECT c.Car_ID, c.Brand, c.Model
FROM Car c
WHERE NOT EXISTS (
    SELECT 1
    FROM Rental r
    WHERE r.Car_ID = c.Car_ID
);

-- Output:
-- CAR_ID  BRAND    MODEL
-- 3       Hyundai  Creta
-- 5       Maruti   Swift
```

**Why NOT EXISTS over NOT IN:**
```sql
-- WRONG: If subquery has NULL, NOT IN returns nothing
WHERE Car_ID NOT IN (SELECT Car_ID FROM Rental WHERE Car_ID IS NULL);

-- RIGHT: NOT EXISTS is NULL-safe
WHERE NOT EXISTS (
    SELECT 1 FROM Rental r WHERE r.Car_ID = c.Car_ID
);
```

---

## PATTERN 2: Highest Value (Max from aggregation)

**Question type:** "Salesman with highest salary", "Car generating highest revenue", "Product with max stock"

```sql
-- Template: SUM/COUNT aggregation, pick top one
SELECT a.pk_id, a.name, SUM(b.value_col) AS Total
FROM TableA a
JOIN TableB b ON a.pk_id = b.fk_id
GROUP BY a.pk_id, a.name
ORDER BY Total DESC
FETCH FIRST 1 ROW ONLY;

-- Example: Car with highest rental revenue
SELECT c.Car_ID, c.Brand, c.Model, SUM(r.Amount) AS Total_Revenue
FROM Car c
JOIN Rental r ON c.Car_ID = r.Car_ID
GROUP BY c.Car_ID, c.Brand, c.Model
ORDER BY Total_Revenue DESC
FETCH FIRST 1 ROW ONLY;

-- Output:
-- CAR_ID  BRAND   MODEL     TOTAL_REVENUE
-- 4       Toyota  Fortuner  17500

-- Alternative for Oracle 11g (use ROWNUM):
SELECT * FROM (
    SELECT c.Car_ID, c.Brand, SUM(r.Amount) AS Revenue
    FROM Car c JOIN Rental r ON c.Car_ID = r.Car_ID
    GROUP BY c.Car_ID, c.Brand
    ORDER BY Revenue DESC
) WHERE ROWNUM = 1;
```

---

## PATTERN 3: Column-wise Count/Sum (Monthly, City-wise, Brand-wise)

**Question type:** "Month-wise revenue", "City-wise customer count", "Brand-wise rental count"

```sql
-- Template
SELECT grouping_col,
       COUNT(*)        AS Record_Count,
       SUM(value_col)  AS Total_Value,
       AVG(value_col)  AS Avg_Value
FROM TableB
GROUP BY grouping_col
ORDER BY grouping_col;

-- Example: Month-wise electricity revenue
SELECT Bill_Month,
       COUNT(Bill_ID)  AS Bills_Generated,
       SUM(Units_Used) AS Total_Units,
       SUM(Amount)     AS Total_Revenue
FROM Bill
GROUP BY Bill_Month
ORDER BY Bill_Month;

-- Output:
-- BILL_MONTH  BILLS_GENERATED  TOTAL_UNITS  TOTAL_REVENUE
-- FEB-2024    5                2130         8520
-- JAN-2024    5                1950         7800

-- With JOIN (Brand-wise rental count)
SELECT c.Brand,
       COUNT(r.Rental_ID) AS Rental_Count,
       SUM(r.Amount)      AS Total_Revenue
FROM Car c
JOIN Rental r ON c.Car_ID = r.Car_ID
GROUP BY c.Brand
ORDER BY Rental_Count DESC;
```

---

## PATTERN 4: More Than Average (Heavy users)

**Question type:** "Customers renting more than average times", "Employees earning above avg salary"

```sql
-- Template: 3-layer logic
SELECT a.pk_id, a.name, COUNT(b.pk_id) AS Total
FROM TableA a
JOIN TableB b ON a.pk_id = b.fk_id
GROUP BY a.pk_id, a.name
HAVING COUNT(b.pk_id) > (
    SELECT AVG(cnt)
    FROM (
        SELECT COUNT(pk_id) AS cnt
        FROM TableB
        GROUP BY fk_id
    )
);

-- Layer breakdown:
-- Innermost : COUNT per entity           → {A:3, B:2, C:1}
-- Middle    : AVG of those counts        → (3+2+1)/3 = 2
-- Outer     : HAVING > that average      → only A with count 3

-- Example: Customers renting more than average
SELECT cu.Cust_ID, cu.Cust_Name, COUNT(r.Rental_ID) AS Rental_Count
FROM Customer cu
JOIN Rental r ON cu.Cust_ID = r.Cust_ID
GROUP BY cu.Cust_ID, cu.Cust_Name
HAVING COUNT(r.Rental_ID) > (
    SELECT AVG(cnt)
    FROM (
        SELECT COUNT(Rental_ID) AS cnt
        FROM Rental
        GROUP BY Cust_ID
    )
);

-- Output:
-- CUST_ID  CUST_NAME  RENTAL_COUNT
-- 1        Tarun      3
```

---

## PATTERN 5: More/Less Than N Value (Threshold)

**Question type:** "Products with stock < 10", "Bills with amount > 5000", "Employees earning > 50000"

```sql
-- Template: Simple WHERE or HAVING based on aggregation

-- On raw column
SELECT * FROM Product
WHERE Stock < 10;

-- On aggregated column (must use HAVING, not WHERE)
SELECT a.pk_id, a.name, SUM(b.value_col) AS Total
FROM TableA a
JOIN TableB b ON a.pk_id = b.fk_id
GROUP BY a.pk_id, a.name
HAVING SUM(b.value_col) > 5000;

-- Example: Consumers with unpaid bills amount > 2000
SELECT c.Cons_ID, c.Cons_Name, SUM(b.Amount) AS Unpaid_Amount
FROM Consumer c
JOIN Bill b ON c.Cons_ID = b.Cons_ID
WHERE b.Status = 'UNPAID'
GROUP BY c.Cons_ID, c.Cons_Name
HAVING SUM(b.Amount) > 2000;

-- KEY RULE: 
-- Raw column → WHERE
-- Aggregated column → HAVING
-- This is the #1 exam mistake!
```

---

## PATTERN 6: MongoDB Aggregation

**Question type:** "Show aggregated billing data", "Brand-wise rental statistics"

### Setup — Insert data

```javascript
db.bills.insertMany([
    { Bill_ID: 1, Cons_ID: 1, Amount: 1280, Units: 320, Status: "PAID" },
    { Bill_ID: 2, Cons_ID: 1, Amount: 1640, Units: 410, Status: "UNPAID" },
    { Bill_ID: 3, Cons_ID: 2, Amount: 600, Units: 150, Status: "PAID" },
    { Bill_ID: 4, Cons_ID: 2, Amount: 800, Units: 200, Status: "UNPAID" },
    { Bill_ID: 5, Cons_ID: 3, Amount: 2120, Units: 530, Status: "UNPAID" }
]);

db.rentals.insertMany([
    { Rental_ID: 1, Cust_ID: 1, Car_ID: 1, Brand: "Toyota", Amount: 7500 },
    { Rental_ID: 2, Cust_ID: 2, Car_ID: 2, Brand: "Honda", Amount: 3600 },
    { Rental_ID: 3, Cust_ID: 1, Car_ID: 4, Brand: "Toyota", Amount: 10500 }
]);
```

### Common aggregations

```javascript
// 1. Month-wise/Group-wise sum (like Pattern 3)
db.bills.aggregate([
    { $group: {
        _id: "$Cons_ID",
        total_amount: { $sum: "$Amount" },
        bill_count: { $sum: 1 },
        avg_amount: { $avg: "$Amount" }
    }},
    { $sort: { total_amount: -1 } }
]);

// 2. Filter + Group (like Pattern 1 + 3)
db.bills.aggregate([
    { $match: { Status: "UNPAID" } },
    { $group: {
        _id: "$Cons_ID",
        unpaid_total: { $sum: "$Amount" }
    }},
    { $sort: { unpaid_total: -1 } }
]);

// 3. Top 1 (like Pattern 2)
db.rentals.aggregate([
    { $group: {
        _id: "$Car_ID",
        total_revenue: { $sum: "$Amount" }
    }},
    { $sort: { total_revenue: -1 } },
    { $limit: 1 }
]);

// 4. With $lookup and $unwind (JOIN equivalent)
db.customer.aggregate([
    { $lookup: {
        from: "bill",
        localField: "Cons_ID",
        foreignField: "Cons_ID",
        as: "bills_data"
    }},
    { $unwind: "$bills_data" },
    { $match: { "bills_data.Status": "UNPAID" } },
    { $group: {
        _id: "$Cons_Name",
        total_unpaid: { $sum: "$bills_data.Amount" }
    }},
    { $sort: { total_unpaid: -1 } }
]);
```

### MongoDB aggregation pipeline stages

```javascript
$match      -- Filter documents (WHERE)
$group      -- Group and aggregate (GROUP BY)
$project    -- Select/reshape fields (SELECT col)
$sort       -- Sort results (ORDER BY)
$limit      -- Top N results (FETCH FIRST N)
$skip       -- Skip N results (pagination OFFSET)
$lookup     -- Join another collection (JOIN)
$unwind     -- Flatten array field (explode)
```

---

## PATTERN 7: PL/SQL Block on Table

**Question type:** "Write PL/SQL block to perform operations on Bill table"

```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Variables (always use %TYPE)
    v_cons_id Bill.Cons_ID%TYPE    := 3;
    v_month   Bill.Bill_Month%TYPE := 'MAR-2024';
    v_units   Bill.Units_Used%TYPE := 460;
    v_amount  Bill.Amount%TYPE;
    v_total   NUMBER := 0;

    -- Cursor
    CURSOR c_bills IS
        SELECT b.Bill_ID, b.Bill_Month, b.Units_Used, b.Amount, b.Status
        FROM Bill b
        WHERE b.Cons_ID = v_cons_id
        ORDER BY b.Bill_Month;

    r_bill c_bills%ROWTYPE;

BEGIN
    -- Calculate amount using slab pricing
    IF v_units <= 100 THEN
        v_amount := 0;
    ELSIF v_units <= 300 THEN
        v_amount := (v_units - 100) * 3.5;
    ELSE
        v_amount := (200 * 3.5) + ((v_units - 300) * 5);
    END IF;

    DBMS_OUTPUT.PUT_LINE('Calculated Amount: Rs.' || v_amount);

    -- Insert new bill
    INSERT INTO Bill (Bill_ID, Cons_ID, Bill_Month, Units_Used, Amount, Status)
    VALUES (11, v_cons_id, v_month, v_units, v_amount, 'UNPAID');
    DBMS_OUTPUT.PUT_LINE('New bill inserted.');

    -- Update payment status
    UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = 5;
    DBMS_OUTPUT.PUT_LINE('Bill marked PAID.');

    -- Cursor loop to print all bills
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('========== Bill Summary ==========');
    OPEN c_bills;
    LOOP
        FETCH c_bills INTO r_bill;
        EXIT WHEN c_bills%NOTFOUND;

        DBMS_OUTPUT.PUT_LINE(
            RPAD(r_bill.Bill_Month, 12) ||
            RPAD(r_bill.Units_Used, 8) ||
            RPAD(r_bill.Amount, 10) ||
            r_bill.Status
        );

        v_total := v_total + r_bill.Amount;
    END LOOP;
    CLOSE c_bills;

    DBMS_OUTPUT.PUT_LINE('==================================');
    DBMS_OUTPUT.PUT_LINE('Total Billed: Rs.' || v_total);

    COMMIT;

EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('ERROR: Duplicate Bill_ID.');
        ROLLBACK;
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('ERROR: Consumer not found.');
        ROLLBACK;
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('ERROR: ' || SQLERRM);
        ROLLBACK;
END;
/
```

**Key concepts:**
- `%TYPE` — Variable borrows column's datatype
- `%ROWTYPE` — One variable holds entire row
- `RPAD()` — Right-pad for alignment
- `EXIT WHEN c_bills%NOTFOUND` — Exit cursor loop
- `SQLERRM` — Oracle's error message string
- Always: `COMMIT` or `ROLLBACK` in exception

---

## PATTERN 8: PL/SQL with TCL (Transactions)

**Question type:** "Write PL/SQL with SAVEPOINT", "Demonstrate transaction control"

```sql
SET SERVEROUTPUT ON;

DECLARE
    v_cons_id  NUMBER := 3;
    v_amount   NUMBER := 5000;
    v_exists   NUMBER;
BEGIN
    -- Check precondition
    SELECT COUNT(*) INTO v_exists
    FROM Consumer WHERE Cons_ID = v_cons_id;

    IF v_exists = 0 THEN
        DBMS_OUTPUT.PUT_LINE('Consumer not found. Aborting.');
        RETURN;
    END IF;

    -- Bookmark before risky operations
    SAVEPOINT before_changes;
    DBMS_OUTPUT.PUT_LINE('Savepoint created: before_changes');

    -- Operation 1: Insert bill
    INSERT INTO Bill VALUES (99, v_cons_id, 'MAR-2024', 450, 1800, 'UNPAID');
    DBMS_OUTPUT.PUT_LINE('Bill inserted.');

    -- Another bookmark
    SAVEPOINT after_insert;
    DBMS_OUTPUT.PUT_LINE('Savepoint created: after_insert');

    -- Operation 2: Update consumer (risky)
    UPDATE Consumer SET City = 'Chennai' WHERE Cons_ID = v_cons_id;
    DBMS_OUTPUT.PUT_LINE('Consumer updated.');

    -- Validation check
    IF v_amount > 10000 THEN
        ROLLBACK TO after_insert;  -- Undo only the update, keep insert
        DBMS_OUTPUT.PUT_LINE('Amount too high. Update rolled back.');
    ELSE
        COMMIT;  -- Everything commits
        DBMS_OUTPUT.PUT_LINE('All changes committed.');
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK TO before_changes;  -- Undo all operations
        DBMS_OUTPUT.PUT_LINE('Fatal error: ' || SQLERRM);
END;
/
```

**TCL Keywords:**
```
COMMIT              -- Makes all changes permanent
ROLLBACK            -- Undoes everything back to last commit
SAVEPOINT name      -- Places a bookmark mid-transaction
ROLLBACK TO name    -- Undoes only back to that bookmark
```

---

# 7 EDGE CASE PATTERNS

## PATTERN 9: Second Highest / Nth Highest Value

**Question type:** "Second highest salary", "Third most rented car"

```sql
-- Method 1: Using MAX on filtered data (second highest only)
SELECT MAX(Amount) AS Second_Highest
FROM Bill
WHERE Amount < (SELECT MAX(Amount) FROM Bill);

-- Method 2: Universal (works for any N)
SELECT Amount FROM (
    SELECT Amount,
           DENSE_RANK() OVER (ORDER BY Amount DESC) AS rnk
    FROM Bill
)
WHERE rnk = 2;  -- Change 2 to N for any rank

-- Important: DENSE_RANK vs RANK
-- If two people share rank 1:
-- RANK gives: 1, 1, 3, 4...     (skips rank 2)
-- DENSE_RANK gives: 1, 1, 2, 3... (no skipping)
-- Use DENSE_RANK for "second highest"
```

---

## PATTERN 10: Date Range Filter / Most Recent Record

**Question type:** "Bills in last 30 days", "Most recent rental per customer"

```sql
-- Between two dates
SELECT * FROM Bill
WHERE Bill_Date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31';

-- Last N days
SELECT * FROM Bill
WHERE Bill_Date >= SYSDATE - 30;

-- Most recent per customer (use window function)
SELECT Cust_ID, Rent_Date, Amount FROM (
    SELECT Cust_ID, Rent_Date, Amount,
           ROW_NUMBER() OVER (PARTITION BY Cust_ID ORDER BY Rent_Date DESC) AS rn
    FROM Rental
)
WHERE rn = 1;

-- Simpler approach (old style)
SELECT Cust_ID, MAX(Rent_Date) AS Last_Rental
FROM Rental
GROUP BY Cust_ID;
```

---

## PATTERN 11: JOIN Types (INNER, LEFT, RIGHT, FULL)

**Question type:** "Show all consumers even if no bills", "Find unmatched records"

```sql
-- INNER JOIN (default): only matching rows
SELECT c.Cons_Name, b.Bill_Month, b.Amount
FROM Consumer c
INNER JOIN Bill b ON c.Cons_ID = b.Cons_ID;
-- Consumers with no bills don't appear

-- LEFT JOIN: all from left table, matched right
SELECT c.Cons_Name, b.Bill_Month, b.Amount
FROM Consumer c
LEFT JOIN Bill b ON c.Cons_ID = b.Cons_ID;
-- Shows all consumers; NULL for bills if no match

-- RIGHT JOIN: all from right table, matched left
SELECT c.Cons_Name, b.Bill_Month
FROM Consumer c
RIGHT JOIN Bill b ON c.Cons_ID = b.Cons_ID;

-- FULL OUTER JOIN: all rows from both tables
SELECT c.Cons_Name, b.Bill_Month
FROM Consumer c
FULL OUTER JOIN Bill b ON c.Cons_ID = b.Cons_ID;

-- SELF JOIN: table joined with itself
SELECT a.Emp_Name, b.Manager_Name
FROM Employee a
JOIN Employee b ON a.Manager_ID = b.Emp_ID;
```

---

## PATTERN 12: TRIGGER (Auto-execute on event)

**Question type:** "Auto-calculate bill amount", "Prevent deletion of paid bills"

```sql
-- BEFORE INSERT: Calculate amount automatically
CREATE OR REPLACE TRIGGER trg_calc_amount
BEFORE INSERT ON Bill
FOR EACH ROW
BEGIN
    IF :NEW.Units_Used <= 100 THEN
        :NEW.Amount := 0;
    ELSIF :NEW.Units_Used <= 300 THEN
        :NEW.Amount := (:NEW.Units_Used - 100) * 3.5;
    ELSE
        :NEW.Amount := (200 * 3.5) + ((:NEW.Units_Used - 300) * 5);
    END IF;
END trg_calc_amount;
/

-- BEFORE DELETE: Protect paid bills
CREATE OR REPLACE TRIGGER trg_protect_paid
BEFORE DELETE ON Bill
FOR EACH ROW
BEGIN
    IF :OLD.Status = 'PAID' THEN
        RAISE_APPLICATION_ERROR(-20001, 'Cannot delete PAID bill!');
    END IF;
END trg_protect_paid;
/

-- AFTER INSERT: Audit trail
CREATE TABLE Bill_Audit (
    Audit_ID NUMBER PRIMARY KEY,
    Bill_ID  NUMBER,
    Action   VARCHAR2(20),
    Changed_On DATE
);

CREATE SEQUENCE audit_seq START WITH 1;

CREATE OR REPLACE TRIGGER trg_audit
AFTER INSERT OR DELETE ON Bill
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO Bill_Audit VALUES (audit_seq.NEXTVAL, :NEW.Bill_ID, 'INSERT', SYSDATE);
    ELSIF DELETING THEN
        INSERT INTO Bill_Audit VALUES (audit_seq.NEXTVAL, :OLD.Bill_ID, 'DELETE', SYSDATE);
    END IF;
END trg_audit;
/

-- View trigger status
SELECT trigger_name, status FROM user_triggers;

-- Disable/Enable trigger
ALTER TRIGGER trg_calc_amount DISABLE;
ALTER TRIGGER trg_calc_amount ENABLE;
```

---

## PATTERN 13: PROCEDURE (Reusable routine)

**Question type:** "Write procedure to update bill status", "Procedure to insert payment"

```sql
CREATE OR REPLACE PROCEDURE proc_pay_bill(
    p_bill_id IN NUMBER,
    p_status OUT VARCHAR2
) AS
    v_current_status Bill.Status%TYPE;
BEGIN
    -- Check if bill exists and get current status
    SELECT Status INTO v_current_status
    FROM Bill WHERE Bill_ID = p_bill_id;

    IF v_current_status = 'PAID' THEN
        p_status := 'ALREADY_PAID';
    ELSE
        UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = p_bill_id;
        p_status := 'SUCCESS';
    END IF;

    COMMIT;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        p_status := 'NOT_FOUND';
        ROLLBACK;
    WHEN OTHERS THEN
        p_status := 'ERROR: ' || SQLERRM;
        ROLLBACK;
END proc_pay_bill;
/

-- Call procedure
DECLARE
    v_result VARCHAR2(100);
BEGIN
    proc_pay_bill(5, v_result);
    DBMS_OUTPUT.PUT_LINE(v_result);
END;
/

-- Or directly
EXEC proc_pay_bill(5, :result);
```

---

## PATTERN 14: FUNCTION (Returns a value)

**Question type:** "Function to calculate total spent", "Function to check bill status"

```sql
CREATE OR REPLACE FUNCTION fn_total_spent(p_cons_id IN NUMBER)
RETURN NUMBER AS
    v_total NUMBER;
BEGIN
    SELECT SUM(Amount) INTO v_total
    FROM Bill WHERE Cons_ID = p_cons_id AND Status = 'PAID';
    RETURN NVL(v_total, 0);
EXCEPTION
    WHEN OTHERS THEN
        RETURN -1;
END fn_total_spent;
/

-- Use in SELECT
SELECT Cons_ID, Cons_Name, fn_total_spent(Cons_ID) AS Total_Paid
FROM Consumer;

-- Use in WHERE
SELECT * FROM Consumer
WHERE fn_total_spent(Cons_ID) > 5000;

-- Use in calculation
SELECT Cons_ID, fn_total_spent(Cons_ID) * 1.10 AS With_Interest
FROM Consumer;
```

---

## PATTERN 15: CASE / DECODE (Conditional logic)

**Question type:** "Categorize consumption", "Label payment status"

```sql
-- CASE (standard SQL, preferred)
SELECT Bill_ID, Units_Used,
       CASE
           WHEN Units_Used < 100  THEN 'Low'
           WHEN Units_Used < 400  THEN 'Medium'
           ELSE                        'High'
       END AS Category
FROM Bill;

-- CASE with status
SELECT Bill_ID,
       CASE Status
           WHEN 'PAID'   THEN 'Cleared'
           WHEN 'UNPAID' THEN 'Pending'
           ELSE               'Unknown'
       END AS Status_Label
FROM Bill;

-- DECODE (Oracle-specific, older)
SELECT Bill_ID,
       DECODE(Status, 'PAID', 'Cleared', 'UNPAID', 'Pending', 'Unknown') AS Label
FROM Bill;
```

---

# PLSQL COMPLETE REFERENCE

## Variables and Data Types

```sql
DECLARE
    -- Basic types
    v_num     NUMBER := 100;
    v_num_p   NUMBER(10,2) := 1234.56;
    v_str     VARCHAR2(50) := 'text';
    v_date    DATE := SYSDATE;
    v_bool    BOOLEAN := TRUE;

    -- Using %TYPE (preferred)
    v_cons_id Bill.Cons_ID%TYPE;
    v_amount  Bill.Amount%TYPE;

    -- Using %ROWTYPE (entire row)
    r_bill    Bill%ROWTYPE;

    -- CURSOR (pointer to result set)
    CURSOR c_bills IS
        SELECT * FROM Bill WHERE Status = 'UNPAID';

    -- Record type (custom)
    TYPE rec_bill IS RECORD (
        id     Bill.Bill_ID%TYPE,
        amount Bill.Amount%TYPE,
        status Bill.Status%TYPE
    );
    r_custom rec_bill;

    -- Table type (array)
    TYPE tab_ids IS TABLE OF NUMBER;
    v_ids tab_ids;
BEGIN
    NULL;
END;
/
```

---

## Control Flow

```sql
BEGIN
    -- IF-ELSE
    IF condition THEN
        statement;
    ELSIF condition THEN
        statement;
    ELSE
        statement;
    END IF;

    -- CASE statement
    CASE variable
        WHEN value1 THEN statement;
        WHEN value2 THEN statement;
        ELSE statement;
    END CASE;

    -- LOOP
    LOOP
        statement;
        EXIT WHEN condition;
    END LOOP;

    -- WHILE loop
    WHILE condition LOOP
        statement;
    END LOOP;

    -- FOR loop
    FOR i IN 1..10 LOOP
        DBMS_OUTPUT.PUT_LINE(i);
    END LOOP;

    -- FOR cursor loop
    FOR r IN c_bills LOOP
        DBMS_OUTPUT.PUT_LINE(r.Bill_ID);
    END LOOP;
END;
/
```

---

## Cursor Operations

```sql
DECLARE
    CURSOR c_data IS
        SELECT Bill_ID, Amount, Status FROM Bill;
    r_data c_data%ROWTYPE;
BEGIN
    OPEN c_data;
    LOOP
        FETCH c_data INTO r_data;
        EXIT WHEN c_data%NOTFOUND;

        -- Use r_data here
        DBMS_OUTPUT.PUT_LINE(r_data.Bill_ID || ' - ' || r_data.Amount);

        -- Cursor attributes
        IF c_data%FOUND THEN       -- Last fetch successful
            NULL;
        END IF;
        IF c_data%ROWCOUNT > 100 THEN  -- Rows fetched so far
            NULL;
        END IF;
    END LOOP;
    CLOSE c_data;
END;
/
```

---

## Exception Handling

```sql
BEGIN
    statement;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No records found');

    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('Duplicate key');

    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('More than one row');

    WHEN VALUE_ERROR THEN
        DBMS_OUTPUT.PUT_LINE('Type mismatch');

    WHEN INVALID_CURSOR THEN
        DBMS_OUTPUT.PUT_LINE('Cursor not opened');

    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Division by zero');

    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        SQLCODE;  -- error number
END;
/
```

---

## PL/SQL Built-in Packages

```sql
-- DBMS_OUTPUT (print)
DBMS_OUTPUT.PUT_LINE('text');
DBMS_OUTPUT.PUT('text');     -- no newline

-- DBMS_SQL (dynamic SQL)
EXECUTE IMMEDIATE 'INSERT INTO Bill VALUES (99, 1, ''JAN'', 300, 1200, ''UNPAID'')';

-- DBMS_LOCK (sleep)
DBMS_LOCK.SLEEP(5);  -- sleep 5 seconds

-- UTL_FILE (file I/O)
v_file := UTL_FILE.FOPEN('/path', 'filename', 'R');
UTL_FILE.GET_LINE(v_file, v_line);

-- DBMS_UTILITY (utilities)
DBMS_UTILITY.GET_TIME;  -- milliseconds since start
```

---

## Package Example

```sql
-- Specification
CREATE OR REPLACE PACKAGE pkg_billing AS
    PROCEDURE pay_bill(p_bill_id IN NUMBER);
    FUNCTION get_total_due(p_cons_id IN NUMBER) RETURN NUMBER;
    g_error_msg VARCHAR2(500);
END pkg_billing;
/

-- Body
CREATE OR REPLACE PACKAGE BODY pkg_billing AS
    PROCEDURE pay_bill(p_bill_id IN NUMBER) AS
    BEGIN
        UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = p_bill_id;
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            g_error_msg := SQLERRM;
            ROLLBACK;
    END pay_bill;

    FUNCTION get_total_due(p_cons_id IN NUMBER) RETURN NUMBER AS
        v_total NUMBER;
    BEGIN
        SELECT SUM(Amount) INTO v_total
        FROM Bill WHERE Cons_ID = p_cons_id AND Status = 'UNPAID';
        RETURN NVL(v_total, 0);
    END get_total_due;
END pkg_billing;
/

-- Use package
EXEC pkg_billing.pay_bill(5);
SELECT pkg_billing.get_total_due(3);
```

---

# JDBC COMPLETE REFERENCE

## Setup

```java
// 1. Import
import java.sql.*;

// 2. Load driver
Class.forName("oracle.jdbc.driver.OracleDriver");

// 3. Connect
Connection con = DriverManager.getConnection(
    "jdbc:oracle:thin:@localhost:1521:XE",
    "system",
    "oracle"
);

// 4. Create statement
Statement stmt = con.createStatement();
```

---

## CRUD Operations

```java
// CREATE (INSERT)
int rows = stmt.executeUpdate(
    "INSERT INTO Consumer VALUES (1, 'Tarun', 'Madurai', 'MTR001')"
);
System.out.println(rows + " row(s) inserted");

// READ (SELECT)
ResultSet rs = stmt.executeQuery("SELECT * FROM Consumer");
while (rs.next()) {
    int    id   = rs.getInt(1);        // by index (starts at 1)
    String name = rs.getString("Cons_Name");  // by name
    System.out.println(id + " | " + name);
}

// UPDATE
int updated = stmt.executeUpdate(
    "UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = 5"
);

// DELETE
int deleted = stmt.executeUpdate(
    "DELETE FROM Bill WHERE Bill_ID = 999"
);
```

---

## PreparedStatement (with parameters)

```java
// Create with placeholders
PreparedStatement ps = con.prepareStatement(
    "INSERT INTO Bill VALUES (?, ?, ?, ?, ?, ?)"
);

// Set parameters
ps.setInt(1, 12);
ps.setInt(2, 3);
ps.setString(3, "MAR-2024");
ps.setDouble(4, 450);
ps.setDouble(5, 1800);
ps.setString(6, "UNPAID");

// Execute
int rows = ps.executeUpdate();

// For SELECT with parameters
ps = con.prepareStatement("SELECT * FROM Bill WHERE Cons_ID = ? AND Status = ?");
ps.setInt(1, 3);
ps.setString(2, "UNPAID");
rs = ps.executeQuery();
while (rs.next()) {
    System.out.println(rs.getInt("Bill_ID"));
}

// Always close
rs.close();
ps.close();
```

---

## Transactions (TCL in JDBC)

```java
try {
    con.setAutoCommit(false);  // Turn off auto-commit

    stmt.executeUpdate("INSERT INTO Bill VALUES (...)");
    stmt.executeUpdate("UPDATE Consumer SET City = 'Chennai' WHERE Cons_ID = 1");

    con.commit();  // Both succeed
    System.out.println("Committed");

} catch (SQLException e) {
    con.rollback();  // Rollback on error
    System.out.println("Rolled back: " + e.getMessage());
}
```

---

## Savepoint

```java
con.setAutoCommit(false);

stmt.executeUpdate("INSERT INTO Bill VALUES (...)");

Savepoint sp = con.setSavepoint("after_insert");

stmt.executeUpdate("UPDATE Bill SET Amount = 9999 WHERE Bill_ID = 999");

con.rollback(sp);  // Undo only update, keep insert
con.commit();

con.close();
```

---

## ResultSetMetaData (dynamic column info)

```java
ResultSet rs = stmt.executeQuery("SELECT * FROM Consumer");
ResultSetMetaData md = rs.getMetaData();

int cols = md.getColumnCount();

// Print headers
for (int i = 1; i <= cols; i++) {
    System.out.print(md.getColumnName(i) + "\t");
}
System.out.println();

// Print rows
while (rs.next()) {
    for (int i = 1; i <= cols; i++) {
        System.out.print(rs.getString(i) + "\t");
    }
    System.out.println();
}
```

---

## Full JDBC Template

```java
import java.sql.*;

public class JDBCApp {
    public static void main(String[] args) throws Exception {
        Connection con = null;
        Statement stmt = null;
        ResultSet rs = null;

        try {
            Class.forName("oracle.jdbc.driver.OracleDriver");
            con = DriverManager.getConnection(
                "jdbc:oracle:thin:@localhost:1521:XE", "system", "oracle"
            );

            stmt = con.createStatement();

            // INSERT
            int rows = stmt.executeUpdate(
                "INSERT INTO Consumer VALUES (99, 'Test', 'City', 'MTR999')"
            );
            System.out.println("Inserted: " + rows);

            // SELECT
            rs = stmt.executeQuery("SELECT * FROM Consumer");
            while (rs.next()) {
                System.out.println(rs.getInt(1) + " | " + rs.getString(2));
            }

            // UPDATE
            int updated = stmt.executeUpdate(
                "UPDATE Consumer SET City = 'Chennai' WHERE Cons_ID = 99"
            );
            System.out.println("Updated: " + updated);

            con.commit();

        } catch (Exception e) {
            e.printStackTrace();
            if (con != null) con.rollback();
        } finally {
            if (rs != null) rs.close();
            if (stmt != null) stmt.close();
            if (con != null) con.close();
        }
    }
}
```

---

## JDBC Common Mistakes

```java
// ❌ Column index starts at 0
rs.getString(0);  // WRONG

// ✓ Column index starts at 1
rs.getString(1);  // RIGHT

// ❌ Using executeQuery for INSERT
ResultSet rs = stmt.executeQuery("INSERT INTO ...");  // WRONG

// ✓ Using executeUpdate for INSERT
int rows = stmt.executeUpdate("INSERT INTO ...");  // RIGHT

// ❌ Closing connection first
con.close();
rs.close();
stmt.close();  // WRONG order

// ✓ Close in reverse order
rs.close();
stmt.close();
con.close();  // RIGHT order

// ❌ Forgetting Class.forName
// JVM doesn't load driver

// ✓ Always load driver first
Class.forName("oracle.jdbc.driver.OracleDriver");

// ❌ Empty catch block
catch (Exception e) {
    ;  // hides errors
}

// ✓ Print error
catch (Exception e) {
    e.printStackTrace();
}

// ❌ Not setting autocommit for transactions
// Every statement auto-commits

// ✓ Disable autocommit
con.setAutoCommit(false);
```

---

# MONGODB REFERENCE

## Setup and Basic Operations

```javascript
// Create database
use billing_db

// Insert single document
db.bills.insertOne({
    Bill_ID: 1,
    Cons_ID: 1,
    Amount: 1280,
    Status: "PAID"
})

// Insert multiple
db.bills.insertMany([
    { Bill_ID: 1, Cons_ID: 1, Amount: 1280, Status: "PAID" },
    { Bill_ID: 2, Cons_ID: 1, Amount: 1640, Status: "UNPAID" },
    { Bill_ID: 3, Cons_ID: 2, Amount: 600, Status: "PAID" }
])

// Select all
db.bills.find({})

// Select with condition
db.bills.find({ Status: "UNPAID" })
db.bills.find({ Amount: { $gt: 1000 } })

// Select specific fields
db.bills.find({ Status: "UNPAID" }, { Bill_ID: 1, Amount: 1, _id: 0 })

// Update
db.bills.updateOne({ Bill_ID: 5 }, { $set: { Status: "PAID" } })
db.bills.updateMany({ Status: "UNPAID" }, { $set: { Status: "PAID" } })

// Delete
db.bills.deleteOne({ Bill_ID: 5 })
db.bills.deleteMany({ Status: "UNPAID" })

// Count
db.bills.countDocuments({ Status: "UNPAID" })

// Distinct
db.bills.distinct("Cons_ID")
```

---

## Aggregation Pipeline

```javascript
// $match (WHERE)
db.bills.aggregate([
    { $match: { Status: "UNPAID", Amount: { $gt: 1000 } } }
])

// $group (GROUP BY)
db.bills.aggregate([
    { $group: {
        _id: "$Cons_ID",           // group by
        total: { $sum: "$Amount" },
        count: { $sum: 1 }
    }}
])

// $project (SELECT)
db.bills.aggregate([
    { $project: {
        Cons_ID: 1,
        Amount: 1,
        Tax: { $multiply: ["$Amount", 0.18] },
        _id: 0
    }}
])

// $sort (ORDER BY)
db.bills.aggregate([
    { $group: { _id: "$Cons_ID", total: { $sum: "$Amount" } } },
    { $sort: { total: -1 } }
])

// $limit (FETCH FIRST)
db.bills.aggregate([
    { $group: { _id: "$Cons_ID", total: { $sum: "$Amount" } } },
    { $sort: { total: -1 } },
    { $limit: 1 }
])

// $skip (OFFSET)
db.bills.aggregate([
    { $skip: 10 },
    { $limit: 5 }
])

// $lookup (JOIN)
db.consumer.aggregate([
    { $lookup: {
        from: "bill",
        localField: "Cons_ID",
        foreignField: "Cons_ID",
        as: "bills_data"
    }}
])

// $unwind (FLATTEN ARRAY)
db.consumer.aggregate([
    { $lookup: {
        from: "bill",
        localField: "Cons_ID",
        foreignField: "Cons_ID",
        as: "bills_data"
    }},
    { $unwind: "$bills_data" }
])

// Full pipeline: Find total unpaid per consumer
db.consumer.aggregate([
    { $lookup: {
        from: "bill",
        localField: "Cons_ID",
        foreignField: "Cons_ID",
        as: "bills"
    }},
    { $unwind: "$bills" },
    { $match: { "bills.Status": "UNPAID" } },
    { $group: {
        _id: "$Cons_Name",
        unpaid_total: { $sum: "$bills.Amount" }
    }},
    { $sort: { unpaid_total: -1 } }
])
```

---

## MongoDB Comparison Operators

```javascript
$eq   // equal           =
$ne   // not equal       !=
$gt   // greater than    >
$gte  // >=              >=
$lt   // <               <
$lte  // <=              <=
$in   // in list         IN (a,b,c)
$nin  // not in list     NOT IN (a,b,c)

// Examples
db.bills.find({ Amount: { $gt: 1000 } })
db.bills.find({ Amount: { $lte: 5000 } })
db.bills.find({ Cons_ID: { $in: [1, 2, 3] } })
db.bills.find({ Status: { $ne: "PAID" } })
```

---

# EXPECTED EXAM QUESTIONS

## Question Set 1 — Car Rental System

```sql
-- Tables
CREATE TABLE Car (Car_ID, Brand, Model, Rent_Per_Day, Availability);
CREATE TABLE Customer (Cust_ID, Cust_Name, License_No, City);
CREATE TABLE Rental (Rental_ID, Cust_ID, Car_ID, Rent_Date, Amount);

-- Q1: Display cars not rented yet (PATTERN 1)
SELECT c.Car_ID, c.Brand, c.Model
FROM Car c
WHERE NOT EXISTS (
    SELECT 1 FROM Rental r WHERE r.Car_ID = c.Car_ID
);

-- Q2: Car generating highest rental revenue (PATTERN 2)
SELECT c.Car_ID, c.Brand, SUM(r.Amount) AS Total_Revenue
FROM Car c
JOIN Rental r ON c.Car_ID = r.Car_ID
GROUP BY c.Car_ID, c.Brand
ORDER BY Total_Revenue DESC
FETCH FIRST 1 ROW ONLY;

-- Q3: Brand-wise rental count (PATTERN 3)
SELECT c.Brand, COUNT(r.Rental_ID) AS Rental_Count
FROM Car c
JOIN Rental r ON c.Car_ID = r.Car_ID
GROUP BY c.Brand
ORDER BY Rental_Count DESC;

-- Q4: Customers renting more than average (PATTERN 4)
SELECT cu.Cust_ID, cu.Cust_Name, COUNT(r.Rental_ID) AS Rental_Count
FROM Customer cu
JOIN Rental r ON cu.Cust_ID = r.Cust_ID
GROUP BY cu.Cust_ID, cu.Cust_Name
HAVING COUNT(r.Rental_ID) > (
    SELECT AVG(cnt) FROM (
        SELECT COUNT(Rental_ID) AS cnt FROM Rental GROUP BY Cust_ID
    )
);

-- Q5: MongoDB aggregation (PATTERN 6)
db.rentals.aggregate([
    { $group: {
        _id: "$Brand",
        total_revenue: { $sum: "$Amount" },
        rental_count: { $sum: 1 }
    }},
    { $sort: { total_revenue: -1 } }
])

-- Q6: PL/SQL block (PATTERN 7)
SET SERVEROUTPUT ON;
DECLARE
    v_cust_id NUMBER := 1;
    v_total NUMBER := 0;
    CURSOR c_rentals IS
        SELECT Rental_ID, Amount FROM Rental WHERE Cust_ID = v_cust_id;
    r_rental c_rentals%ROWTYPE;
BEGIN
    OPEN c_rentals;
    LOOP
        FETCH c_rentals INTO r_rental;
        EXIT WHEN c_rentals%NOTFOUND;
        v_total := v_total + r_rental.Amount;
        DBMS_OUTPUT.PUT_LINE(r_rental.Rental_ID || ' - ' || r_rental.Amount);
    END LOOP;
    CLOSE c_rentals;
    DBMS_OUTPUT.PUT_LINE('Total: ' || v_total);
    COMMIT;
EXCEPTION WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
    ROLLBACK;
END;
/
```

---

## Question Set 2 — Electricity Billing System

```sql
-- Tables
CREATE TABLE Consumer (Cons_ID, Cons_Name, City, Meter_No);
CREATE TABLE Bill (Bill_ID, Cons_ID, Bill_Month, Units_Used, Amount, Status);

-- Q1: Consumers with unpaid bills (PATTERN 1 variant)
SELECT DISTINCT c.Cons_ID, c.Cons_Name, c.City
FROM Consumer c
JOIN Bill b ON c.Cons_ID = b.Cons_ID
WHERE b.Status = 'UNPAID';

-- Q2: Consumer with highest usage (PATTERN 2)
SELECT c.Cons_ID, c.Cons_Name, SUM(b.Units_Used) AS Total_Units
FROM Consumer c
JOIN Bill b ON c.Cons_ID = b.Cons_ID
GROUP BY c.Cons_ID, c.Cons_Name
ORDER BY Total_Units DESC
FETCH FIRST 1 ROW ONLY;

-- Q3: Month-wise revenue (PATTERN 3)
SELECT Bill_Month,
       SUM(Amount) AS Total_Revenue,
       COUNT(Bill_ID) AS Bills_Generated
FROM Bill
GROUP BY Bill_Month
ORDER BY Bill_Month;

-- Q4: City-wise average consumption (PATTERN 3)
SELECT c.City,
       ROUND(AVG(b.Units_Used), 2) AS Avg_Units
FROM Consumer c
JOIN Bill b ON c.Cons_ID = b.Cons_ID
GROUP BY c.City
ORDER BY Avg_Units DESC;

-- Q5: PL/SQL with TCL (PATTERN 8)
SET SERVEROUTPUT ON;
DECLARE
    v_bill_id NUMBER := 5;
    v_amount NUMBER := 2500;
BEGIN
    SAVEPOINT before_update;
    UPDATE Bill SET Status = 'PAID' WHERE Bill_ID = v_bill_id;
    DBMS_OUTPUT.PUT_LINE('Bill marked PAID');

    IF v_amount > 10000 THEN
        ROLLBACK TO before_update;
        DBMS_OUTPUT.PUT_LINE('Amount too high, rolled back');
    ELSE
        COMMIT;
        DBMS_OUTPUT.PUT_LINE('Changes committed');
    END IF;
EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

---

## Question Set 3 — Custom Schema (Likely Exam)

```sql
-- If given a custom schema, apply patterns:

-- PATTERN 1: Not existing
SELECT parent_table.*
FROM parent_table p
WHERE NOT EXISTS (
    SELECT 1 FROM child_table c WHERE c.fk_id = p.pk_id
);

-- PATTERN 2: Highest
SELECT p.pk_id, p.name, SUM(c.value) AS Total
FROM parent_table p
JOIN child_table c ON p.pk_id = c.fk_id
GROUP BY p.pk_id, p.name
ORDER BY Total DESC
FETCH FIRST 1 ROW ONLY;

-- PATTERN 3: Group-wise
SELECT grouping_col, COUNT(*), SUM(value_col)
FROM table
GROUP BY grouping_col
ORDER BY grouping_col;

-- PATTERN 4: Above average
SELECT p.pk_id, p.name, COUNT(c.pk_id) AS cnt
FROM parent_table p
JOIN child_table c ON p.pk_id = c.fk_id
GROUP BY p.pk_id, p.name
HAVING COUNT(c.pk_id) > (
    SELECT AVG(cnt) FROM (
        SELECT COUNT(pk_id) AS cnt FROM child_table GROUP BY fk_id
    )
);

-- PATTERN 5: Threshold
SELECT * FROM table WHERE value > 5000;
-- OR
SELECT p.pk_id, SUM(c.value) AS total
FROM parent p JOIN child c ON p.pk_id = c.fk_id
GROUP BY p.pk_id
HAVING SUM(c.value) < 1000;
```

---

# COMMON MISTAKES TO AVOID

## SQL Mistakes

### ❌ Mistake 1: GROUP BY missing column
```sql
-- WRONG
SELECT Cust_ID, Cust_Name, COUNT(*)
FROM Customer
GROUP BY Cust_ID;  -- ERROR: Cust_Name not in GROUP BY

-- RIGHT
SELECT Cust_ID, Cust_Name, COUNT(*)
FROM Customer
GROUP BY Cust_ID, Cust_Name;
```

### ❌ Mistake 2: WHERE instead of HAVING
```sql
-- WRONG: filtering aggregate before grouping
SELECT Cust_ID, SUM(Amount)
FROM Bill
WHERE SUM(Amount) > 5000  -- ERROR: WHERE can't have aggregates
GROUP BY Cust_ID;

-- RIGHT
SELECT Cust_ID, SUM(Amount) AS Total
FROM Bill
GROUP BY Cust_ID
HAVING SUM(Amount) > 5000;  -- HAVING after GROUP BY
```

### ❌ Mistake 3: NOT IN with NULL
```sql
-- WRONG: if subquery returns NULL, NOT IN returns no rows
SELECT * FROM Car
WHERE Car_ID NOT IN (SELECT Car_ID FROM Rental WHERE Car_ID IS NULL);

-- RIGHT: use NOT EXISTS
SELECT c.* FROM Car c
WHERE NOT EXISTS (
    SELECT 1 FROM Rental r WHERE r.Car_ID = c.Car_ID
);
```

### ❌ Mistake 4: Comparing DATE with time
```sql
-- WRONG: includes time, won't match
WHERE Bill_Date = DATE '2024-01-01';  -- searches for midnight only

-- RIGHT: use TRUNC
WHERE TRUNC(Bill_Date) = DATE '2024-01-01';
```

### ❌ Mistake 5: Missing column in ORDER BY that exists in SELECT
```sql
-- WRONG (in some Oracle versions)
SELECT Cust_Name FROM Customer ORDER BY City;

-- RIGHT
SELECT Cust_Name, City FROM Customer ORDER BY City;
```

---

## PL/SQL Mistakes

### ❌ Mistake 6: Using variable instead of %TYPE
```sql
-- WRONG: hardcoded type, breaks if column type changes
v_amount NUMBER(10,2);

-- RIGHT: borrows column type
v_amount Bill.Amount%TYPE;
```

### ❌ Mistake 7: Cursor loop without EXIT
```sql
-- WRONG: infinite loop
OPEN c;
LOOP
    FETCH c INTO r;
    -- no EXIT, loops forever
END LOOP;

-- RIGHT
OPEN c;
LOOP
    FETCH c INTO r;
    EXIT WHEN c%NOTFOUND;
END LOOP;
CLOSE c;
```

### ❌ Mistake 8: Modifying :NEW in AFTER trigger
```sql
-- WRONG: AFTER trigger can't change data
CREATE TRIGGER tr AFTER INSERT ON Bill FOR EACH ROW
BEGIN
    :NEW.Amount := 500;  -- ERROR
END;

-- RIGHT: use BEFORE
CREATE TRIGGER tr BEFORE INSERT ON Bill FOR EACH ROW
BEGIN
    :NEW.Amount := 500;  -- OK
END;
```

### ❌ Mistake 9: Empty exception block
```sql
-- WRONG: hides all errors
EXCEPTION WHEN OTHERS THEN NULL;

-- RIGHT: print error
EXCEPTION WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
    ROLLBACK;
```

### ❌ Mistake 10: Forgetting COMMIT/ROLLBACK
```sql
-- WRONG: data inserted but not committed
BEGIN
    INSERT INTO Bill VALUES (...);
    -- transaction lost if connection drops
END;

-- RIGHT
BEGIN
    INSERT INTO Bill VALUES (...);
    COMMIT;
EXCEPTION WHEN OTHERS THEN ROLLBACK;
END;
```

---

## JDBC Mistakes

### ❌ Mistake 11: Column index starts at 0
```java
// WRONG
int id = rs.getInt(0);

// RIGHT
int id = rs.getInt(1);  // first column is 1
```

### ❌ Mistake 12: Wrong executeXXX method
```java
// WRONG: executeQuery for INSERT
ResultSet rs = stmt.executeQuery("INSERT INTO ...");

// RIGHT
int rows = stmt.executeUpdate("INSERT INTO ...");
```

### ❌ Mistake 13: Closing in wrong order
```java
// WRONG
con.close();
rs.close();
stmt.close();

// RIGHT
rs.close();
stmt.close();
con.close();
```

### ❌ Mistake 14: Forgetting Class.forName
```java
// WRONG: driver not loaded
Connection con = DriverManager.getConnection(...);

// RIGHT
Class.forName("oracle.jdbc.driver.OracleDriver");
Connection con = DriverManager.getConnection(...);
```

### ❌ Mistake 15: Not handling autocommit for transactions
```java
// WRONG: auto-commits after each statement
stmt.executeUpdate("INSERT...");
stmt.executeUpdate("UPDATE...");
// If second fails, first is already committed

// RIGHT
con.setAutoCommit(false);
stmt.executeUpdate("INSERT...");
stmt.executeUpdate("UPDATE...");
con.commit();
```

---

## MongoDB Mistakes

### ❌ Mistake 16: Nested array in $unwind without preserveNull
```javascript
// WRONG: documents with empty array are removed
db.consumer.aggregate([
    { $lookup: { from: "bill", ... as: "bills" }},
    { $unwind: "$bills" }
    // Consumers with no bills are gone
]);

// RIGHT: preserve empty arrays
db.consumer.aggregate([
    { $lookup: { from: "bill", ... as: "bills" }},
    { $unwind: { path: "$bills", preserveNullAndEmptyArrays: true }}
]);
```

### ❌ Mistake 17: Subquery in WHERE without wrapping
```sql
-- WRONG (SQL example but applies to logic)
HAVING count(*) > select avg(cnt) from (...)
         ^^^ missing ()

-- RIGHT
HAVING count(*) > (select avg(cnt) from (...))
```

---

## Exam Day Checklist

```
BEFORE YOU START:
□ Enable SET SERVEROUTPUT ON (for PL/SQL)
□ Check schema: CREATE TABLE if needed
□ INSERT sample data
□ Read the question 2x to catch if it's Pattern 1-8

WHILE CODING:
□ Use correct JOIN (INNER/LEFT based on question)
□ GROUP BY includes ALL non-aggregated columns
□ WHERE for raw columns, HAVING for aggregates
□ COMMIT or ROLLBACK at end of transactions

BEFORE SUBMITTING:
□ No typos in table/column names
□ All cursors CLOSED
□ All exceptions handled (not empty)
□ ResultSet/Statement/Connection closed
□ / at end of PL/SQL block
□ Tested the query (at least dry-run mentally)
```

---

## Time Management

```
Total Exam Time: 3 hours

0:00-0:15  Read all questions, plan approaches (5-7 queries)
0:15-1:00  Pattern 1-5 queries (SQL) - 45 minutes
1:00-1:30  Pattern 6 (MongoDB) - 30 minutes
1:30-2:30  Pattern 7-8 (PL/SQL) - 60 minutes
2:30-3:00  Buffer: debug, recheck, submit - 30 minutes

Per question (SQL): 8-10 minutes
Per question (MongoDB): 8-10 minutes
Per question (PL/SQL): 12-15 minutes
```

---

**Final Note:** You know 8 patterns + 7 edge cases + all syntaxes. You've seen the mistakes. You're ready. Trust your preparation, read carefully, and code systematically. Good luck tomorrow! 🎯
