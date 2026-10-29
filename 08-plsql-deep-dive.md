# PL/SQL — Deep-Dive Interview Handbook

> Structured as: **Concept → Internals → Code → Real-world Example → Best Practices → Common Mistakes → Follow-up Questions**. Tricky questions flagged 🎯.

---

## Table of Contents
1. PL/SQL Fundamentals & Block Structure
2. Cursors (Implicit, Explicit, Cursor FOR Loops, REF CURSORs)
3. Exception Handling
4. Procedures, Functions & Parameter Modes
5. Packages
6. Triggers
7. Collections (Associative Arrays, Nested Tables, VARRAYs) & Records
8. Bulk Operations (BULK COLLECT, FORALL) & Performance
9. Dynamic SQL
10. Transactions, Locking & Concurrency
11. Views, Materialized Views & Indexes (SQL-adjacent, frequently paired with PL/SQL questions)
12. Performance Tuning & Execution Plans
13. Advanced Oracle Features (Analytic Functions, PIVOT, MERGE, Partitioning basics)
14. Rapid-Fire Tricky Q&A

---

## 1. PL/SQL Fundamentals & Block Structure

### Q1.1 — What is PL/SQL, and why does Oracle extend SQL with a procedural language? Explain the anonymous block structure.

**Answer:** PL/SQL (Procedural Language/SQL) is Oracle's procedural extension to SQL, adding variables, control structures (loops, conditionals), error handling, and modularity (procedures, functions, packages) — capabilities plain SQL (a declarative, set-based language) lacks. It's tightly integrated with SQL (you can embed SQL statements directly in PL/SQL without an API/driver layer) and runs **inside the Oracle database engine** itself, not as external client code.

**Block structure:**
```sql
DECLARE                          -- optional: variable/cursor/exception declarations
    v_salary NUMBER;
    v_name   VARCHAR2(100);
BEGIN                             -- mandatory: executable statements
    SELECT salary, name INTO v_salary, v_name FROM employees WHERE employee_id = 100;
    DBMS_OUTPUT.PUT_LINE(v_name || ' earns ' || v_salary);
EXCEPTION                         -- optional: error handling
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Employee not found');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

**Why run logic inside the database (vs entirely in application code)?**
- **Reduced network round-trips** — a multi-step operation (several queries + logic) executes as ONE call to the database instead of many round-trips between app server and DB, each with network latency.
- **Data integrity enforcement close to the data** — business rules enforced via triggers/constraints/stored procedures can't be bypassed by an application connecting directly with ad-hoc SQL (unlike rules only enforced in application code, which any other client bypassing that app could violate).
- **Performance** — set-based, bulk operations (Section 8) can process large data volumes far more efficiently inside the database than fetching rows to an app server, processing, and sending them back.

**Tradeoffs (the balanced answer interviewers want, not just "PL/SQL is great"):**
- Business logic split between application code and the database makes the system harder to reason about/version-control/test in some organizations' workflows (though this is largely a process/tooling maturity issue, not an inherent PL/SQL flaw).
- Vendor lock-in — PL/SQL is Oracle-specific (unlike ANSI-standard SQL), so heavy PL/SQL usage complicates any future database migration.
- Scaling application logic embedded in the database can be harder than scaling stateless application servers horizontally.

**Best Practice (modern consensus):** Use PL/SQL for what it excels at — set-based bulk data processing, tight data-integrity constraints/triggers, and operations that genuinely benefit from proximity to the data (avoiding costly round-trips) — while keeping general application/business logic in the application tier where it's easier to test, version, and scale independently. Avoid the "everything is a stored procedure" anti-pattern from an earlier era of enterprise development, but also avoid "PL/SQL is legacy, never use it" — for genuinely data-intensive batch/ETL work, it remains the right tool.

**Follow-up:** *"What's the difference between an anonymous block and a named PL/SQL unit (procedure/function/package)?"* → An anonymous block (as above) is compiled and executed once, not stored in the database, and cannot be called by other code — used for ad-hoc scripts/one-off testing. Named units (procedures, functions, packages, triggers) are compiled and **stored** in the database (`CREATE OR REPLACE PROCEDURE ...`), can be invoked repeatedly by other PL/SQL code, application code (via JDBC/drivers), or scheduled jobs, and their compiled form is cached, avoiding recompilation on each call.

---

### Q1.2 — Explain `%TYPE` and `%ROWTYPE` — why are they considered best practice over hardcoded data types?

**Answer:**
- **`%TYPE`** — declares a variable with the **same data type** as a referenced column or another variable, without hardcoding the type explicitly.
- **`%ROWTYPE`** — declares a **record** (a composite variable) with one field per column of a referenced table/view (or cursor), matching structure automatically.

```sql
DECLARE
    v_salary employees.salary%TYPE;        -- automatically matches the column's type/precision
    v_emp    employees%ROWTYPE;             -- one field per column in the employees table
BEGIN
    SELECT * INTO v_emp FROM employees WHERE employee_id = 100;
    v_salary := v_emp.salary * 1.1;
    DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ': ' || v_salary);
END;
```

**Why this matters (the real reason, not just "it's convention"):** If the underlying column's data type/precision changes later (e.g., `salary NUMBER(8,2)` → `NUMBER(10,2)`), code using `%TYPE`/`%ROWTYPE` **automatically stays in sync** without any code change — code using a hardcoded `NUMBER(8,2)` declaration would need to be found and updated manually everywhere it appears, and worse, a mismatch could silently truncate/error on data that no longer fits the hardcoded precision.

**Best Practice:** Always prefer `%TYPE`/`%ROWTYPE` over hardcoded types for variables that mirror table/column structures — reduces maintenance burden and a whole class of type-mismatch bugs.

**Follow-up:** *"Does `%ROWTYPE` include columns added to the table after the code was compiled?"* → No — PL/SQL is compiled, and a `%ROWTYPE` record's structure is fixed at **compile time** based on the table's structure then. If a column is added to the table later, existing compiled code using `%ROWTYPE` won't automatically pick it up until the code is recompiled (which happens automatically the next time it's invoked if Oracle detects the dependency is stale, via automatic dependency invalidation — but the record's field list itself only reflects columns known at that recompilation point).

---

## 2. Cursors

### Q2.1 — Implicit vs Explicit cursors — internals, and why `SELECT INTO` needs explicit exception handling.

**Answer:** A **cursor** is a private SQL work area Oracle uses to execute a SQL statement and hold the result set's context (pointer/position) — every SQL statement in PL/SQL implicitly uses a cursor, but you can also declare one explicitly for more control.

**Implicit cursor:** Automatically created/managed by Oracle for any SQL statement not tied to an explicitly declared cursor (`SELECT INTO`, `INSERT`, `UPDATE`, `DELETE`). Attributes accessible via `SQL%`:
```sql
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 10;
DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' rows updated'); -- SQL%ROWCOUNT, SQL%FOUND, SQL%NOTFOUND, SQL%ISOPEN
```

**`SELECT INTO` and its two mandatory exception cases 🎯:** A `SELECT INTO` expects **exactly one row**. If zero rows match, Oracle raises `NO_DATA_FOUND`; if more than one row matches, it raises `TOO_MANY_ROWS`. **Both must be handled explicitly** in production code — a very common interview trap is forgetting `TOO_MANY_ROWS` (people remember `NO_DATA_FOUND` but not the other).
```sql
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE last_name = 'SMITH'; -- could match 0 or many rows
EXCEPTION
    WHEN NO_DATA_FOUND THEN DBMS_OUTPUT.PUT_LINE('No employee found');
    WHEN TOO_MANY_ROWS THEN DBMS_OUTPUT.PUT_LINE('Multiple employees found — ambiguous');
END;
```

**Explicit cursor:** Declared and controlled manually — needed when processing a **multi-row** result set row by row.
```sql
DECLARE
    CURSOR c_emp IS SELECT employee_id, salary FROM employees WHERE department_id = 10;
    v_emp_id employees.employee_id%TYPE;
    v_salary employees.salary%TYPE;
