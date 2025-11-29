# 🕵️‍♂️ SQL Murder Mystery — Who Killed the CEO?
🗂️ 21-Days SQL Challenge Project- by **Indian Data Club**

## Background Story

The CEO of TechNova Inc. was found dead in their office on
📅 October 15, 2025
🕘 9:00 PM

I have been assigned as the lead data analyst to solve the case using SQL.
All clues are hidden across several tables in the company database:

🔑 keycard logs

📞 call records

🕒 employee alibis

🧤 evidence from different rooms

Mission:

Find out who killed the CEO, how, where, and when — using only SQL.

## 🗃️ Database Schema

The database (via the provided SQL file) contains 4 tables: employees, calls, alibis, evidence

## 🕵️ Let the investigation begin…  

#### 🔎 1️⃣ Step 1 — Where & when did the crime happen?  
```
SELECT *
FROM evidence
WHERE room LIKE '%CEO%'
ORDER BY found_time;
```
#### 🚪 2️⃣ Step 2 — Who accessed the CEO’s Office around the murder time?
```
SELECT k.log_id, k.employee_id, e.name, e.department, e.role,
       k.room, k.entry_time, k.exit_time
FROM keycard_logs k
JOIN employees e ON e.employee_id = k.employee_id
WHERE k.room LIKE '%CEO%'
  AND k.entry_time BETWEEN '2025-10-15 20:30:00' AND '2025-10-15 21:30:00'
ORDER BY k.entry_time;
```
#### 🕒 3️⃣ Step 3 — Who lied about their alibi?
```
SELECT 
    a.alibi_id, 
    a.employee_id, 
    emp.name, 
    a.claimed_location, 
    a.claim_time,
    k.room AS actual_room,
    k.entry_time AS actual_entry_time,
    k.exit_time AS actual_exit_time
FROM alibis a
JOIN employees emp ON emp.employee_id = a.employee_id
LEFT JOIN keycard_logs k 
       ON k.employee_id = a.employee_id
      AND k.entry_time <= a.claim_time
      AND k.exit_time >= a.claim_time
WHERE k.room IS NULL OR k.room <> a.claimed_location;

```

#### 📞 4️⃣ Step 4 — Who made suspicious calls at 20:50–21:00?
```
SELECT c.call_id, ca.name AS caller, re.name AS receiver,
       c.call_time, c.duration_sec
FROM calls c
LEFT JOIN employees ca ON ca.employee_id = c.caller_id
LEFT JOIN employees re ON re.employee_id = c.receiver_id
WHERE c.call_time BETWEEN '2025-10-15 20:50:00' AND '2025-10-15 21:00:00'
ORDER BY c.call_time;
```
#### 🧪 5️⃣ Step 5 — Whose movements overlap with found evidence?
```
SELECT 
    ev.evidence_id,
    ev.room AS evidence_room,
    ev.description,
    ev.found_time,
    k.employee_id,
    emp.name AS employee_name,
    k.room AS employee_location,
    k.entry_time,
    k.exit_time
FROM evidence ev
LEFT JOIN keycard_logs k
       ON k.room = ev.room
      AND k.entry_time <= ev.found_time
      AND k.exit_time >= ev.found_time - INTERVAL 10 MINUTE
LEFT JOIN employees emp ON emp.employee_id = k.employee_id
ORDER BY ev.found_time;
```
#### 🧩 6️⃣ Step 6 — Combine all clues and narrow down suspects
```
WITH in_office AS (
    SELECT DISTINCT employee_id
    FROM keycard_logs
    WHERE room LIKE '%CEO%'
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
```

## 🗡️ FINAL QUERY — Identify the Killer

(As required: single column named killer)
```
SELECT e.name AS killer
FROM employees e
WHERE e.employee_id IN (
  SELECT DISTINCT employee_id
  FROM keycard_logs
  WHERE room LIKE '%CEO%'
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
```

## 🧾 Conclusion / Explanation

Using SQL joins, filtering, and CTEs, we narrowed down the suspect pool based on:

✔ presence at the CEO’s Office
✔ suspicious phone calls
✔ a false alibi
✔ proximity to the evidence

All clues converge on one employee, revealed in the Final Query.

🎯 Running the final query reveals the murderer.
