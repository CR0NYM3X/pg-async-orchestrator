 
# 🧪 DIAMOND LABORATORY: REAL-WORLD OPERATIONS & CRISIS SIMULATOR

Dear team, this laboratory demonstrates the raw power of the `bg` AROF framework operating on real-world data scenarios. It has been updated to include Zero-Day mitigations, Optimistic Locking, and Hardware Governance. Please execute each block in order.

## 🛠️ STEP 0: Environment Setup (Production Sandbox)

We are going to create tables that simulate your company's reality: bank accounts, DBA cleanup logs, and IoT sensor records.

```sql
\c postgres
DROP DATABASE IF EXISTS bk;
CREATE DATABASE bk;
\c bk

-- 1. Create the isolated schema
CREATE SCHEMA IF NOT EXISTS bg_lab;

-- 2. Table for the Backend Developer (Finance)
CREATE TABLE IF NOT EXISTS bg_lab.bank_accounts (
    account_id SERIAL PRIMARY KEY,
    client_name VARCHAR(50),
    balance NUMERIC(15,2)
);

-- 3. Table for the Senior ETL Engineer (Massive Loads)
CREATE TABLE IF NOT EXISTS bg_lab.etl_sales_staging (
    id SERIAL PRIMARY KEY,
    region VARCHAR(50),
    amount NUMERIC(10,2),
    is_processed BOOLEAN DEFAULT FALSE
);

-- 4. Table for QA / Stress Testing (IoT)
CREATE TABLE IF NOT EXISTS bg_lab.iot_sensor_logs (
    log_id SERIAL PRIMARY KEY,
    sensor_id INT,
    reading NUMERIC(5,2),
    recorded_at TIMESTAMP DEFAULT CLOCK_TIMESTAMP()
);

-- 5. Prepare initial baseline data
TRUNCATE TABLE bg_lab.bank_accounts, bg_lab.etl_sales_staging, bg_lab.iot_sensor_logs RESTART IDENTITY CASCADE;

INSERT INTO bg_lab.bank_accounts (client_name, balance) 
VALUES 
    ('Company A', 10000.00), 
    ('Supplier B', 0.00),
    ('Alpha Corporate LLC', 1500000.00),
    ('Mary Johnson', 3450.50),
    ('John Smith', 125.00),
    ('Tech Solutions Inc', 89400.75),
    ('Victoria Ruiz', 45000.20);

INSERT INTO bg_lab.etl_sales_staging (region, amount, is_processed) 
VALUES
    ('North', 12500.00, FALSE),
    ('South', 8400.50, FALSE),
    ('Central', 450.25, TRUE),
    ('West', 32000.00, FALSE),
    ('East', 1500.99, TRUE);

INSERT INTO bg_lab.iot_sensor_logs (sensor_id, reading) 
VALUES
    (101, 24.50),
    (102, 18.25),
    (101, 24.65), 
    (103, 89.10),
    (104, 12.00);

-- Central Vault for Stress Testing (Scenario 9)
INSERT INTO bg_lab.bank_accounts (client_name, balance) VALUES ('Central Vault', 0.00);

SELECT * FROM bg_lab.bank_accounts;

```

---

## 🏛️ SCENARIO 1: The Financial Shield (Role: Backend Developer)

* **Mode:** `SEQUENTIAL_STRICT` (One by one. If one fails, abort all subsequent activities).
* **The Real Case:** We need to transfer $5,000 from Company A to Supplier B. The system subtracts the money from Company A (Step 1). But suddenly, the network fails (Step 2).
* **What is it for?:** Prevents data corruption. If the system wasn't strict, Step 3 would execute and give Supplier B a free $5,000 without completing the validation.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '01_BANK_TRANSFER', 
    p_mode => 'SEQUENTIAL_STRICT', 
    p_queries => ARRAY[
        'UPDATE bg_lab.bank_accounts SET balance = balance - 5000 WHERE client_name = ''Company A'';',
        'SELECT 1 / 0; -- SIMULATED CRITICAL NETWORK FAILURE',
        'UPDATE bg_lab.bank_accounts SET balance = balance + 5000 WHERE client_name = ''Supplier B'';'
    ],
    p_timeout_seconds => 300,
    p_max_retries => 0,
    p_max_parallel_processes => 1,
    p_allocation_policy => 'ADAPTIVE'
);

```

**🔍 How to Validate (Audit):**

```sql
-- 1. Check OS Threads (Should be 0, the orchestrator killed everything instantly)
SELECT pid, query_start, state, query FROM pg_stat_activity WHERE backend_type = 'pg_background' AND pid != pg_backend_pid();