BEGIN
    OPEN c_emp;
    LOOP
        FETCH c_emp INTO v_emp_id, v_salary;
        EXIT WHEN c_emp%NOTFOUND;              -- MUST check after each fetch, standard pattern
        -- process each row
    END LOOP;
    CLOSE c_emp;                                -- MUST close explicitly — resource leak otherwise
END;
```

**Cursor attributes (`%FOUND`, `%NOTFOUND`, `%ROWCOUNT`, `%ISOPEN`)** — same names for implicit (`SQL%...`) and explicit (`cursor_name%...`) cursors, but referring to the respective cursor's state.

**Cursor FOR loop (the idiomatic, safer shorthand — Oracle implicitly handles OPEN/FETCH/EXIT/CLOSE):**
```sql
BEGIN
    FOR emp_rec IN (SELECT employee_id, salary FROM employees WHERE department_id = 10) LOOP
        DBMS_OUTPUT.PUT_LINE(emp_rec.employee_id || ': ' || emp_rec.salary);
    END LOOP; -- cursor automatically opened, fetched each iteration, and closed at the end (even on exception)
END;
```

**Best Practices:**
- Prefer the **Cursor FOR loop** for simple row-by-row processing — eliminates the risk of forgetting to close a cursor or mishandling `%NOTFOUND`.
- Always explicitly `CLOSE` manually-opened cursors, including in exception handlers (or use a FOR loop, which handles this automatically even on an exception mid-loop).
- Never use `SELECT INTO` without handling both `NO_DATA_FOUND` and `TOO_MANY_ROWS` in production code.

**Common Mistakes:**
- Forgetting `TOO_MANY_ROWS` handling.
- Not closing explicit cursors — leads to `ORA-01000: maximum open cursors exceeded` under load (each session has a max open cursor limit, `OPEN_CURSORS` init parameter).
- Row-by-row explicit cursor processing when a **set-based** SQL statement (a single `UPDATE`/`INSERT ... SELECT`) would achieve the same result far more efficiently — see Q2.2/Section 8 ("row-by-row is slow-by-slow").

**Follow-up 🎯:** *"What's the maximum number of cursors a session can have open, and what causes `ORA-01000`?"* → Governed by the `OPEN_CURSORS` init parameter (a DBA-configurable session-level limit, commonly defaulting to 300-1000 depending on the environment). `ORA-01000` occurs when code repeatedly opens cursors (explicitly, or implicitly via poorly-written dynamic SQL or PL/SQL that opens cursors in a loop) without closing them — a classic resource-leak bug, usually from a missing `CLOSE` in an exception path or a loop that doesn't reuse a single cursor.

---

### Q2.2 — What is a `REF CURSOR`, and how does it differ from a regular cursor? (Very common for app-integration questions.)

**Answer:** A `REF CURSOR` is a **cursor variable** — unlike a static cursor (whose SQL query is fixed at declaration time), a `REF CURSOR` can be associated with **different queries at runtime**, and critically, it can be **passed as a parameter** — most importantly, returned from a stored procedure/function to a **calling client application** (Java/JDBC, .NET, etc.) so the client can iterate the result set, which a regular PL/SQL cursor cannot do (a regular cursor's scope is confined to PL/SQL).

```sql
-- Strongly-typed REF CURSOR (return type fixed to a specific row structure)
TYPE emp_cursor_type IS REF CURSOR RETURN employees%ROWTYPE;

-- Weakly-typed REF CURSOR (SYS_REFCURSOR — the generic, most commonly used form, no fixed return type)
PROCEDURE get_employees_by_dept(p_dept_id IN NUMBER, p_cursor OUT SYS_REFCURSOR) IS
BEGIN
    OPEN p_cursor FOR SELECT * FROM employees WHERE department_id = p_dept_id;
END;
```

**Real-world use case:** A Java/Spring application calling an Oracle stored procedure via JDBC's `CallableStatement`, retrieving a `SYS_REFCURSOR` OUT parameter and mapping it directly to a `ResultSet` in Java — a common integration pattern for encapsulating complex query logic inside the database while still consuming the result set naturally on the application side, without needing the app to know the underlying SQL.

**Follow-up:** *"Does a REF CURSOR need to be explicitly closed?"* → Yes — like any cursor, it holds database resources until closed; when returned to a client application, the client-side driver/framework (e.g., JDBC's `ResultSet`/`Statement.close()`) is typically responsible for closing it once done, and forgetting to do so leaks cursor resources on the database side just like an unclosed explicit cursor.

---

## 3. Exception Handling

### Q3.1 — Predefined vs User-Defined exceptions, `RAISE_APPLICATION_ERROR`, and exception propagation rules.

**Answer:**

**Predefined exceptions** — Oracle automatically raises these for common error conditions, each with a standard name mapped to a specific Oracle error number: `NO_DATA_FOUND` (ORA-01403), `TOO_MANY_ROWS` (ORA-01422), `DUP_VAL_ON_INDEX` (ORA-00001, unique constraint violation), `ZERO_DIVIDE` (ORA-01476), `VALUE_ERROR` (ORA-06502, e.g., converting an invalid string to a number), `INVALID_NUMBER`, `CURSOR_ALREADY_OPEN`, etc.

**User-defined exceptions** — declared and explicitly raised for application-specific error conditions:
```sql
DECLARE
    insufficient_balance EXCEPTION;
    PRAGMA EXCEPTION_INIT(insufficient_balance, -20001); -- optionally associates with a specific error number
    v_balance NUMBER;
BEGIN
    SELECT balance INTO v_balance FROM accounts WHERE account_id = 1001;
    IF v_balance < 100 THEN
        RAISE insufficient_balance;
    END IF;
EXCEPTION
    WHEN insufficient_balance THEN
        DBMS_OUTPUT.PUT_LINE('Balance too low');
END;
```

**`RAISE_APPLICATION_ERROR`** — the standard way to raise a custom error **with a message** back to the calling application (e.g., a Java client catching an `SQLException`), using a reserved error number range **-20000 to -20999** specifically set aside by Oracle for user-defined application errors:
```sql
CREATE OR REPLACE PROCEDURE withdraw(p_account_id NUMBER, p_amount NUMBER) IS
    v_balance NUMBER;
BEGIN
    SELECT balance INTO v_balance FROM accounts WHERE account_id = p_account_id FOR UPDATE;
    IF v_balance < p_amount THEN
        RAISE_APPLICATION_ERROR(-20001, 'Insufficient balance for withdrawal');
    END IF;
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_account_id;
    COMMIT;
END;
```
When called from Java/JDBC, this surfaces as an `SQLException` with error code `20001` and the given message — giving the calling application layer a clean, structured way to detect and handle this specific business-rule violation (rather than a generic/opaque Oracle error).

**Exception propagation rules:**
- An exception raised in a block's executable section, if not caught by a matching `WHEN` clause in that block's own `EXCEPTION` section, **propagates to the enclosing block** — continues propagating outward until caught, or reaches the outermost block/calling application unhandled (terminating the transaction, with any uncommitted DML rolled back by default unless explicitly committed earlier).
- `WHEN OTHERS` catches **any** exception not matched by earlier `WHEN` clauses — should almost always be the **last** handler, and per best practice, should generally **re-raise** (via bare `RAISE;`) after logging, rather than silently swallowing unexpected errors (see Common Mistakes below).

**Best Practices:**
- Handle specific, expected exceptions explicitly (`NO_DATA_FOUND`, custom business exceptions) — don't rely on `WHEN OTHERS` as your primary error-handling strategy.
- When using `WHEN OTHERS`, always log the error (`SQLCODE`, `SQLERRM`, or better, `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` for the actual failing line number) and **re-raise** unless you have a specific, deliberate recovery action — silently swallowing unexpected errors hides real bugs and can leave the caller believing an operation succeeded when it didn't.
- Use `RAISE_APPLICATION_ERROR` with a documented, consistent error-number scheme for application-specific business errors surfaced to calling code.

**Common Mistake 🎯 (one of the most frequently flagged anti-patterns in PL/SQL code review):**
```sql
EXCEPTION
    WHEN OTHERS THEN
        NULL; -- SEVERE ANTI-PATTERN: silently swallows EVERY exception, including ones you never anticipated
