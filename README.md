# 🕵️ SQL Murder Mystery — *Who Killed the CEO?*  
For the **21 Days SQL Challenge** by **Indian Data Club**

---

## 🧩 **Story / Background**

The CEO of **TechNova Inc.** was found dead in their office on:

**📅 October 15, 2025 — 🕘 9:00 PM**

Your mission: use SQL to uncover **who murdered the CEO**, **when**, **where**, and **how** — using only the company’s internal databases:

- Keycard logs  
- Call records  
- Alibis  
- Physical evidence  
- Employee registry  

All clues are hidden across these tables.

---

## 🗂 **Database Schema**

Tables:  
`employees`, `keycard_logs`, `calls`, `alibis`, `evidence`

> Load the dataset using the included file:  
> **SQL_Murder_Mystery.sql**

---

# 🔍 **Investigation Steps & SQL Queries**

Below are the **6 investigation steps**, each with a SQL query to solve them.

---

🕵️ **Let the investigation begin…**

1️⃣ Step 1 — Where & when did the crime happen?
SELECT *
FROM evidence
WHERE room LIKE '%CEO%'
ORDER BY found_time;

2️⃣ Step 2 — Who entered the CEO’s Office near the murder time?
SELECT k.log_id, k.employee_id, e.name, e.department, e.role,
       k.room, k.entry_time, k.exit_time
FROM keycard_logs k
JOIN employees e ON e.employee_id = k.employee_id
WHERE (k.room LIKE '%CEO%' OR k.room = 'CEO Office')
  AND k.entry_time BETWEEN '2025-10-15 20:30:00' AND '2025-10-15 21:30:00'
ORDER BY k.entry_time;

3️⃣ Step 3 — Who lied about their alibi?
SELECT a.alibi_id, a.employee_id, emp.name, a.claimed_location, a.claim_time
FROM alibis a
JOIN employees emp ON emp.employee_id = a.employee_id
WHERE NOT EXISTS (
    SELECT 1 
    FROM keycard_logs k
    WHERE k.employee_id = a.employee_id
      AND k.room = a.claimed_location
      AND k.entry_time <= a.claim_time
      AND k.exit_time >= a.claim_time
);

4️⃣ Step 4 — Suspicious calls between 20:50–21:00
SELECT c.call_id, ca.name AS caller, re.name AS receiver,
       c.call_time, c.duration_sec
FROM calls c
LEFT JOIN employees ca ON ca.employee_id = c.caller_id
LEFT JOIN employees re ON re.employee_id = c.receiver_id
WHERE c.call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
ORDER BY c.call_time;

5️⃣ Step 5 — Match movement with evidence found
SELECT ev.evidence_id, ev.description, ev.found_time,
       k.employee_id, emp.name, k.entry_time, k.exit_time
FROM evidence ev
LEFT JOIN keycard_logs k 
       ON k.room = ev.room
      AND NOT (k.exit_time < ev.found_time OR k.entry_time > ev.found_time)
LEFT JOIN employees emp ON emp.employee_id = k.employee_id
WHERE ev.room LIKE '%CEO%'
ORDER BY ev.found_time;

6️⃣ Step 6 — Combine suspicious behavior (presence + calls + bad alibi)

MySQL 8.0+ supports CTEs, so this works.

WITH in_office AS (
    SELECT DISTINCT employee_id
    FROM keycard_logs
    WHERE (room LIKE '%CEO%' OR room = 'CEO Office')
      AND entry_time BETWEEN '2025-10-15 20:30:00' AND '2025-10-15 21:30:00'
),
call_window AS (
    SELECT caller_id AS employee_id
    FROM calls
    WHERE call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
    UNION
    SELECT receiver_id
    FROM calls
    WHERE call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
),
bad_alibi AS (
    SELECT employee_id
    FROM alibis a
    WHERE NOT EXISTS (
        SELECT 1 
        FROM keycard_logs k2
        WHERE k2.employee_id = a.employee_id
          AND k2.room = a.claimed_location
          AND k2.entry_time <= a.claim_time
          AND k2.exit_time >= a.claim_time
    )
)
SELECT e.employee_id, e.name, e.department, e.role
FROM employees e
WHERE e.employee_id IN (SELECT employee_id FROM in_office)
  AND e.employee_id IN (SELECT employee_id FROM call_window)
  AND e.employee_id IN (SELECT employee_id FROM bad_alibi);

🕵️ FINAL QUERY — Reveal the Killer (Single Column Output)
SELECT e.name AS killer
FROM employees e
WHERE e.employee_id IN (
  SELECT DISTINCT employee_id
  FROM keycard_logs
  WHERE (room LIKE '%CEO%' OR room = 'CEO Office')
    AND entry_time BETWEEN '2025-10-15 20:30:00' AND '2025-10-15 21:30:00'
)
AND e.employee_id IN (
  SELECT caller_id FROM calls WHERE call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
  UNION
  SELECT receiver_id FROM calls WHERE call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
)
AND e.employee_id IN (
  SELECT employee_id
  FROM alibis a
  WHERE NOT EXISTS (
      SELECT 1 
      FROM keycard_logs k2
      WHERE k2.employee_id = a.employee_id
        AND k2.room = a.claimed_location
        AND k2.entry_time <= a.claim_time
        AND k2.exit_time >= a.claim_time
  )
);

📝 Short Explanation — How We Caught the Killer
Evidence confirmed the murder occurred inside the CEO’s Office at around 9 PM.

Keycard logs narrowed suspects to those who entered the office between 20:30–21:30.

Call logs showed a suspicious call just minutes before the murder.

Alibi comparison exposed a suspect who lied about their whereabouts.

Combining:

presence at the scene,

suspicious phone activity,

and a false alibi
…identified one employee who matched all three red flags.

🎯 Running the final query reveals the murderer.