-- 2. Orchestration Audit
SELECT execution_id, job_name, actual_status, completed, errors, pending 
FROM bg.vw_corporate_progress_status WHERE job_name = '01_BANK_TRANSFER';
-- Expected: ❌ ABORTED (STRICT FAILURE). Completed: 1, Errors: 1, Pending: 1.

-- 3. Business Logic Audit
SELECT client_name, balance FROM bg_lab.bank_accounts WHERE client_name IN ('Company A', 'Supplier B');
-- Expected: Supplier B still has $0.00. The DB is safe.

```

---

## 🧹 SCENARIO 2: Operational Continuity (Role: DBA)

* **Mode:** `SEQUENTIAL_NORMAL` (One by one. If one fails, log it and move to the next).
* **The Real Case:** Scheduled historical cleanup. February's table was accidentally deleted yesterday. If the script were strict, the entire maintenance routine would abort.
* **What is it for?:** Absorbs the error, isolates the failure in the forensic box, and forces the engine to finish the remaining non-dependent work.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '02_DBA_MAINTENANCE', 
    p_mode => 'SEQUENTIAL_NORMAL', 
    p_queries => ARRAY[
        'DELETE FROM bg_lab.iot_sensor_logs WHERE log_id <= 3;',
        'DROP TABLE history_table_february; -- ERROR: TABLE DOES NOT EXIST',
        'DELETE FROM bg_lab.etl_sales_staging WHERE is_processed = false;'
    ]
);

```

**🔍 How to Validate (Audit):**

```sql
-- 1. Check the Orchestrator
SELECT * FROM bg.vw_corporate_progress_status WHERE job_name = '02_DBA_MAINTENANCE' LIMIT 1;
-- Expected: ⚠️ COMPLETED WITH ERRORS. Completed: 2, Errors: 1.

-- 2. Check the Forensic Black Box
SELECT task_status, failed_attempt, error_log FROM bg.run_tasks_errors_history;
-- Expected: 'table "history_table_february" does not exist' securely logged.

```

---

## ⚡ SCENARIO 3: Performance Burst (Role: Senior ETL Engineer)

* **Mode:** `CONCURRENT_ORDERED`
* **The Real Case:** Consolidate massive sales calculations for 4 regions.
* **What is it for?:** Injects the workload directly into the server processors in parallel, drastically reducing execution time.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '03_REGIONAL_ETL_CLOSING', 
    p_mode => 'CONCURRENT_ORDERED', 
    p_queries => ARRAY[
        'SELECT pg_sleep(4); -- Processing North Region',
        'SELECT pg_sleep(4); -- Processing South Region',
        'SELECT pg_sleep(4); -- Processing East Region',
        'SELECT pg_sleep(4); -- Processing West Region'
    ],
    p_max_parallel_processes => 4
);

```

**🔍 How to Validate (Live Visual Audit):**

```sql
-- RUN THIS IMMEDIATELY IN TERMINAL 2:
SELECT pid, state, query FROM pg_stat_activity WHERE backend_type = 'pg_background' AND pid != pg_backend_pid();
-- Expected: You will see 5 processes (1 Boss Orchestrator, 4 simultaneous Workers). Parallelism is absolute.

```

---

## 🚦 SCENARIO 4: The Traffic Controller (Role: Data Engineer)

* **Mode:** `RANDOM` with `p_max_parallel_processes` safety limit.
* **The Real Case:** Importing 6 gigantic CSV files. If launched simultaneously, they crash the RAM.
* **What is it for?:** Acts as a safety valve. Limits concurrency strictly to 2 active files at any given time.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '04_CONTROLLED_MASSIVE_LOAD', 
    p_mode => 'RANDOM', 
    p_queries => ARRAY[
        'SELECT pg_sleep(3);', 'SELECT pg_sleep(3);', 'SELECT pg_sleep(3);', 
        'SELECT pg_sleep(3);', 'SELECT pg_sleep(3);', 'SELECT pg_sleep(3);'
    ],
    p_max_parallel_processes => 2 -- SAFETY VALVE
);

```

**🔍 How to Validate (Funnel Audit):**

```sql
-- Run this every second during execution:
SELECT active_workers, pending FROM bg.vw_corporate_progress_status WHERE job_name = '04_CONTROLLED_MASSIVE_LOAD';
-- Expected: `active_workers` will NEVER exceed 2. It takes 9 seconds to process all 6 files safely.

```

---