END;
```
This hides bugs, makes debugging production issues nearly impossible (no log, no trace), and can leave data in an inconsistent state while the caller believes the operation succeeded. Always at minimum log, and generally re-raise:
```sql
EXCEPTION
    WHEN OTHERS THEN
        log_error(SQLCODE, SQLERRM, DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
        RAISE; -- re-raise so the caller/transaction is correctly aware something failed
END;
```

**Follow-up:** *"What's the difference between `SQLCODE`/`SQLERRM` and `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE`?"* → `SQLCODE` returns the numeric error code, `SQLERRM` returns the error message text for the current exception — but neither tells you **where** (which line) the error actually occurred if it propagated up through several nested calls. `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` (or `DBMS_UTILITY.FORMAT_CALL_STACK`) gives the actual call stack/line-number trace, which is essential for real debugging in production — a senior-level detail many candidates miss.

---

## 4. Procedures, Functions & Parameter Modes

### Q4.1 — Procedures vs Functions, and explain `IN`, `OUT`, `IN OUT` parameter modes in depth.

**Answer:**

| | Procedure | Function |
|---|---|---|
| Returns a value | No (uses OUT params instead, if needed) | Yes — exactly one value via `RETURN` |
| Callable from SQL (e.g., `SELECT func(x) FROM dual`) | No | Yes, if it meets purity rules (see below) |
| Typical use | Perform an action (DML, orchestration) | Compute and return a value |

**Parameter modes:**
- **`IN`** (default) — value passed **into** the procedure/function; read-only inside — cannot be reassigned within the body (compile error if attempted). Passed by reference internally for efficiency, but treated as read-only.
- **`OUT`** — value passed **out**; the parameter starts as `NULL` inside the subprogram regardless of what the caller passed (a common surprise for candidates unfamiliar with this) — must be assigned before the subprogram completes to return meaningful data to the caller.
- **`IN OUT`** — value passed in AND can be modified, with the modified value passed back out.

```sql
CREATE OR REPLACE PROCEDURE adjust_salary(
    p_emp_id   IN  employees.employee_id%TYPE,
    p_percent  IN  NUMBER,
    p_new_salary OUT NUMBER
) IS
BEGIN
    UPDATE employees SET salary = salary * (1 + p_percent/100) WHERE employee_id = p_emp_id
    RETURNING salary INTO p_new_salary; -- RETURNING clause captures the updated value directly, avoiding a second SELECT
END;
```

**🎯 Passing mechanism nuance (a genuinely tricky, frequently-asked distinction):** `IN` parameters are passed **by reference** (a pointer to the actual value, avoiding a copy — efficient, especially for large values like `CLOB`s), but the compiler enforces read-only semantics so this is invisible/safe to the caller. `OUT` and `IN OUT` parameters, by default, are passed **by value** (technically "by reference conceptually but with copy-in/copy-out semantics" — a local copy is used during execution, and only copied back to the caller's actual variable **if the subprogram completes successfully**) — meaning if an **unhandled exception** occurs partway through, an `OUT` parameter's partially-updated value is **NOT** propagated back to the caller (it reverts to whatever it would have been, since the copy-back never happens) — a subtle, real interview trap. You CAN request `NOCOPY` as a compiler hint for pass-by-reference semantics on `OUT`/`IN OUT` params for performance with large values (like big collections), at the cost of losing this "all-or-nothing" copy-back safety guarantee if an exception occurs mid-execution.

**Function purity rules (`SELECT function_name(...) FROM table`):** A function called directly from a SQL statement generally must not perform certain side effects (like DML that would violate read-consistency within the same statement) — Oracle enforces "purity levels" (`WNDS` - Writes No Database State, `RNDS` - Reads No Database State, etc.) especially strictly for functions called from SQL, to avoid inconsistent/unpredictable results within a single query's execution.

**Best Practices:**
- Use functions for computing/returning values (especially ones you want callable directly from SQL); use procedures for actions/orchestration, especially multi-step operations with side effects.
- Prefer `RETURNING INTO` (as above) over a separate follow-up `SELECT` to fetch a just-modified row's values — saves a round-trip/extra query.
- Be deliberate about `NOCOPY` — only use it for large `OUT`/`IN OUT` parameters (big collections, LOBs) where the copy-back performance cost is measurable, understanding the tradeoff around exception-safety.

**Follow-up:** *"Can a function have OUT parameters, and if so, why would that be considered bad practice?"* → Technically allowed, but discouraged — it blurs the function/procedure distinction (a function's whole purpose is to compute and return ONE value via `RETURN`), and a function with `OUT` parameters generally **cannot** be called directly from a SQL statement (`SELECT func(x) FROM dual` doesn't support out params) — defeating one of the main reasons to use a function over a procedure in the first place.

---

## 5. Packages

### Q5.1 — What are Packages, and why are they considered PL/SQL best practice over standalone procedures/functions?

**Answer:** A package bundles related procedures, functions, variables, constants, cursors, and types into a single logical unit, split into a **specification** (public interface) and a **body** (private implementation).

```sql
-- Package Specification (public interface — what callers see)
CREATE OR REPLACE PACKAGE emp_mgmt AS
    g_max_salary CONSTANT NUMBER := 500000; -- package-level constant, publicly visible
    PROCEDURE hire_employee(p_name VARCHAR2, p_dept_id NUMBER);
    FUNCTION get_salary(p_emp_id NUMBER) RETURN NUMBER;
END emp_mgmt;
/

-- Package Body (implementation — hidden from callers)
CREATE OR REPLACE PACKAGE BODY emp_mgmt AS
    v_hire_count NUMBER := 0; -- PRIVATE package-level variable, not visible outside the body

    PROCEDURE validate_dept(p_dept_id NUMBER) IS -- PRIVATE helper procedure, not in the spec, not callable externally
    BEGIN
        -- validation logic
        NULL;
    END;

    PROCEDURE hire_employee(p_name VARCHAR2, p_dept_id NUMBER) IS
    BEGIN
        validate_dept(p_dept_id);
        INSERT INTO employees (first_name, department_id) VALUES (p_name, p_dept_id);
        v_hire_count := v_hire_count + 1; -- persists across calls WITHIN the same session (see below)
    END;

    FUNCTION get_salary(p_emp_id NUMBER) RETURN NUMBER IS
        v_salary NUMBER;
    BEGIN
        SELECT salary INTO v_salary FROM employees WHERE employee_id = p_emp_id;
        RETURN v_salary;
    END;
END emp_mgmt;
/
```

**Why packages are best practice (know these reasons cold — a very common "why" question):**
1. **Encapsulation** — the body can hide implementation details/private helper procedures (like `validate_dept` above) that aren't part of the public contract, exactly like `private` in OOP.
2. **Modularity/organization** — related logic grouped logically, easier to navigate/maintain than dozens of unrelated standalone procedures.
3. **Package-level (session-persistent) state** — variables declared in the package body (or spec, if public) persist for the **duration of the database session**, not just one call — enabling simple in-session caching or state tracking across multiple calls without needing a table (as `v_hire_count` demonstrates above).
4. **Performance — reduced parsing/recompilation overhead**: the **entire package is loaded into memory as a unit** the first time any of its members is called in a session, and subsequent calls to other procedures/functions in the same package don't require separate loading — plus, changing the package **body** doesn't invalidate objects that depend on the package **specification** (as long as the spec itself doesn't change) — a significant advantage over standalone procedures, where any dependent object could be invalidated more broadly.
5. **Overloading** — packages support procedure/function **overloading** (same name, different parameter signatures) — not possible with standalone (schema-level) procedures/functions.

```sql
-- Overloading example — only possible within a package
PROCEDURE process_payment(p_amount NUMBER);
PROCEDURE process_payment(p_amount NUMBER, p_currency VARCHAR2);
```

**Best Practices:**
- Group logically related functionality into packages rather than many standalone top-level procedures.
- Keep the package **specification** stable/minimal — since changing the spec invalidates all dependent objects, while changing only the **body** (with the same spec) does not.
- Use package-level state sparingly and intentionally (understand it's session-scoped, not shared across sessions — a common point of confusion since it can *feel* like shared/global state but isn't across different DB sessions).

**Follow-up 🎯:** *"If two different sessions call the same package's procedure that modifies a package-level variable, do they interfere with each other?"* → No — package state is instantiated **per session** (each session gets its own independent copy of package-level variables) — Session A's changes to `v_hire_count` are completely invisible to Session B. This is a very commonly misunderstood point; package variables are NOT shared/global across all sessions like a static variable might be in some other contexts — they behave more like `ThreadLocal` (Java) conceptually, scoped per session rather than per JVM/globally.

---

## 6. Triggers

### Q6.1 — Types of triggers, `:NEW`/`:OLD`, and the mutating table problem.

**Answer:** A trigger is PL/SQL code automatically executed in response to a specified database event (DML operation, DDL operation, or database-level event like login).

**DML Trigger timing/level combinations:**

| | Row-level (`FOR EACH ROW`) | Statement-level (default, no `FOR EACH ROW`) |
|---|---|---|
| Fires | Once per affected row | Once per triggering statement, regardless of row count |
| Access to `:NEW`/`:OLD` | Yes | No |
| Use case | Per-row validation, audit logging with row-level detail | Statement-level auditing, enforcing statement-level business rules |

```sql
CREATE OR REPLACE TRIGGER trg_salary_audit
    BEFORE UPDATE OF salary ON employees
    FOR EACH ROW
    WHEN (NEW.salary != OLD.salary) -- optional trigger-level filter, evaluated before the trigger body even runs
BEGIN
    INSERT INTO salary_audit (employee_id, old_salary, new_salary, changed_at)
    VALUES (:OLD.employee_id, :OLD.salary, :NEW.salary, SYSDATE);

    IF :NEW.salary > 1000000 THEN
        RAISE_APPLICATION_ERROR(-20002, 'Salary exceeds maximum allowed');
    END IF;
END;
```

**`:NEW` and `:OLD` pseudo-records:**
- `:OLD` — the row's values **before** the change (NULL for all columns in an `INSERT` trigger — there is no "before" state).
- `:NEW` — the row's values **after** the change (NULL for all columns in a `DELETE` trigger — there is no "after" state).
- Both available in `UPDATE` triggers.

**`BEFORE` vs `AFTER` triggers:**
- **`BEFORE`** — fires before the DML actually modifies the row; can **modify `:NEW` values** (e.g., auto-populate an audit column, enforce/correct a value) before it's actually written — commonly used for validation and default-value population.
- **`AFTER`** — fires after the DML has completed; cannot modify `:NEW` (the row is already written) — commonly used for auditing (logging what happened) or triggering downstream actions, since you're guaranteed the change actually succeeded and was applied.

**🎯 The Mutating Table Problem (a classic, frequently-asked trap):** A row-level trigger **cannot query or modify the same table it's defined on** (the "mutating table") during a triggering statement that affects multiple rows — Oracle raises `ORA-04091: table is mutating, trigger/function may not see it` — because the table's state is inconsistent/in-flux mid-statement (some rows already changed, others not yet), so Oracle can't guarantee a consistent, correct view of the table to a query from within that same statement's row-level trigger.

```sql
-- BUGGY — querying the SAME table (employees) the trigger is defined on, from within a row-level trigger
CREATE OR REPLACE TRIGGER trg_check_avg_salary
    BEFORE UPDATE ON employees FOR EACH ROW
DECLARE
    v_avg NUMBER;
BEGIN
    SELECT AVG(salary) INTO v_avg FROM employees; -- ORA-04091 when updating multiple rows in one statement
    IF :NEW.salary > v_avg * 3 THEN
        RAISE_APPLICATION_ERROR(-20003, 'Salary too far above average');
    END IF;
END;
```
**Fixes:**
1. **Compound trigger** (Oracle 11g+) — lets you defer the aggregate check to the `AFTER STATEMENT` timing point, using a package-level (or compound-trigger-scoped) collection to accumulate row-level data during `FOR EACH ROW`, then process it once, safely, in `AFTER STATEMENT` (after the whole statement, and thus the mutating condition, is resolved):
```sql
CREATE OR REPLACE TRIGGER trg_check_avg_salary
FOR UPDATE ON employees
COMPOUND TRIGGER
    TYPE t_ids IS TABLE OF employees.employee_id%TYPE;
    v_changed_ids t_ids := t_ids();

    BEFORE EACH ROW IS
    BEGIN
        v_changed_ids.EXTEND;
        v_changed_ids(v_changed_ids.COUNT) := :NEW.employee_id;
    END BEFORE EACH ROW;

    AFTER STATEMENT IS
        v_avg NUMBER;
    BEGIN
        SELECT AVG(salary) INTO v_avg FROM employees; -- safe now — statement has fully completed
        -- validate collected IDs against v_avg here
    END AFTER STATEMENT;
END trg_check_avg_salary;
```
2. Alternatively, restructure the check as a statement-level trigger, or move complex cross-row validation logic out of a trigger entirely into an explicit stored procedure called deliberately after the DML.

**Best Practices:**
- Keep triggers simple and fast — complex/slow trigger logic on frequently-updated tables directly hurts DML performance for every affected statement.
- Avoid cascading trigger chains (trigger A's DML fires trigger B, which fires trigger C) — very hard to reason about/debug, easy to create infinite loops or unexpected side effects.
- Use triggers primarily for auditing, simple validation, and default-value population — not for complex business logic (which belongs in procedures/packages, invoked deliberately, not implicitly).
- Be very cautious with triggers on tables with high-volume DML (e.g., a `BEFORE INSERT` trigger doing extra queries on every single row insert in a bulk load can catastrophically slow down bulk operations — see Section 8).

**Follow-up:** *"Can you disable a trigger, and why would you need to?"* → `ALTER TRIGGER trigger_name DISABLE;` — commonly needed during bulk data loads/migrations where trigger logic (e.g., audit logging on every row) would be prohibitively slow or semantically inappropriate for a one-time bulk load, then re-enabled afterward. Always understand exactly what a disabled trigger was protecting against/logging before disabling it in production, and re-enable it promptly.

---

## 7. Collections & Records

### Q7.1 — Associative Arrays vs Nested Tables vs VARRAYs — differences and use cases.

**Answer:**

| | Associative Array (Index-By Table) | Nested Table | VARRAY |
|---|---|---|---|
| Storage | PL/SQL memory only (not directly storable in a DB column) | Can be a DB column type; stored as a separate table internally when persisted | Can be a DB column type; stored inline (or in a LOB if large) — has a fixed max size |
| Indexing | Sparse — indexed by integer OR string (`VARCHAR2`), doesn't need to be contiguous, can have "gaps" | Dense initially, but can become sparse after `DELETE`ing individual elements | Dense only — always contiguous, cannot delete individual elements (only trim from the end) |
| Bounds | No fixed upper bound | No fixed upper bound | Fixed maximum size, declared upfront |
| Typical use | In-memory lookup structures/caching within PL/SQL, especially string-indexed for key-value-style lookups | Storing/passing variable-length collections, especially as table column types or bulk-operation intermediate structures | Small, fixed-size, ordered collections (e.g., "12 months of the year" style fixed data) |

```sql
-- Associative Array — very common for in-memory lookup/caching
DECLARE
    TYPE t_salary_map IS TABLE OF NUMBER INDEX BY VARCHAR2(50);
    v_salaries t_salary_map;
BEGIN
    v_salaries('SMITH') := 50000;
    v_salaries('JONES') := 60000;
    IF v_salaries.EXISTS('SMITH') THEN
        DBMS_OUTPUT.PUT_LINE(v_salaries('SMITH'));
    END IF;
END;

-- Nested Table — can be a schema-level type, usable as a table column or bulk-operation structure
CREATE TYPE t_id_list AS TABLE OF NUMBER;
DECLARE
    v_ids t_id_list := t_id_list(101, 102, 103);
BEGIN
    v_ids.EXTEND;
    v_ids(v_ids.COUNT) := 104;
    FOR i IN v_ids.FIRST .. v_ids.LAST LOOP
        DBMS_OUTPUT.PUT_LINE(v_ids(i));
    END LOOP;
END;
```

**Collection methods (commonly tested):** `.COUNT`, `.FIRST`, `.LAST`, `.EXISTS(n)`, `.NEXT(n)`, `.PRIOR(n)`, `.EXTEND`, `.TRIM`, `.DELETE`.

**Records vs Collections — the key distinction:** A **record** is a composite variable representing a single row-like structure with named fields of potentially different types (like a struct) — `%ROWTYPE` produces a record. A **collection** is a set of elements of the **same** type, indexed/ordered — like an array/list. You often combine them: a collection of records (e.g., a nested table of `employees%ROWTYPE`) to represent an in-memory "table" of rows.

```sql
DECLARE
    TYPE t_emp_rec IS RECORD (id NUMBER, name VARCHAR2(100), salary NUMBER);
    TYPE t_emp_list IS TABLE OF t_emp_rec INDEX BY PLS_INTEGER;
    v_employees t_emp_list;
BEGIN
    v_employees(1).id := 100;
    v_employees(1).name := 'Smith';
END;
```

**Follow-up:** *"Why can't you directly use an associative array as a database table column type?"* → Associative arrays exist purely in PL/SQL's session memory (they have no persistent storage representation Oracle can map to a column) — only Nested Tables and VARRAYs (both schema-level object types, `CREATE TYPE ... AS TABLE OF ...` / `CREATE TYPE ... AS VARRAY(n) OF ...`) can be used as actual column types in a table definition, since Oracle needs a defined, storable structure for persistence.

---

## 8. Bulk Operations & Performance — BULK COLLECT & FORALL

### Q8.1 🎯 — Why is row-by-row processing ("slow-by-slow") bad, and how do `BULK COLLECT` and `FORALL` fix it?

**Answer:** One of the most consistently asked PL/SQL performance questions. Every time PL/SQL code executes a SQL statement (even inside a loop), there's a **context switch** between the PL/SQL engine and the SQL engine — these are technically separate engines within the Oracle database, and each switch has real overhead (not free, even though both run "inside" the same database process). A loop issuing one SQL statement per iteration for, say, 100,000 rows incurs **100,000 context switches** — this is the infamous "row-by-row processing, aka slow-by-slow" anti-pattern.

```sql
-- SLOW — row-by-row, 100,000 individual INSERT statements, 100,000 context switches
BEGIN
    FOR i IN 1..100000 LOOP
        INSERT INTO order_archive SELECT * FROM orders WHERE order_id = i;
    END LOOP;
END;
```

**`BULK COLLECT`** — fetches **multiple rows in a single context switch** from SQL into a PL/SQL collection, instead of one row at a time:
```sql
DECLARE
    TYPE t_id_tab IS TABLE OF orders.order_id%TYPE;
    v_ids t_id_tab;
BEGIN
    SELECT order_id BULK COLLECT INTO v_ids FROM orders WHERE status = 'PENDING';
    -- one single context switch fetched ALL matching rows into v_ids at once
END;
```
**With `LIMIT` (critical for large result sets — avoid loading millions of rows into PL/SQL memory at once):**
```sql
DECLARE
    CURSOR c_orders IS SELECT order_id FROM orders WHERE status = 'PENDING';
    TYPE t_id_tab IS TABLE OF orders.order_id%TYPE;
    v_ids t_id_tab;
BEGIN
    OPEN c_orders;
    LOOP
        FETCH c_orders BULK COLLECT INTO v_ids LIMIT 1000; -- process in batches of 1000, bounding memory usage
        EXIT WHEN v_ids.COUNT = 0;
        -- process this batch of 1000
    END LOOP;
    CLOSE c_orders;
END;
```

**`FORALL`** — the write-side counterpart: sends multiple `INSERT`/`UPDATE`/`DELETE` statements (from a collection's contents) to the SQL engine in a **single** context switch, instead of one DML statement per loop iteration:
```sql
DECLARE
    TYPE t_id_tab IS TABLE OF orders.order_id%TYPE;
    v_ids t_id_tab;
BEGIN
    SELECT order_id BULK COLLECT INTO v_ids FROM orders WHERE status = 'PENDING';
    FORALL i IN v_ids.FIRST .. v_ids.LAST
        UPDATE orders SET status = 'PROCESSED' WHERE order_id = v_ids(i);
    -- ALL updates sent to the SQL engine in ONE context switch, not v_ids.COUNT separate ones
    COMMIT;
END;
```

**`FORALL` error handling — `SAVE EXCEPTIONS`:** By default, `FORALL` stops entirely on the first error within the batch. To process as many as possible and collect errors for rows that failed (without aborting the whole batch), use `SAVE EXCEPTIONS`:
```sql
BEGIN
    FORALL i IN v_ids.FIRST .. v_ids.LAST SAVE EXCEPTIONS
        UPDATE orders SET status = 'PROCESSED' WHERE order_id = v_ids(i);
EXCEPTION
    WHEN OTHERS THEN
        FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
            DBMS_OUTPUT.PUT_LINE('Error at index ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
                                  ': ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE));
        END LOOP;
END;
```

**Performance impact (a number worth quoting in an interview to show real understanding):** Converting row-by-row loops to `BULK COLLECT`/`FORALL` commonly yields **10x-100x performance improvements** for large-volume operations, purely from eliminating the context-switch overhead — this is one of the single highest-leverage PL/SQL performance optimizations and a near-guaranteed discussion point in any PL/SQL performance interview question.

**Best Practices:**
- Always use `LIMIT` with `BULK COLLECT` for potentially large result sets — unbounded `BULK COLLECT` on millions of rows can exhaust PGA/session memory (`ORA-04030` or similar).
- Use `FORALL` for any bulk DML derived from a collection, rather than a `FOR` loop issuing individual DML statements.
- Where possible, prefer a **single set-based SQL statement** (a plain `UPDATE ... WHERE ...` or `INSERT ... SELECT`) over BULK COLLECT/FORALL entirely, when the logic doesn't genuinely need row-by-row PL/SQL processing — set-based SQL is generally even faster than BULK COLLECT/FORALL for cases where you don't actually need procedural per-row logic in between the read and write.

**Follow-up:** *"If a set-based single UPDATE statement can do the same thing, why ever use BULK COLLECT/FORALL instead?"* → When there's genuine procedural logic needed per row that can't be expressed in pure SQL (complex conditional business rules, calling other procedures/functions per row, row-by-row validation with early-exit conditions) — BULK COLLECT/FORALL gives you the performance benefits of batching while still allowing PL/SQL logic between the fetch and the write, which a single set-based SQL statement can't accommodate for sufficiently complex per-row logic.

---

## 9. Dynamic SQL

### Q9.1 — `EXECUTE IMMEDIATE` vs static SQL — when is dynamic SQL necessary, and how do you prevent SQL injection in PL/SQL?

**Answer:** **Dynamic SQL** builds and executes a SQL/PL-SQL statement as a **string at runtime**, needed when the statement's structure (table name, column list, WHERE clause shape) isn't known until runtime — something static SQL (fixed at compile time) can't express.

```sql
CREATE OR REPLACE PROCEDURE archive_table(p_table_name VARCHAR2, p_cutoff_date DATE) IS
    v_sql VARCHAR2(500);
BEGIN
    -- Table names cannot be bind variables (Oracle doesn't allow binding identifiers, only values) —
    -- dynamic SQL is REQUIRED here since the table name varies at runtime
    v_sql := 'INSERT INTO archive_' || p_table_name || ' SELECT * FROM ' || p_table_name ||
             ' WHERE created_date < :cutoff';
    EXECUTE IMMEDIATE v_sql USING p_cutoff_date; -- bind variable used for the actual VALUE
END;
```

**With bind variables and OUT parameters:**
```sql
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
    INTO v_salary
    USING p_emp_id;

EXECUTE IMMEDIATE 'UPDATE employees SET salary = :new_sal WHERE employee_id = :id RETURNING salary INTO :out_sal'
    USING p_new_salary, p_emp_id
    RETURNING INTO v_result;
```

**🎯 SQL Injection risk — a critical, very commonly-probed security question:** Dynamic SQL built via **string concatenation of user input** (rather than bind variables) is vulnerable to SQL injection, exactly like in any other language:
```sql
-- VULNERABLE — user input concatenated directly into the SQL string
v_sql := 'SELECT * FROM employees WHERE last_name = ''' || p_user_input || '''';
EXECUTE IMMEDIATE v_sql;
-- if p_user_input = ''' OR ''1''=''1', the WHERE clause becomes always-true, exposing all rows
-- worse inputs could chain additional statements or exploit further
```
**Fix — always use bind variables for VALUES, never concatenate user-supplied data directly into the SQL string:**
```sql
v_sql := 'SELECT * FROM employees WHERE last_name = :name';
EXECUTE IMMEDIATE v_sql INTO ... USING p_user_input; -- safe — bind variable, not string concatenation
```
**For identifiers (table/column names) that genuinely must be dynamic** (bind variables can't parameterize identifiers, only values) — use `DBMS_ASSERT` functions (`DBMS_ASSERT.SIMPLE_SQL_NAME`, `DBMS_ASSERT.SQL_OBJECT_NAME`) to validate/sanitize the identifier string is a legitimate, existing database object name before concatenating it into dynamic SQL, rather than trusting raw user input directly.

**Native Dynamic SQL (`EXECUTE IMMEDIATE`) vs `DBMS_SQL` package:**
- `EXECUTE IMMEDIATE` — simpler syntax, sufficient for the vast majority of dynamic SQL needs (including when the number of bind variables/columns is known, even if the actual SQL text varies).
- `DBMS_SQL` — a lower-level API needed for genuinely dynamic cases where even the **number of bind variables or columns is unknown until runtime** (fully dynamic column lists), or when you need to describe a cursor's structure programmatically — more verbose, used less often, reserved for these advanced cases.

**Best Practices:**
- Prefer static SQL whenever the statement structure is fully known at compile time — static SQL benefits from compile-time syntax/semantic checking, dependency tracking, and generally simpler, more efficient execution plans caching.
- Always use bind variables for VALUES in dynamic SQL — never string-concatenate user-controllable values.
- Use `DBMS_ASSERT` to validate dynamically-constructed identifiers (table/column names) when they must come from external/user input.

**Follow-up:** *"Besides SQL injection risk, what's another downside of overusing dynamic SQL?"* → **Shared pool/cursor cache inefficiency** — each distinct SQL text string (even differing only by a literal value that should have been a bind variable) is parsed and cached as a **separate** entry in Oracle's shared pool (library cache), rather than reusing a single cached execution plan for parameterized variations of "the same" query — this bloats the shared pool and causes excessive hard-parsing overhead if dynamic SQL is built with literals instead of bind variables repeatedly for what is logically the same query shape (directly related to the SQL injection fix above — using bind variables solves BOTH the security issue AND this performance/cursor-sharing issue simultaneously).

---

## 10. Transactions, Locking & Concurrency

### Q10.1 — COMMIT/ROLLBACK/SAVEPOINT, autonomous transactions, and Oracle's locking/concurrency model.

**Answer:**

**Transaction control:**
```sql
BEGIN
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    SAVEPOINT before_credit;
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

    IF some_validation_fails THEN
        ROLLBACK TO before_credit; -- undoes only the second update, first update's debit remains pending
    END IF;

    COMMIT; -- makes ALL changes since the last commit permanent
END;
```

**Autonomous Transactions (`PRAGMA AUTONOMOUS_TRANSACTION`) — a frequently-asked, distinctive PL/SQL feature:** Allows a nested block/procedure to commit or roll back **independently** of the calling (outer) transaction — the inner transaction's commit/rollback doesn't affect, and isn't affected by, the outer transaction's eventual outcome.

```sql
CREATE OR REPLACE PROCEDURE log_error(p_message VARCHAR2) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO error_log (message, logged_at) VALUES (p_message, SYSDATE);
    COMMIT; -- commits ONLY the log insert, independent of whatever happens in the calling transaction
END;
```
**Real-world use case:** Error/audit logging that must persist **even if the main business transaction later rolls back** — e.g., logging "payment attempt failed" should survive even though the failed payment transaction itself rolls back; without an autonomous transaction, the log entry would be rolled back along with everything else in the same transaction.

**🎯 Important caveat:** Autonomous transactions run as a genuinely separate transaction — this means they **cannot see uncommitted changes** made by the calling (outer) transaction (since from the autonomous transaction's perspective, that data isn't committed yet) — a common source of confusion/bugs if misused for something other than its intended independent-logging-style use case.

**Locking model — Oracle's approach vs many other databases (a strong differentiator worth knowing):** Oracle uses **row-level locking** with **multi-version concurrency control (MVCC)** by default — readers **never block writers, and writers never block readers** (a very important, frequently-highlighted Oracle characteristic vs some other databases' default locking behavior). When a transaction reads data, Oracle constructs a **read-consistent view** using **undo segments** — if another transaction is concurrently modifying the same rows (uncommitted), the reading transaction sees the data **as it was before** that uncommitted change, not a lock-wait or a dirty read.

**`SELECT ... FOR UPDATE`** — explicitly locks selected rows, preventing other sessions from modifying (or also locking) them until the locking transaction commits/rolls back — used when you need to read a value and then update it based on that read, ensuring no other session changes it in between (avoiding a lost-update race condition):
```sql
SELECT balance INTO v_balance FROM accounts WHERE account_id = 1001 FOR UPDATE; -- locks this row
-- ... business logic using v_balance ...
UPDATE accounts SET balance = v_balance - 100 WHERE account_id = 1001;
COMMIT; -- releases the lock
```
`FOR UPDATE NOWAIT` — fails immediately with `ORA-00054` if the row is already locked, instead of waiting; `FOR UPDATE WAIT n` — waits up to `n` seconds before failing; `SKIP LOCKED` (12c+) — skips already-locked rows entirely, useful for queue-processing patterns (multiple workers pulling from a shared queue table without blocking each other on already-claimed rows).

**Best Practices:**
- Keep transactions as short as possible — long-held locks (especially from `FOR UPDATE` without a prompt commit) block other sessions and hurt concurrency.
- Use autonomous transactions sparingly and specifically for logging/auditing use cases — not as a general workaround for transaction-scoping problems, since overuse can create hard-to-reason-about partial-commit states.
- Prefer Oracle's default MVCC-based read consistency over explicit locking wherever possible — only use `FOR UPDATE` when you specifically need to prevent a concurrent modification between your read and your subsequent write (the classic read-modify-write race condition).

**Follow-up 🎯:** *"If Oracle's readers never block writers and writers never block readers, how does it prevent a 'lost update' — where two sessions read the same row and both write back based on stale data?"* → MVCC prevents **dirty reads** (seeing uncommitted data) and provides read consistency, but it does NOT by itself prevent lost updates from concurrent read-then-write logic — that specific problem requires either explicit locking (`SELECT ... FOR UPDATE`, as above) to serialize the read-modify-write sequence, or an optimistic concurrency approach (a `version`/`last_updated` column checked in the `WHERE` clause of the final `UPDATE`, similar to JPA's `@Version` mechanism, failing/retrying if the row changed since it was read) — MVCC solves consistent reads, not the separate lost-update race condition, and conflating the two is a common interview mistake to avoid.

---

## 11. Views, Materialized Views & Indexes

### Q11.1 — Regular Views vs Materialized Views — and when do you use each?

**Answer:**

| | Regular (Simple) View | Materialized View |
|---|---|---|
| Data storage | None — just a stored query definition; executed fresh every time it's queried | Physically stores the query's result set on disk |
| Freshness | Always current (reflects underlying tables in real time) | Can be stale until refreshed (`ON DEMAND` or `ON COMMIT` or scheduled) |
| Query performance | Same cost as running the underlying query each time | Fast — reads pre-computed data directly, no need to re-run potentially expensive joins/aggregations |
| Use case | Simplifying complex queries, security (restricting visible columns/rows), abstraction layer over underlying tables | Expensive aggregate/summary queries (reporting/dashboards), data warehousing, reducing load on OLTP tables from heavy analytical queries |

```sql
CREATE MATERIALIZED VIEW mv_daily_sales
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND     -- or: REFRESH FAST ON COMMIT (needs a materialized view log on base tables)
AS
SELECT order_date, SUM(amount) AS total_sales, COUNT(*) AS order_count
FROM orders
GROUP BY order_date;

-- Manually refresh:
EXEC DBMS_MVIEW.REFRESH('MV_DAILY_SALES', 'C'); -- 'C' = complete refresh; 'F' = fast (incremental) refresh
```

**`REFRESH FAST` vs `REFRESH COMPLETE`:** Complete refresh re-executes the entire defining query from scratch (simple but expensive for large result sets). Fast refresh applies only the **incremental changes** since the last refresh (using a **materialized view log** on the base table(s), which tracks changed rows) — far more efficient for large materialized views with relatively small incremental changes, but requires the MV log to be set up and the query to meet fast-refresh eligibility requirements (certain query shapes/aggregate types are supported, others aren't).

**Updatable views and the `WITH CHECK OPTION`:** A view can sometimes be used for `INSERT`/`UPDATE`/`DELETE` (if it meets certain conditions — typically simple, single-table, no aggregates/DISTINCT/GROUP BY). `WITH CHECK OPTION` ensures any row inserted/updated through the view still satisfies the view's own `WHERE` clause (preventing you from inserting/updating a row through the view such that it would then no longer appear in that view — a subtle but sometimes-asked correctness detail).

**Best Practices:**
- Use materialized views for expensive, frequently-repeated aggregate queries (dashboards, reports) where slightly-stale data is acceptable — dramatically reduces load on source tables and query latency for consumers.
- Set up materialized view logs and use `REFRESH FAST` where the query pattern supports it, rather than `REFRESH COMPLETE`, for large MVs refreshed frequently.
- Use regular views for security/abstraction purposes (hiding sensitive columns, simplifying a complex join for downstream consumers) where always-fresh data is required and the underlying query isn't prohibitively expensive.

**Follow-up:** *"What's Query Rewrite, and how does it relate to materialized views?"* → Oracle's optimizer can **automatically** redirect a query against the base tables to instead use a matching materialized view (if one exists covering the needed data, and query rewrite is enabled) **without the querying application even knowing the MV exists** — transparently speeding up ad-hoc queries/reports that happen to match (or be coverable by) a pre-aggregated MV, a powerful data-warehousing optimization technique.

---

## 12. Performance Tuning & Execution Plans

### Q12.1 — How do you diagnose and tune a slow PL/SQL procedure / slow SQL statement in production?

**Answer:**

**`EXPLAIN PLAN` / `DBMS_XPLAN`** — shows the optimizer's chosen execution plan (how it intends to execute a query — full table scan vs index access, join order, join method) **without actually running the query**:
```sql
EXPLAIN PLAN FOR SELECT * FROM orders WHERE customer_id = 123 AND status = 'PENDING';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**`DBMS_XPLAN.DISPLAY_CURSOR`** (with `gather_plan_statistics` hint or `STATISTICS_LEVEL=ALL`) — shows the **actual** execution plan **after** running the query, including real row counts/timing at each step (vs the optimizer's mere *estimates* in a plain `EXPLAIN PLAN`) — critical for finding where the optimizer's estimates diverge significantly from reality (a strong signal of stale statistics or a cardinality misestimation causing a poor plan choice):
```sql
SELECT /*+ gather_plan_statistics */ * FROM orders WHERE customer_id = 123;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

**Key execution plan red flags to look for:**
- **Full Table Scan** on a large table where an index-based access path would be expected — often indicates a missing index, a function applied to an indexed column preventing index use (same issue as covered in the SQL section of the Java handbook), or the optimizer (correctly) determining a full scan is actually cheaper (e.g., retrieving a large fraction of the table's rows).
- **Large discrepancy between Estimated and Actual row counts** — signals stale or missing statistics (`DBMS_STATS.GATHER_TABLE_STATS`) causing the optimizer to choose a suboptimal plan based on wrong assumptions about data volume/distribution.
- **Nested Loop join on very large row sets** — nested loops are efficient when the outer row set is small, but can be catastrophic (essentially O(n×m)) if the optimizer misjudges cardinality and picks nested loops for two large tables where a Hash Join would be far more efficient.

**`DBMS_STATS`** — gathering accurate, current statistics is one of the single most impactful things a developer/DBA can do for query performance, since Oracle's cost-based optimizer relies entirely on statistics to choose good plans:
```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'ORDERS', cascade => TRUE); -- cascade also gathers index stats
```

**Bind variable peeking / cursor sharing:** Oracle can reuse a cached execution plan for a parameterized (bind-variable) query across executions with different bind values — usually a performance win (avoids repeated hard-parsing) but can occasionally cause a **suboptimal plan** to be reused for a bind value with very different data-skew characteristics than the value used when the plan was first cached ("bind variable peeking" issue) — Oracle's **Adaptive Cursor Sharing** (11g+) mitigates this by allowing multiple plans to be cached for the same statement when bind-sensitive skew is detected.

**AWR / ASH (Automatic Workload Repository / Active Session History)** — for diagnosing production-wide performance issues (top SQL by elapsed time/CPU/I/O, wait event analysis) beyond a single statement's execution plan — an enterprise-licensed diagnostic/performance tuning toolset DBAs commonly use for broader system-level tuning.

**Best Practices:**
- Keep table/index statistics current (scheduled `DBMS_STATS` jobs, Oracle's automatic stats-gathering job).
- Use bind variables (Section 9) — both for injection safety AND for allowing Oracle to reuse cached execution plans instead of hard-parsing a new plan for every literal-value variation of "the same" query.
- Index columns used in `WHERE`/`JOIN`/`ORDER BY` per the same general indexing principles as any RDBMS — composite index column order matters (leftmost-prefix rule, same as covered generally in relational database indexing).
- Avoid applying functions to indexed columns in `WHERE` clauses unless a matching function-based index exists.

**Follow-up:** *"What's the difference between a hard parse and a soft parse, and why does it matter for performance?"* → A **hard parse** fully parses, validates, and generates a new execution plan for a SQL statement not already in the shared pool/library cache — CPU-intensive and involves latches/locks that can become a serious contention bottleneck under high concurrency. A **soft parse** reuses an already-cached, matching statement's parsed form/execution plan (found via a hash match on the exact SQL text) — much cheaper. Using bind variables instead of literals means logically-identical queries share the *same* cached SQL text/plan (enabling soft parses), whereas literal-embedded SQL creates a distinct cache entry per distinct literal value, forcing unnecessary hard parses — directly connecting back to the dynamic SQL/bind-variable discussion in Section 9.

---

## 13. Advanced Oracle Features

### Q13.1 — Analytic (Window) Functions, `MERGE`, and `PIVOT` — with practical examples.

**Answer:**

**Analytic/Window functions** — compute a value across a set of rows related to the current row, **without collapsing the result set** into groups (unlike `GROUP BY` aggregates):
```sql
SELECT employee_id, department_id, salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_salary_rank,
       AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary,
       SUM(salary) OVER (ORDER BY employee_id ROWS UNBOUNDED PRECEDING) AS running_total,
       LAG(salary) OVER (PARTITION BY department_id ORDER BY hire_date) AS prev_hire_salary
FROM employees;
```
- `RANK()`/`DENSE_RANK()`/`ROW_NUMBER()` — ranking, with `RANK` leaving gaps after ties, `DENSE_RANK` not leaving gaps, `ROW_NUMBER` assigning a unique sequential number regardless of ties.
- `LAG`/`LEAD` — access a prior/subsequent row's value without a self-join.
- `PARTITION BY` — like a `GROUP BY` but for the window function's scope only, without collapsing rows.

**`MERGE`** — combines `INSERT` and `UPDATE` (upsert) logic into a single statement, commonly used for data synchronization/ETL:
```sql
MERGE INTO target_accounts t
USING staging_accounts s ON (t.account_id = s.account_id)
WHEN MATCHED THEN
    UPDATE SET t.balance = s.balance, t.updated_at = SYSDATE
    WHERE t.balance != s.balance -- optional: only update if actually different
WHEN NOT MATCHED THEN
    INSERT (account_id, balance, created_at) VALUES (s.account_id, s.balance, SYSDATE);
```

**`PIVOT`** — transforms rows into columns (common for reporting-style transformations):
```sql
SELECT * FROM (
    SELECT department_id, TO_CHAR(hire_date, 'MON') AS hire_month, salary FROM employees
)
PIVOT (
    SUM(salary) FOR hire_month IN ('JAN' AS jan, 'FEB' AS feb, 'MAR' AS mar)
);
```

**Best Practices:**
- Prefer analytic functions over self-joins/correlated subqueries for ranking/running-total/comparison-to-group-aggregate style requirements — generally far more efficient (single pass) than the equivalent self-join approach.
- Use `MERGE` for upsert-style ETL logic rather than a manual "check if exists, then INSERT or UPDATE" pattern (which requires two statements and is subject to race conditions under concurrency without additional locking).

**Follow-up:** *"What's the difference between a regular aggregate function and the same function used as an analytic/window function?"* → A regular aggregate (`GROUP BY`) collapses each group into a single output row. The same function used with an `OVER()` clause (analytic form) computes the aggregate **per partition** but **retains every original row** in the output, attaching the computed aggregate value alongside each row's own detail columns — enabling row-level detail alongside group-level context in a single result set, which a plain `GROUP BY` cannot do without a self-join.

---

## 14. Rapid-Fire Tricky Q&A

**Q: Why does `IF v_value = NULL THEN ... END IF;` never evaluate to true, even if `v_value` genuinely is NULL?**
A: In SQL/PL-SQL, `NULL` represents "unknown," and any comparison involving `NULL` using `=` or `!=` evaluates to `NULL` (neither true nor false) — not `TRUE`, even when comparing `NULL = NULL`. You must use `IS NULL`/`IS NOT NULL` for NULL checks — a classic, near-universal SQL/PL-SQL trap frequently tested in interviews.

**Q: What's the difference between `%NOTFOUND` immediately after `OPEN` (before any `FETCH`) — what value does it have?**
A: `cursor%NOTFOUND` is `NULL` before the first `FETCH` (cursor attributes are undefined/NULL until at least one fetch has occurred) — checking it before any fetch is a logic error that can silently behave unexpectedly (since `NULL` in a boolean/`EXIT WHEN` context is treated as not-true, the loop would continue rather than exit, but relying on this is fragile and considered poor practice — always structure loops with `FETCH` before checking `%NOTFOUND`).

**Q: Can a function called from a SQL `WHERE` clause perform a `COMMIT`?**
A: No — a function invoked from within a SQL statement generally cannot execute DDL, `COMMIT`, or `ROLLBACK` (violates the "purity rules" mentioned in Section 4) — Oracle raises `ORA-14551: cannot perform a DML operation inside a query` or similar, specifically to prevent a query from having unpredictable side effects on the very data (and transaction state) it's reading, mid-execution. (An autonomous transaction *inside* that function is the standard workaround if a commit is genuinely needed from such a context, e.g., for logging — Section 10.)

**Q: Why might `DELETE FROM large_table;` (no WHERE clause, deleting all rows) be much slower than `TRUNCATE TABLE large_table;`, and what's the tradeoff?**
A: `DELETE` is a DML operation — logged row-by-row in the redo/undo logs (enabling rollback and point-in-time recovery), and fires any `DELETE` triggers per row — expensive for large tables. `TRUNCATE` is DDL — deallocates the table's storage extents directly, minimally logged, doesn't fire triggers, and is dramatically faster — but it **cannot be rolled back** (it implicitly commits) and resets the high-water mark, and any `ON DELETE` triggers/referential-integrity `ON DELETE CASCADE` expectations tied to row-level DELETE won't apply the same way. Choose `TRUNCATE` for genuinely clearing an entire table (e.g., a staging table between ETL runs) where these tradeoffs are acceptable; use `DELETE` (with a `WHERE` clause or not) when transactional rollback capability or trigger-firing is required.

**Q: What happens if you don't explicitly `COMMIT` at the end of an anonymous PL/SQL block executed via SQL*Plus/SQL Developer, and you then disconnect?**
A: Uncommitted changes are **rolled back automatically** on session disconnect (whether graceful or abrupt) — this is standard transactional behavior; only explicitly committed changes (or those from an auto-commit context, e.g., many GUI tools default to auto-commit per statement, which is itself a common source of confusion/bugs when someone assumes explicit COMMIT is happening when the tool is silently doing it after every statement) persist.

**Q: Why is `WHEN OTHERS THEN RAISE;` (bare RAISE, no arguments) different from `WHEN OTHERS THEN RAISE some_other_exception;`?**
A: A bare `RAISE;` inside an exception handler **re-raises the original exception**, preserving its original error code, message, and (critically) its error stack/backtrace information for accurate diagnosis further up the call chain. Explicitly raising a *different* named exception discards the original exception's specific identity/context (unless you explicitly capture and log `SQLCODE`/`SQLERRM`/backtrace before doing so) — generally, prefer bare `RAISE;` when merely propagating an unexpected error after logging it, reserving explicit re-raising of a different exception for cases where you're deliberately translating a low-level error into a more meaningful, higher-level application exception for the caller.

---

## Quick Self-Check Before Your Interview

- [ ] Can you explain the mutating table problem and demonstrate the compound trigger fix?
- [ ] Can you explain exactly why BULK COLLECT/FORALL are faster, using the "context switch" reasoning, not just "it's faster, trust me"?
- [ ] Can you explain package-level state and confirm it's session-scoped, not globally shared?
- [ ] Can you explain the difference between `IN`, `OUT`, and `IN OUT` parameter passing semantics, including the exception-rollback nuance for OUT params?
- [ ] Can you explain Oracle's MVCC/read-consistency model and why it doesn't by itself prevent lost updates?
- [ ] Can you write a safe piece of dynamic SQL using bind variables, and explain the SQL injection risk of not doing so?
- [ ] Can you explain the difference between a materialized view's `REFRESH FAST` and `REFRESH COMPLETE`?
- [ ] Do you have a real project story about a slow PL/SQL procedure you diagnosed and optimized (ideally involving BULK COLLECT/FORALL or an execution plan fix)?