## 🔪 SCENARIO 5: The Unyielding Executioner (Role: DBA)

* **The Real Case:** A junior analyst executes an infinite Cartesian join that drains the CPU.
* **What is it for?:** Enforces timeouts. The Orchestrator intercepts the rogue thread and sends a SIGINT cancellation directly to the OS.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '05_JUNIOR_ANALYST_REPORT', 
    p_mode => 'SEQUENTIAL_NORMAL', 
    p_queries => ARRAY[
        'WITH RECURSIVE t(n) AS (VALUES (1) UNION ALL SELECT n+1 FROM t) SELECT count(*) FROM t; -- Real infinite loop'
    ],
    p_timeout_seconds => 2, -- KILL IF EXCEEDS 2 SECONDS
    p_max_retries => 0
);

```

**🔍 How to Validate (Forensic Audit):**

```sql
-- Wait 3 seconds:
SELECT status, error_log FROM bg.run_tasks ORDER BY run_task_id DESC LIMIT 1;
-- Expected: FAILED | 'Killed by Parent (Strict Timeout)'

```

---

## 🛡️ SCENARIO 9: The Titanium Shield (Race Condition Mitigation)

* **The Real Case:** 50 concurrent payment webhooks hit the "Central Vault" simultaneously. High OS pressure causes minor stutters. Without Optimistic Locking, the orchestrator might mistakenly retry a successful task, doubling the deposit.
* **What is it for?:** Proves the new `AND status = 'RUNNING'` patch works. Mathematical idempotency guarantees zero double-charges.

**Execution:**

```sql
-- Reset Vault
UPDATE bg_lab.bank_accounts SET balance = 0 WHERE client_name = 'Central Vault';

SELECT bg.launch_job_one_shot(
    p_job_name => '09_TITANIUM_SHIELD_TEST',
    p_mode => 'CONCURRENT_ORDERED',
    p_queries => bg.replicate_query(
        'SELECT pg_sleep(1); UPDATE bg_lab.bank_accounts SET balance = balance + 100 WHERE client_name = ''Central Vault'';',
        50 -- 50 simultaneous transactions of $100
    ),
    p_timeout_seconds => 5,
    p_max_retries => 1,
    p_max_parallel_processes => 10,
    p_allocation_policy => 'ADAPTIVE'
);

```

**🔍 How to Validate (The Ultimate Test):**

```sql
-- Wait for the job to complete, then check the math:
SELECT balance AS "Final Balance", 
       5000.00 AS "Expected Math Balance",
       CASE WHEN balance = 5000.00 THEN '✅ SUCCESS: Zero Phantom Aborts' ELSE '❌ DANGER: Race Condition Detected' END AS "Verdict"
FROM bg_lab.bank_accounts WHERE client_name = 'Central Vault';

-- Check Forensic Logs (Should be 0 rows, no fake OS aborts triggered)
SELECT task_status, failed_attempt, error_log FROM bg.run_tasks_errors_history WHERE run_id = (SELECT max(run_id) FROM bg.run_jobs);

```

---

## 🎛️ SCENARIO 10: Hardware Strangulation & Governance (STRICT vs ADAPTIVE)

* **The Real Case:** Your server runs out of `max_worker_processes`. A critical financial report (`STRICT`) needs to fail-fast and page IT immediately. A background sync (`ADAPTIVE`) can afford to wait until resources free up.
* **What is it for?:** Tests the new Hardware Allocation Policy and the `system_alerts` column.

**Execution (Run all 3 blocks sequentially fast):**

```sql
-- 1. THE DEVOURER (Choke the OS with 30 sleeping processes)
SELECT bg.launch_job_one_shot(
    p_job_name => '10_RESOURCE_DEVOURER',
    p_mode => 'CONCURRENT_ORDERED',
    p_queries => bg.replicate_query('SELECT pg_sleep(10);', 30),
    p_max_parallel_processes => 30,
    p_allocation_policy => 'ADAPTIVE'
);

-- 2. THE STRICT SLA (Will fail immediately because OS is full)
SELECT bg.launch_job_one_shot(
    p_job_name => '10A_URGENT_REPORT_STRICT',
    p_mode => 'SEQUENTIAL_NORMAL',
    p_queries => ARRAY['SELECT pg_sleep(1);'],
    p_allocation_policy => 'STRICT'
);

-- 3. THE FLEXIBLE SURVIVOR (Will patiently wait)
SELECT bg.launch_job_one_shot(
    p_job_name => '10B_BACKGROUND_SYNC_ADAPTIVE',
    p_mode => 'SEQUENTIAL_NORMAL',
    p_queries => ARRAY['SELECT pg_sleep(1);'],
    p_allocation_policy => 'ADAPTIVE'
);

```

**🔍 How to Validate (Governance Audit):**

```sql
-- Immediately check the corporate dashboard:
SELECT job_name, actual_status, system_alerts 
FROM bg.vw_corporate_progress_status 
WHERE job_name LIKE '10%' 
ORDER BY execution_id DESC;

-- Expected Results:
-- 10A_URGENT_REPORT_STRICT: 🛑 ABORTED (HARDWARE LIMIT) | 🛑 ABORTED: Hardware slot limit reached under STRICT policy.
-- 10B_BACKGROUND_SYNC_ADAPTIVE: ⏳ PENDING / INITIALIZING | ⚠️ ADAPTIVE: Running in degraded mode. Hardware slots saturated.
-- 10_RESOURCE_DEVOURER: 🔥 RUNNING (ENGINE ACTIVE) | All systems nominal

```



 
## 📜 SCENARIO 11: Historical Immutability (The John and Robert Case)

* **The Real Crisis:** John creates a Job template. Robert runs it in January. In February, John updates the Job and adds more steps. When querying January's reports, the database shows falsified historical data if the system isn't immutable.
* **What is it for?:** The framework uses **Decoupled Snapshots**. You can alter the templates (`def_jobs`) 100 times, and the historical execution logs will never be corrupted.

**Execution:**
```sql
-- 1. Template is defined (Day 1)
SELECT bg.create_job_definition(
    p_job_name => '07_CASH_CLOSING', 
    p_mode => 'SEQUENTIAL_NORMAL', 
    p_queries => ARRAY['SELECT pg_sleep(1);']
);

-- 2. Job is launched (Saved in history with 1 task)
SELECT bg.launch_job_by_name(p_job_name => '07_CASH_CLOSING');

-- 3. Template is REWRITTEN atomically adding more load (Day 2)
SELECT bg.create_job_definition(
    p_job_name => '07_CASH_CLOSING', 
    p_mode => 'SEQUENTIAL_NORMAL', 
    p_queries => ARRAY['SELECT pg_sleep(1);', 'SELECT pg_sleep(1);', 'SELECT pg_sleep(1);']
);

-- 4. New version is launched
SELECT bg.launch_job_by_name(p_job_name => '07_CASH_CLOSING');

```

**🔍 How to Validate (Audit):**

```sql
SELECT execution_id, job_name, total_tasks, actual_status 
FROM bg.vw_corporate_progress_status 
WHERE job_name = '07_CASH_CLOSING' ORDER BY execution_id DESC;
-- Expected: TWO records with the exact same Job Name. The newest shows `total_tasks: 3`, the old one shows `total_tasks: 1`. History remains mathematically pure.

```

---

## 🚑 SCENARIO 12: The Lifesaver Against Micro-Outages (Role: Integration Engineer)

* **The Real Crisis:** Your database connects to an external Bank API that has network "micro-outages". If your system tries only once and fails, the entire day's financial reports fail.
* **What is it for?:** Tests the `p_max_retries` parameter. If a task fails, the Orchestrator archives the error, returns it to the pending queue, and tries again up to N times.

**Execution:**

```sql
SELECT bg.launch_job_one_shot(
    p_job_name => '08_EXCHANGE_RATE_SYNC', 
    p_mode => 'SEQUENTIAL_NORMAL', 
    p_queries => ARRAY['SELECT 1 / 0; -- Simulating temporary Bank server crash'],
    p_timeout_seconds => 5,
    p_max_retries => 3, -- 🚀 THE LIFESAVER: 1 original attempt + 3 retries
    p_allocation_policy => 'ADAPTIVE'
);

```

**🔍 How to Validate (Forensic Audit):**

```sql
-- 1. Persistence Audit:
SELECT run_task_id, status, attempt AS "Current Attempt", error_log 
FROM bg.run_tasks WHERE run_id = (SELECT max(run_id) FROM bg.run_jobs WHERE job_id = (SELECT job_id FROM bg.def_jobs WHERE job_name = '08_EXCHANGE_RATE_SYNC'));
-- Expected: `Current Attempt` is 4. Status is FAILED. The engine exhausted all its lives.

-- 2. Forensic Black Box Audit:
SELECT task_status, failed_attempt, error_log FROM bg.run_tasks_errors_history ORDER BY history_id DESC LIMIT 3;
-- Expected: Attempts 1, 2, and 3 securely archived in the black box.

```


 



