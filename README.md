# JSONB DML Query Analysis Report — RDS Configuration Sections

> **Scope**: Analysis of all `## RDS Configuration` sections across the changelogs in this repository.
> **Files Reviewed**: 50+ changelog files across RTB-18000 to RTB-23999 ranges
> **Focus**: Identifying, cataloguing, and classifying every PostgreSQL JSONB DML pattern used

---

## Table of Contents

1. [Overview](#overview)
2. [Summary Table of JSONB Patterns Found](#summary-table-of-jsonb-patterns-found)
3. [Pattern 1 — Field Access Operators (`->` and `->>`)](#pattern-1--field-access-operators---and----)
4. [Pattern 2 — `jsonb_set()` for Nested Updates](#pattern-2--jsonb_set-for-nested-updates)
5. [Pattern 3 — `||` Concatenation / Merge Operator](#pattern-3----concatenation--merge-operator)
6. [Pattern 4 — `?` Key Existence Check](#pattern-4----key-existence-check)
7. [Pattern 5 — `#>` and `#>>` Path Operators](#pattern-5--and--path-operators)
8. [Pattern 6 — `@>` Containment Operator](#pattern-6----containment-operator)
9. [Pattern 7 — `jsonb_agg()` Aggregation](#pattern-7--jsonb_agg-aggregation)
10. [Pattern 8 — `jsonb_build_object()` Construction](#pattern-8--jsonb_build_object-construction)
11. [Pattern 9 — `jsonb_array_elements()` Lateral Unnesting](#pattern-9--jsonb_array_elements-lateral-unnesting)
12. [Pattern 10 — `jsonb_array_length()`](#pattern-10--jsonb_array_length)
13. [Pattern 11 — `jsonb_build_array()`](#pattern-11--jsonb_build_array)
14. [Pattern 12 — `::jsonb` Cast Literals in INSERT/UPDATE](#pattern-12---jsonb-cast-literals-in-insertupdate)
15. [Pattern 13 — `COALESCE(col, '{}'::jsonb)`](#pattern-13--coalescecolumnjsonb)
16. [Pattern 14 — `INSERT INTO` with Embedded JSONB Schema](#pattern-14--insert-into-with-embedded-jsonb-schema)
17. [Pattern 15 — `RETURNS TABLE(data jsonb)` in PL/pgSQL Functions](#pattern-15--returns-tabledata-jsonb-in-plpgsql-functions)
18. [Pattern 16 — Trigger Functions Using JSONB Navigation](#pattern-16--trigger-functions-using-jsonb-navigation)
19. [Pattern 17 — CTE with Conditional JSONB Updates](#pattern-17--cte-with-conditional-jsonb-updates)
20. [Pattern 18 — `jsonb_set` Inside `document_attributes` and `statuses` Arrays](#pattern-18--jsonb_set-inside-document_attributes-and-statuses-arrays)
21. [Catalogue of Tables Targeted](#catalogue-of-tables-targeted)
22. [Learning Guide — What to Study](#learning-guide--what-to-study)
23. [Assessment Question Sets](#assessment-question-sets)

---

## Overview

The changelogs in this repository are deployment instructions for the **INCENTIHUB** platform. The `## RDS Configuration` sections contain raw PostgreSQL DML scripts that configure the platform's data layer. The system heavily uses **JSONB columns** to store semi-structured data such as:

- `core_attributes` — core entity data (agent codes, names, dates, finance figures)
- `additional_attributes` — lookup data, RTB-specific fields
- `system_attributes` — audit/status metadata (last_modified_date, created_by, etc.)
- `si_input_schema` / `si_output_schema` — JSON Schema structures for service interactions
- `processor_schema` — preprocessor/postprocessor configuration
- `statuses` / `stage_status` — workflow status machine definitions
- `document_attributes` — document handling metadata
- `allowed_stages` — permissions/stage configuration (stored as JSON array in text, cast to jsonb)

All queries operate on tables such as `agent`, `service_interaction`, `service_registry`, `object_repository`, `parenttxn`, `childtxn`, `entitytype`, and their staging variants.

---

## Summary Table of JSONB Patterns Found

| # | Pattern | SQL Feature Used | DML Type | Frequency |
|---|---------|-----------------|----------|-----------|
| 1 | Read single field as text | `->>` | SELECT/WHERE | ⭐⭐⭐⭐⭐ Very High |
| 2 | Read nested JSONB object | `->` | SELECT/WHERE | ⭐⭐⭐⭐⭐ Very High |
| 3 | Set/overwrite nested field | `jsonb_set()` | UPDATE | ⭐⭐⭐⭐⭐ Very High |
| 4 | Merge/append to JSONB object | `\|\|` operator | UPDATE | ⭐⭐⭐⭐ High |
| 5 | Check key existence | `?` operator | WHERE / CASE | ⭐⭐⭐⭐ High |
| 6 | Deep path access | `#>` / `#>>` | SELECT/WHERE | ⭐⭐⭐ Medium |
| 7 | Check containment | `@>` | WHERE | ⭐⭐⭐ Medium |
| 8 | Aggregate rows to JSON array | `jsonb_agg()` | SELECT | ⭐⭐⭐⭐ High |
| 9 | Build JSON object inline | `jsonb_build_object()` | SELECT/UPDATE | ⭐⭐⭐⭐ High |
| 10 | Unnest JSONB array laterally | `jsonb_array_elements()` | FROM/JOIN | ⭐⭐⭐⭐ High |
| 11 | Measure array length | `jsonb_array_length()` | WHERE/IF | ⭐⭐⭐ Medium |
| 12 | Build literal JSONB array | `jsonb_build_array()` | UPDATE | ⭐⭐ Low-Med |
| 13 | Insert JSONB document literal | `'...'::jsonb` | INSERT/UPDATE | ⭐⭐⭐⭐⭐ Very High |
| 14 | Safe null initialize | `COALESCE(col, '{}'::jsonb)` | UPDATE | ⭐⭐⭐ Medium |
| 15 | Insert full schema row | `INSERT INTO ... VALUES(... ::jsonb)` | INSERT | ⭐⭐⭐⭐ High |
| 16 | Function returning jsonb table | `RETURNS TABLE(data jsonb)` | FUNCTION | ⭐⭐⭐ Medium |
| 17 | Trigger reading JSONB | `NEW.col -> 'key' ->> 'field'` | TRIGGER | ⭐⭐ Low-Med |
| 18 | Conditional CTE + update | `WITH ... AS (...) UPDATE` | CTE+UPDATE | ⭐⭐⭐ Medium |

---

## Pattern 1 — Field Access Operators (`->` and `->>`)

### Description
- `->` returns the value as **JSONB** (preserves type)
- `->>` returns the value as **TEXT** (for comparison/display)

### Real Examples from Changelogs

```sql
-- RTB-21690 / RTB-22853 — reading agent fields in WHERE clause
WHERE ag.stage->>'stage_id' = 'active'
AND ag.core_attributes->>'agent_code' IS NOT NULL
AND et.core_attributes->>'auto_termination_on_license_exp' = 'true'
AND ag.licencedetail->'core_attributes'->>'license_expiry_date' IS NOT NULL
AND date(to_timestamp(ag.licencedetail->'core_attributes'->>'license_expiry_date', 'DD/MM/YYYY HH24:MI:SS')) < date(current_timestamp)
```

```sql
-- RTB-22637 — accessing nested lookup values
WHERE (arr0.opt0 -> 'core_attributes') ->> 'mapping_status' = 'Active'
  AND agent.system_attributes->>'status' = 'Active'
```

```sql
-- RTB-22951 — trigger function reading system_attributes
NEW.last_modified_ts := to_timestamp(
  (NEW.system_attributes->'tablename'->>'last_modified_date')::text,
  'DD/MM/YYYY HH24:MI:SS'
);
NEW.last_modified_aggregate_ts := to_timestamp(
  (NEW.system_attributes->'tablename'->'aspect'->>'aspect_modified_date')::text,
  'DD/MM/YYYY HH24:MI:SS'
);
```

### Key Rules
- `->` can be chained: `col->'a'->'b'->'c'`
- Mix `->` then `->>` at the final level: `col->'nested'->>'text_value'`
- Use `->>` in comparisons; use `->` when passing to JSONB functions

---

## Pattern 2 — `jsonb_set()` for Nested Updates

### Description
```sql
jsonb_set(target jsonb, path text[], new_value jsonb, create_missing boolean)
```
Modifies a specific key path inside a JSONB column without replacing the entire document.

### Real Examples from Changelogs

```sql
-- RTB-21690 — adding a new attribute to object_repository.core_attributes
UPDATE object_repository or2
SET core_attributes = jsonb_set(
    core_attributes,
    '{subcode_reason_for_termination}',
    '{"is_pii":false,"datatype":"LOV","label_id":"Sub Code Reason for Termination",
      "date_only":true,"lov_detail":{"lov":["Same As Parent"],"data_owner":"Client"},
      "description":"new attribute","is_unique_id":false,"is_filterable":false,
      "attribute_name":"subcode_reason_for_termination"}'::jsonb,
    true
)
WHERE object_id='entitytype' AND object_mode='Draft';
```

```sql
-- RTB-21690 — updating nested path in service_interaction.si_input_schema
UPDATE service_interaction si
SET si_input_schema = jsonb_set(
    si_input_schema,
    '{properties,entitytype,properties,core_attributes,properties,subcode_reason_for_termination}',
    '{"enum": ["Same As Parent"]}',
    true
)
WHERE si_input_schema::text ILIKE '%"entitytype"%' AND object_id = 'entitytype';
```

```sql
-- RTB-21690 — appending to an array within a JSONB path
UPDATE service_interaction si
SET si_input_schema = jsonb_set(
    si_input_schema,
    '{properties,entitytype,properties,core_attributes,order}',
    COALESCE(si_input_schema#>'{properties,entitytype,properties,core_attributes,order}', '[]'::jsonb)
      || '["subcode_reason_for_termination"]'::jsonb,
    true
)
WHERE si_input_schema::text ILIKE '%"entitytype"%' AND object_id = 'entitytype'
AND NOT (COALESCE(
    si.si_input_schema->'properties'->'entitytype'->'properties'->'core_attributes'->'order',
    '[]'::jsonb
) @> '["subcode_reason_for_termination"]'::jsonb);
```

```sql
-- RTB-22699 — building entire nested object with jsonb_build_object inside jsonb_set
UPDATE service_registry
SET processor_schema = jsonb_set(
    processor_schema,
    '{postprocessor}',
    jsonb_build_object(
        'type', 'sns',
        'payload', jsonb_build_object('keys', jsonb_build_array(primary_object)),
        'configuration', jsonb_build_object(
            'topic', jsonb_build_array('{{audit-log-topic}}', '{{object-cud-status-chain-topic}}')
        )
    ),
    true
)
WHERE service_repository_id ILIKE '%update%internal%' OR service_repository_id ILIKE '%bulkupload';
```

```sql
-- RTB-22716 — moving data from one JSONB field to another
UPDATE agent
SET
    core_attributes = jsonb_set(
        core_attributes,
        '{ifsc_code}',
        bankdetail -> 'core_attributes' -> 'ifsc_code',
        true
    ),
    additional_attributes = jsonb_set(
        additional_attributes,
        '{rtb_lookup_data,ifsc_code}',
        bankdetail -> 'additional_attributes' -> 'rtb_lookup_data' -> 'ifsc_code',
        true
    )
WHERE bankdetail -> 'core_attributes' -> 'ifsc_code' IS NOT NULL;
```

### Key Rules
- Path is always `'{key1,key2,...}'` — a text array literal
- `create_missing = true` creates the key if absent; `false` does nothing if key doesn't exist
- Can be nested: `jsonb_set(jsonb_set(...), ...)` for multiple paths (or use `||`)
- Value argument must be JSONB — use `'...'::jsonb` for literals

---

## Pattern 3 — `||` Concatenation / Merge Operator

### Description
Merges two JSONB objects. Overlapping keys are **overwritten by the right operand**.

### Real Examples from Changelogs

```sql
-- RTB-22480 — conditionally adding an attribute if not present
UPDATE object_repository
SET core_attributes = COALESCE(core_attributes, '{}'::jsonb)
    || '{"termination_letter":{"attribute_name":"termination_letter","datatype":"Boolean",
         "date_only":true,"description":"termination_letter","is_filterable":false,
         "is_pii":false,"is_unique_id":false,"label_id":"Termination Letter"}}'::jsonb
WHERE object_id = 'entitytype'
AND NOT (COALESCE(core_attributes, '{}'::jsonb) ? 'termination_letter');
```

```sql
-- RTB-22480 — appending to a sequence array inside document_attributes
UPDATE object_repository
SET document_attributes = jsonb_set(
    COALESCE(document_attributes, '{}'::jsonb)
      || '{"termination_letter":{...}}'::jsonb,
    '{sequence}',
    COALESCE(document_attributes->'sequence', '[]'::jsonb) || '["termination_letter"]'::jsonb
)
WHERE object_id = 'agentbusinesscode'
AND NOT (COALESCE(document_attributes, '{}'::jsonb) ? 'termination_letter');
```

```sql
-- RTB-22027 — casting text JSON field to jsonb and appending
UPDATE service_registry
SET allowed_stages = (
    (allowed_stages::jsonb || '"terminated"'::jsonb)::text
)
WHERE NOT (allowed_stages::jsonb ? 'terminated')
AND service_repository_id = 'deleteagenttermination';
```

```sql
-- RTB-23679 — appending an item to a JSONB array column
SET input_domain_objects = input_domain_objects || '["agententitlementfcmgst"]'::jsonb
```

### Key Rules
- `{} || {...}` = merge objects (right wins on conflict)
- `[] || [...]` = concat arrays
- `[] || '"item"'::jsonb` = append single item to array
- The `||` result type is `jsonb` — cast back to text if column is stored as `text`

---

## Pattern 4 — `?` Key Existence Check

### Description
Checks whether a JSONB object **contains a given key** (top-level only).

### Real Examples from Changelogs

```sql
-- RTB-22480 — guard before adding a key
WHERE NOT (COALESCE(core_attributes, '{}'::jsonb) ? 'termination_letter');
```

```sql
-- RTB-22384 — conditional in CASE expression
CASE
    WHEN statuses -> 'onboarding' ? 'formsgenerate1a1bannexure' THEN statuses
    ELSE jsonb_set(statuses, '{onboarding, formsgenerate1a1bannexure}', '...'::jsonb, true)
END
```

```sql
-- RTB-22668 — check at nested path
WHEN si_input_schema #> '{properties,agent,properties,agentbankdetail,properties,core_attributes,properties}'
     ? 'bank_branch_name'
THEN 'already_exists'
```

```sql
-- RTB-22027 — check array item existence in text-stored JSON
WHERE NOT (allowed_stages::jsonb ? 'terminated')
```

### Key Rules
- `col ? 'key'` → checks only the immediate top-level keys
- For nested: first navigate with `#>`, then apply `?`
- `?|` = any of keys exist; `?&` = all of keys exist (less common but available)

---

## Pattern 5 — `#>` and `#>>` Path Operators

### Description
Navigate deep JSONB paths using an **array of key names**.
- `#>` returns JSONB
- `#>>` returns text

### Real Examples from Changelogs

```sql
-- RTB-22668 — deep path access in WHERE
WHERE si_input_schema #> '{properties,agent,properties,agentbankdetail,properties,core_attributes,properties}'
      ? 'bank_branch_name'
```

```sql
-- RTB-21690 — COALESCE with a path to avoid null on array append
COALESCE(si_input_schema #> '{properties,entitytype,properties,core_attributes,order}', '[]'::jsonb)
  || '["subcode_reason_for_termination"]'::jsonb
```

```sql
-- RTB-23679 — access to deeply nested last_modified path in function body
to_timestamp(
  jsonb_array_elements(a.withholdtaxexemption) -> 'core_attributes' ->> 'certificate_applicable_date',
  'DD/MM/YYYY HH24:MI:SS'
)
```

### Key Rules
- `col #> '{a,b,c}'` ≡ `col->'a'->'b'->'c'` but more readable for deep paths
- Used especially when the path is dynamic or very long
- Combine with `?` for conditionally present nested keys

---

## Pattern 6 — `@>` Containment Operator

### Description
Tests whether a JSONB value **contains** another JSONB value (subset check for objects or arrays).

### Real Examples from Changelogs

```sql
-- RTB-21690 — check if an array already contains a specific string item
AND NOT (
    COALESCE(
        si.si_input_schema->'properties'->'entitytype'->'properties'->'core_attributes'->'order',
        '[]'::jsonb
    ) @> '["subcode_reason_for_termination"]'::jsonb
)
```

```sql
-- RTB-22384 — check if sequence array already contains "forms" string
WHEN NOT (document_attributes -> 'sequence' @> '"forms"')
THEN jsonb_set(document_attributes, '{sequence}', (document_attributes->'sequence') || '"forms"'::jsonb, true)
```

### Key Rules
- `'["a","b","c"]'::jsonb @> '["b"]'::jsonb` → `true`
- `'{"x":1,"y":2}'::jsonb @> '{"x":1}'::jsonb` → `true`
- The operands must be the same JSONB type (both arrays or both objects)
- Commonly used with `NOT @>` to avoid duplicate inserts into arrays

---

## Pattern 7 — `jsonb_agg()` Aggregation

### Description
Aggregates multiple rows into a single JSONB array.

### Real Examples from Changelogs

```sql
-- RTB-22637 — aggregating agent codes as a JSONB array
agentCodes := (
    SELECT jsonb_agg(core_attributes->>'agent_code')
    FROM agent
    WHERE last_modified_ts > last_requested_date_times
       OR last_modified_aggregate_ts > last_requested_date_times
);
```

```sql
-- RTB-22637 — return result as jsonb from function
RETURN QUERY SELECT jsonb_agg(jsonb_build_object('agentmaster_id', agentmaster_id))
FROM agent
WHERE last_modified_ts > last_requested_date_times
   OR last_modified_aggregate_ts > last_requested_date_times;
```

```sql
-- RTB-23679 — aggregating full objects with COALESCE null-safe default
SELECT jsonb_build_object(
    'parenttxn_variables',
    COALESCE(
        (SELECT jsonb_agg(jsonb_build_object(
            'parent_txn_id', parent_txn_id,
            'agent_code', agent_code,
            'CGST', cgst,
            'Total_GST', total_gst
        )) FROM grouped_parenttxn),
        '[]'::jsonb
    )
) AS result_json;
```

### Key Rules
- Returns `NULL` if no rows exist — use `COALESCE(..., '[]'::jsonb)` for empty array
- Combine with `jsonb_build_object()` to aggregate complex objects
- Can be assigned to a `JSONB` variable in PL/pgSQL

---

## Pattern 8 — `jsonb_build_object()` Construction

### Description
Builds a JSONB object from alternating key-value pairs.

### Real Examples from Changelogs

```sql
-- RTB-22699 — building nested postprocessor config
jsonb_build_object(
    'type', 'sns',
    'payload', jsonb_build_object(
        'keys', jsonb_build_array(primary_object)
    ),
    'configuration', jsonb_build_object(
        'topic', jsonb_build_array(
            '{{audit-log-topic}}',
            '{{object-cud-status-chain-topic}}'
        )
    )
)
```

```sql
-- RTB-23679 — building result in a SELECT
SELECT jsonb_build_object(
    'parenttxn_variables',   COALESCE((SELECT jsonb_agg(...) FROM grouped_parenttxn), '[]'::jsonb),
    'parenttxnhold_variables', COALESCE((SELECT jsonb_agg(...) FROM grouped_hold), '[]'::jsonb),
    'parenttxnrelease_variables', COALESCE((SELECT jsonb_agg(...) FROM grouped_release), '[]'::jsonb)
) AS result_json;
```

```sql
-- RTB-22637 — building return value
RETURN jsonb_build_object('error', 'Payout cycle not found');
```

### Key Rules
- Arguments are `key, value, key, value, ...` (must be even count)
- Keys are always text; values are any SQL type (auto-converted to JSONB)
- Nest calls for deeply structured output
- Combine with `jsonb_agg()` for array values

---

## Pattern 9 — `jsonb_array_elements()` Lateral Unnesting

### Description
Expands a JSONB array into a set of rows — each element becomes one row.

### Real Examples from Changelogs

```sql
-- RTB-21690, RTB-22637 — lateral join to unnest supervisormap
FROM agent ag
LEFT JOIN LATERAL jsonb_array_elements(ag.supervisormap) AS json_data ON true
WHERE (json_data->'core_attributes'->>'mapping_status') = 'Active'
```

```sql
-- RTB-22637 — unnest with ORDINALITY to get index
LEFT JOIN LATERAL jsonb_array_elements(a.supervisormap) WITH ORDINALITY arr0(opt0, ord0)
    ON (arr0.opt0 -> 'core_attributes') ->> 'mapping_status' = 'Active'
```

```sql
-- RTB-23679 — unnest with element filtering
FROM jsonb_array_elements(b.adjustmentdetail) AS element
WHERE element->>'status' = 'Active'
```

```sql
-- RTB-23679 — unnest and sort inside subquery
SELECT jsonb_array_elements(a.withholdtaxexemption)
ORDER BY to_timestamp(
    jsonb_array_elements(a.withholdtaxexemption)->'core_attributes'->>'certificate_applicable_date',
    'DD/MM/YYYY HH24:MI:SS'
) DESC
```

### Key Rules
- Must be in a `FROM` or `JOIN LATERAL` clause
- Each row produced has the element as a `jsonb` column
- Use `WITH ORDINALITY` to get a sequential index alongside
- Use `AS alias(colname, ord)` to name the columns
- `CROSS JOIN jsonb_array_elements(col)` expands every array element × every row

---

## Pattern 10 — `jsonb_array_length()`

### Description
Returns the number of elements in a JSONB array.

### Real Examples from Changelogs

```sql
-- RTB-22637 — guard conditional before expensive insert
IF jsonb_array_length(agentCodes) > 0 THEN
    -- ...expensive insert...
END IF;
```

```sql
-- RTB-23679 — conditional in SELECT CASE
CASE
    WHEN jsonb_array_length(a.withholdtaxexemption) > 0 THEN 1
    ELSE 0
END AS has_exempt_certs
```

### Key Rules
- Returns `0` for empty `[]`, error for non-array values
- Always check that the JSONB value is actually an array before calling

---

## Pattern 11 — `jsonb_build_array()`

### Description
Constructs a JSONB array from a list of values.

### Real Examples from Changelogs

```sql
-- RTB-22699 — build an array of topic names
jsonb_build_array(
    '{{audit-log-topic}}',
    '{{object-cud-status-chain-topic}}'
)
```

```sql
-- RTB-22699 — build array from a column value
jsonb_build_array(primary_object)
```

### Key Rules
- Takes any number of arguments (values are auto-typed)
- Equivalent to `'["a","b"]'::jsonb` for static arrays, but dynamic

---

## Pattern 12 — `::jsonb` Cast Literals in INSERT/UPDATE

### Description
Casting a string literal containing JSON to the `jsonb` type. Used heavily to insert structured schema documents.

### Real Examples from Changelogs

```sql
-- RTB-22286 / RTB-22027 — INSERT with large schema body
INSERT INTO service_interaction
    (si_id, si_version, si_name, si_status, si_source,
     service_repository_id, object_id, si_input_schema, si_output_schema,
     si_type, applicable_role, applicable_stage_status, audit)
VALUES(8000, 1, 'addressupdateinternal', 'active', 'standard',
       'updateagentbulkupload', 'agent',
       '{"type":"object","required":["agent"],"properties":{...}}'::jsonb,
       '[{"spec":{"agent":"&"},"operation":"shift"}]'::jsonb,
       'default', NULL, NULL, NULL);
```

```sql
-- RTB-22480 — literal patch value in UPDATE
SET core_attributes = COALESCE(core_attributes, '{}'::jsonb)
    || '{"termination_letter":{...}}'::jsonb
```

```sql
-- Empty JSONB values
'{}'::jsonb     -- empty object
'[]'::jsonb     -- empty array
'null'::jsonb   -- JSON null
```

### Key Rules
- Always validate JSON is well-formed before casting — runtime error on bad JSON
- Very large schema literals are common in `service_interaction.si_input_schema`
- `'[]'::jsonb` is the safe empty array default
- `'{}'::jsonb` is the safe empty object default

---

## Pattern 13 — `COALESCE(column, '{}'::jsonb)`

### Description
Guards against NULL JSONB columns before performing merge or `?` checks.

### Real Examples from Changelogs

```sql
-- RTB-22480 — safe merge even when column is NULL
SET core_attributes = COALESCE(core_attributes, '{}'::jsonb)
    || '{"new_key":{...}}'::jsonb
WHERE NOT (COALESCE(core_attributes, '{}'::jsonb) ? 'new_key');
```

```sql
-- RTB-22668 — safe path init
jsonb_set(
    COALESCE(document_attributes, '{}'::jsonb) || '{"termination_letter":{...}}'::jsonb,
    '{sequence}',
    COALESCE(document_attributes->'sequence', '[]'::jsonb) || '["termination_letter"]'::jsonb
)
```

### Key Rules
- `NULL || '{}'::jsonb` returns `NULL` — always COALESCE first
- `NULL ? 'key'` returns `NULL` — guard before existence checks too
- Standard idiom: `COALESCE(col, '{}'::jsonb) || '{"k":"v"}'::jsonb`

---

## Pattern 14 — `INSERT INTO` with Embedded JSONB Schema

### Description
Full-row INSERTs into configuration tables where columns hold complete JSON Schema documents.

### Real Examples from Changelogs

```sql
-- RTB-22027 — inserting a full service_interaction row
INSERT INTO service_interaction
    (si_id, si_version, si_name, si_status, si_source,
     service_repository_id, object_id, si_input_schema, si_output_schema,
     si_type, applicable_role, applicable_stage_status, audit)
VALUES(
    6602, 1, 'deletetermination', 'active', 'standard',
    'deleteagenttermination', 'agenttermination',
    '{"type": "object", "required": ["agenttermination"], "properties": {
        "agenttermination": {"type": "object", "required": [], "properties": {
            "root_id": {"type": "number"},
            "core_attributes": {"type": "object", "order": [...], "required": [], "properties": {
                "remarks": {"type": "string"},
                "termination_date": {"type": "string", "pattern": "^...$"},
                ...
            }}
        }}
    }}'::jsonb,
    '[{"spec": {"agent": "&", "agenttermination": "&"}, "operation": "shift"}]'::jsonb,
    'default', NULL, NULL, NULL
);
```

```sql
-- RTB-23679 — inserting object_repository entry with multiple JSONB columns
INSERT INTO object_repository VALUES(
    'agententitlementfcmgst', 'FCMGST', 'Cycle Invoice', 'DAO', 'FCMGST',
    false, NULL, NULL,
    '{...core_attributes schema...}'::jsonb,
    'agententitlement', NULL,
    '{}'::jsonb,
    '{...statuses/workflow JSON...}'::jsonb,
    'Draft', 'Multiple', 15776, NULL, NULL, 'agententitlement',
    NULL, NULL,
    '{...document_attributes...}'::jsonb,
    NULL, NULL, NULL, false, NULL, false, false,
    '[]'::jsonb
);
```

### Key Rules
- Multiple JSONB columns per row are common — each gets `'...'::jsonb`
- The JSON Schema structure (`type`, `properties`, `required`, `pattern`) is standardized
- Large INSERTs are hard to read — validate JSON externally before pasting

---

## Pattern 15 — `RETURNS TABLE(data jsonb)` in PL/pgSQL Functions

### Description
PostgreSQL functions that return a result as a table of JSONB rows, enabling the caller to process structured results.

### Real Examples from Changelogs

```sql
-- RTB-22637
CREATE OR REPLACE FUNCTION refresh_agent_staging()
RETURNS TABLE(data jsonb)
LANGUAGE plpgsql
AS $function$
DECLARE
    agentCodes jsonb;
    ...
BEGIN
    agentCodes := (
        SELECT jsonb_agg(core_attributes->>'agent_code')
        FROM agent
        WHERE last_modified_ts > last_requested_date_times
    );
    IF jsonb_array_length(agentCodes) > 0 THEN
        -- ... do INSERT ...
    END IF;
    RETURN QUERY SELECT jsonb_agg(jsonb_build_object('agentmaster_id', agentmaster_id))
    FROM agent
    WHERE last_modified_ts > last_requested_date_times;
END;
$function$;
```

```sql
-- RTB-23679 — function with multiple JSONB variables
CREATE OR REPLACE FUNCTION calculate_payout(p_payoutcycle_id TEXT)
RETURNS jsonb
LANGUAGE plpgsql
AS $function$
DECLARE
    v_agent_missing JSONB;
    v_entitlement_missing JSONB;
    v_tax_config JSONB;
    v_core_attributes JSONB;
BEGIN
    IF NOT FOUND THEN
        RETURN jsonb_build_object('error', 'Payout cycle not found');
    END IF;
    SELECT COALESCE(jsonb_agg(a.core_attributes->>'agent_code'), '[]'::jsonb)
    INTO v_entitlement_missing FROM ...;
    IF jsonb_array_length(v_entitlement_missing) > 0 THEN
        RETURN jsonb_build_object('reference_object_missing_entitlement', v_entitlement_missing);
    END IF;
    ...
END;
$function$;
```

### Key Rules
- Use `RETURN QUERY SELECT ...` to stream rows from functions
- Use `RETURNS jsonb` for single-value functions; `RETURNS TABLE(data jsonb)` for multi-row
- `JSONB` variables are declared in `DECLARE` block and assigned with `:=`

---

## Pattern 16 — Trigger Functions Using JSONB Navigation

### Description
`BEFORE INSERT OR UPDATE` triggers that read JSONB columns from `NEW.*` to populate computed columns.

### Real Examples from Changelogs

```sql
-- RTB-22951 — trigger function to sync last_modified timestamps
CREATE OR REPLACE FUNCTION tablename_update_last_modified_ts()
RETURNS trigger AS $$
BEGIN
    NEW.last_modified_ts := to_timestamp(
        (NEW.system_attributes->'tablename'->>'last_modified_date')::text,
        'DD/MM/YYYY HH24:MI:SS'
    );
    NEW.last_modified_aggregate_ts := to_timestamp(
        (NEW.system_attributes->'tablename'->'aspect'->>'aspect_modified_date')::text,
        'DD/MM/YYYY HH24:MI:SS'
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tablename_trg_update_last_modified_ts
BEFORE INSERT OR UPDATE ON tablename
FOR EACH ROW
EXECUTE FUNCTION tablename_update_last_modified_ts();
```

### Key Rules
- `NEW.col` accesses the incoming row's JSONB value
- Result is written back to `NEW.computed_col` for derived columns
- Trigger must `RETURN NEW` after modification

---

## Pattern 17 — CTE with Conditional JSONB Updates

### Description
Uses a Common Table Expression (CTE) to evaluate conditions on JSONB fields, then applies the update.

### Real Examples from Changelogs

```sql
-- RTB-22668 — safe idempotent update using CTE
WITH check_and_update AS (
    SELECT
        si_id, si_version,
        CASE
            WHEN si_input_schema #> '{properties,agent,properties,agentbankdetail,properties,core_attributes,properties}'
                 ? 'bank_branch_name'
            THEN 'already_exists'
            ELSE 'newly_added'
        END AS status,
        CASE
            WHEN si_input_schema #> '{properties,agent,properties,agentbankdetail,properties,core_attributes,properties}'
                 ? 'bank_branch_name'
            THEN si_input_schema
            ELSE jsonb_set(
                si_input_schema,
                '{properties,agent,properties,agentbankdetail,properties,core_attributes,properties,bank_branch_name}',
                '{"type": "string"}'::jsonb,
                true
            )
        END AS updated_schema
    FROM service_interaction
    WHERE si_id = '5000' AND si_version = '12'
)
UPDATE service_interaction si
SET si_input_schema = c.updated_schema
FROM check_and_update c
WHERE si.si_id = c.si_id AND si.si_version = c.si_version
RETURNING si.si_id, c.status AS action_taken;
```

### Key Rules
- CTEs make the logic readable and auditable
- `RETURNING` clause confirms what changed
- The status field (`already_exists` / `newly_added`) is used for deployment audit

---

## Pattern 18 — `jsonb_set` Inside `document_attributes` and `statuses` Arrays

### Description
Combines `||` merge and `jsonb_set` to update both object keys and embedded arrays atomically.

### Real Examples from Changelogs

```sql
-- RTB-22384 — complex double update: statuses + document_attributes in one UPDATE
UPDATE object_repository
SET
    statuses = CASE
        WHEN statuses -> 'onboarding' ? 'formsgenerate1a1bannexure'
        THEN statuses
        ELSE jsonb_set(
            statuses,
            '{onboarding, formsgenerate1a1bannexure}',
            '{"final":false,"initial":false,"publish":true,"label_id":"Forms Generate 1A & 1B & Annexure",
              "status_id":"formsgenerate1a1bannexure","status_name":"formsgenerate1a1bannexure",
              "final_status_objects":[],"mandatory_attributes":[]}'::jsonb,
            true
        )
    END,
    document_attributes = jsonb_set(
        CASE
            WHEN NOT (document_attributes ? 'sequence') THEN
                jsonb_set(document_attributes, '{sequence}', '["forms"]'::jsonb, true)
            WHEN NOT (document_attributes -> 'sequence' @> '"forms"') THEN
                jsonb_set(document_attributes, '{sequence}',
                    (document_attributes -> 'sequence') || '"forms"'::jsonb, true)
            ELSE document_attributes
        END,
        '{forms}',
        '{"datatype":"Document","label_id":"Form 1-A & 1-B & Annexure -1",
          "attribute_name":"forms","document_master_id":"Forms","document_category_id":"statement"}'::jsonb,
        true
    )
WHERE object_id = 'agentindividualdetails' AND statuses ? 'onboarding';
```

---

## Catalogue of Tables Targeted

| Table | JSONB Columns Updated |
|-------|-----------------------|
| `agent` | `core_attributes`, `additional_attributes`, `system_attributes`, `supervisormap`, `bankdetail`, `licencedetail`, `termination`, `stage` |
| `object_repository` | `core_attributes`, `statuses`, `document_attributes` |
| `service_interaction` | `si_input_schema`, `si_output_schema` |
| `service_registry` | `processor_schema`, `allowed_stages` (text→jsonb cast) |
| `parenttxn` / `childtxn` | `peoplecreditmap`, `creditmap`, `txnlifecycleeventhistory` |
| `entitytype` | `core_attributes` |
| `agententitlement` | `core_attributes`, `stage_status`, `system_attributes`, `reference_object` |

---

## Learning Guide — What to Study

### Level 1 — Fundamentals (Start Here)

| Topic | What to Learn |
|-------|---------------|
| JSONB vs JSON | Differences in storage, indexing, and operators |
| `->` vs `->>` | When each is appropriate; chaining |
| `::jsonb` cast | How PostgreSQL parses JSON string literals |
| `'{}'::jsonb` defaults | Safe initialization pattern |
| Basic `SELECT` with JSONB | `WHERE col->>'key' = 'value'` queries |

**Practice Table:**
```sql
CREATE TABLE test_jsonb (id SERIAL, data JSONB);
INSERT INTO test_jsonb(data) VALUES
  ('{"name":"Alice","age":30,"address":{"city":"Mumbai","pincode":"400001"}}'),
  ('{"name":"Bob","age":25,"address":{"city":"Delhi","pincode":"110001"}}');
```

**Practice Queries:**
```sql
SELECT data->>'name', data->'address'->>'city' FROM test_jsonb;
SELECT * FROM test_jsonb WHERE data->>'name' = 'Alice';
SELECT * FROM test_jsonb WHERE data->'address'->>'city' = 'Mumbai';
```

---

### Level 2 — UPDATE Patterns (Core Skills)

| Topic | What to Learn |
|-------|---------------|
| `jsonb_set()` | Path syntax, `create_missing` flag |
| `\|\|` operator | Object merge vs array concatenation |
| `?` operator | Key existence, `?|`, `?&` |
| `NOT (col ? 'key')` | Idempotent add-if-missing |
| `COALESCE(col, '{}'::jsonb)` | NULL safety |

**Practice Queries:**
```sql
-- Add a field
UPDATE test_jsonb SET data = jsonb_set(data, '{email}', '"alice@example.com"', true)
WHERE data->>'name' = 'Alice';

-- Merge
UPDATE test_jsonb SET data = data || '{"phone":"9999999999"}'::jsonb
WHERE NOT (data ? 'phone');

-- Nested update
UPDATE test_jsonb SET data = jsonb_set(data, '{address,state}', '"Maharashtra"', true);
```

---

### Level 3 — Aggregation and Construction

| Topic | What to Learn |
|-------|---------------|
| `jsonb_agg()` | Aggregate rows to array |
| `jsonb_build_object()` | Construct objects inline |
| `jsonb_array_elements()` | Unnest arrays into rows |
| `jsonb_array_length()` | Count array elements |
| `LATERAL` joins with JSONB | Join each row's array |

**Practice Queries:**
```sql
-- Aggregate all names as JSONB array
SELECT jsonb_agg(data->>'name') FROM test_jsonb;

-- Build object per row
SELECT jsonb_build_object('id', id, 'city', data->'address'->>'city') FROM test_jsonb;

-- Unnest supervisormap-like array
SELECT id, elem->>'key' FROM test_jsonb, jsonb_array_elements(data->'tags') elem;
```

---

### Level 4 — Advanced Patterns

| Topic | What to Learn |
|-------|---------------|
| `#>` / `#>>` | Deep path arrays |
| `@>` containment | Subset queries |
| CTE + conditional UPDATE | Idempotent schema migrations |
| PL/pgSQL with JSONB variables | Functions using `JSONB` declare/assign |
| Trigger functions | `NEW.col` JSONB manipulation |
| GIN Indexes | Performance on JSONB columns |

**Practice:**
```sql
-- Deep path
SELECT data #>> '{address,city}' FROM test_jsonb;

-- Containment
SELECT * FROM test_jsonb WHERE data @> '{"address": {"city": "Mumbai"}}';

-- CTE conditional update
WITH check_cte AS (
    SELECT id, CASE WHEN data ? 'phone' THEN data ELSE data || '{"phone":"0000"}'::jsonb END AS new_data
    FROM test_jsonb
)
UPDATE test_jsonb t SET data = c.new_data FROM check_cte c WHERE t.id = c.id;
```

---

### Recommended Reading Order

1. [PostgreSQL JSONB Documentation](https://www.postgresql.org/docs/current/functions-json.html)
2. `jsonb_set` reference — focus on the `path` syntax
3. GIN index creation: `CREATE INDEX ON table USING gin(jsonb_col);`
4. `LATERAL` joins — critical for working with JSONB arrays
5. PL/pgSQL variables of JSONB type

---

## Assessment Question Sets

### Set A — Beginner (Field Access)

**Q1.** Given a table `agent` with a `core_attributes jsonb` column, write a query to return all agents whose `agent_code` is `'A001'`.

<details><summary>Answer</summary>

```sql
SELECT * FROM agent WHERE core_attributes->>'agent_code' = 'A001';
```
</details>

---

**Q2.** Write a query to get the `city` nested inside `address` inside `core_attributes` as plain text.

<details><summary>Answer</summary>

```sql
SELECT core_attributes->'address'->>'city' AS city FROM agent;
```
</details>

---

**Q3.** What is the difference between `->` and `->>` in PostgreSQL JSONB? When would you use each?

<details><summary>Answer</summary>

- `->` returns JSONB (preserves type; can be used as input to JSONB functions)
- `->>` returns TEXT (for string comparisons, WHERE clauses, display)

Use `->` when passing to `jsonb_set()`, `@>`, or further navigation.
Use `->>` in `WHERE col->>'field' = 'value'` comparisons.
</details>

---

**Q4.** Write a query that selects all agents who are in `stage_id = 'active'` (where `stage` is a top-level JSONB column).

<details><summary>Answer</summary>

```sql
SELECT * FROM agent WHERE stage->>'stage_id' = 'active';
-- or using -> then ->>:
SELECT * FROM agent WHERE (stage->>'stage_id') = 'active';
```
</details>

---

### Set B — Intermediate (UPDATE Patterns)

**Q5.** Using `jsonb_set`, write an UPDATE to add a field `"email_verified": true` inside `core_attributes` of the `agent` table for all agents where `core_attributes->>'agent_code' = 'A001'`. Create the key if missing.

<details><summary>Answer</summary>

```sql
UPDATE agent
SET core_attributes = jsonb_set(
    core_attributes,
    '{email_verified}',
    'true'::jsonb,
    true
)
WHERE core_attributes->>'agent_code' = 'A001';
```
</details>

---

**Q6.** Write an UPDATE to merge `{"mobile_verified": false}` into `core_attributes` for all agents, but only if `mobile_verified` key does NOT already exist.

<details><summary>Answer</summary>

```sql
UPDATE agent
SET core_attributes = COALESCE(core_attributes, '{}'::jsonb) || '{"mobile_verified": false}'::jsonb
WHERE NOT (COALESCE(core_attributes, '{}'::jsonb) ? 'mobile_verified');
```
</details>

---

**Q7.** Using `jsonb_set`, update the nested path `processor_schema -> preprocessor -> configuration -> lambda` to `"my-lambda"` in the `service_registry` table for a specific `service_repository_id`.

<details><summary>Answer</summary>

```sql
UPDATE service_registry
SET processor_schema = jsonb_set(
    processor_schema,
    '{preprocessor,configuration,lambda}',
    '"my-lambda"',
    true
)
WHERE service_repository_id = 'mysvc';
```
</details>

---

**Q8.** You have a `document_attributes` JSONB column. Write an UPDATE that:
1. Adds key `"passport": {...details...}` to `document_attributes`
2. Also appends `"passport"` to the `sequence` array inside `document_attributes`
Do it only if `"passport"` is not already present.

<details><summary>Answer</summary>

```sql
UPDATE object_repository
SET document_attributes = jsonb_set(
    COALESCE(document_attributes, '{}'::jsonb)
      || '{"passport": {"datatype": "Document", "label_id": "Passport", "attribute_name": "passport"}}'::jsonb,
    '{sequence}',
    COALESCE(document_attributes->'sequence', '[]'::jsonb) || '["passport"]'::jsonb
)
WHERE object_id = 'agent'
AND NOT (COALESCE(document_attributes, '{}'::jsonb) ? 'passport');
```
</details>

---

**Q9.** What does `allowed_stages::jsonb ? 'active'` do when `allowed_stages` is stored as a `TEXT` column?

<details><summary>Answer</summary>

It first casts the text column to `jsonb`, then checks whether the JSON value (object or array) contains the key/element `'active'`. For a JSON array like `["active","inactive"]`, `? 'active'` returns `true`. You need the `::jsonb` cast since `?` requires a JSONB operand.
</details>

---

### Set C — Advanced (Aggregation, Arrays, and CTEs)

**Q10.** Write a query that returns a single JSONB object with the structure:
```json
{"total_agents": 100, "active_agents": 75}
```
using `jsonb_build_object` and subqueries on the `agent` table.

<details><summary>Answer</summary>

```sql
SELECT jsonb_build_object(
    'total_agents', COUNT(*),
    'active_agents', COUNT(*) FILTER (WHERE stage->>'stage_id' = 'active')
) FROM agent;
```
</details>

---

**Q11.** The `supervisormap` column in `agent` is a JSONB array. Each element has `core_attributes.mapping_status`. Write a query that returns each agent code alongside all their **active** supervisor codes (as a JSONB array).

<details><summary>Answer</summary>

```sql
SELECT
    core_attributes->>'agent_code' AS agent_code,
    jsonb_agg(sup->'core_attributes'->>'code') AS active_supervisors
FROM agent ag,
     jsonb_array_elements(ag.supervisormap) AS sup
WHERE sup->'core_attributes'->>'mapping_status' = 'Active'
GROUP BY core_attributes->>'agent_code';
```
</details>

---

**Q12.** Write a CTE-based UPDATE that:
1. Checks if `si_input_schema -> 'properties' -> 'agent' ? 'new_field'` exists in `service_interaction`
2. If NOT — adds `{"new_field": {"type": "string"}}` at that path
3. Returns the `si_id` and whether it was `already_exists` or `newly_added`

<details><summary>Answer</summary>

```sql
WITH check_and_update AS (
    SELECT
        si_id, si_version,
        CASE
            WHEN si_input_schema->'properties'->'agent' ? 'new_field' THEN 'already_exists'
            ELSE 'newly_added'
        END AS status,
        CASE
            WHEN si_input_schema->'properties'->'agent' ? 'new_field' THEN si_input_schema
            ELSE jsonb_set(
                si_input_schema,
                '{properties,agent,new_field}',
                '{"type":"string"}'::jsonb,
                true
            )
        END AS updated_schema
    FROM service_interaction
    WHERE object_id = 'agent'
)
UPDATE service_interaction si
SET si_input_schema = c.updated_schema
FROM check_and_update c
WHERE si.si_id = c.si_id AND si.si_version = c.si_version
RETURNING si.si_id, c.status AS action_taken;
```
</details>

---

**Q13.** Write a PL/pgSQL function `get_agent_summary(p_stage TEXT)` that:
- Takes a stage name as input
- Returns `TABLE(data jsonb)` with each row being `{"agent_code": "...", "stage": "..."}`
- Uses `jsonb_agg` and `jsonb_build_object`

<details><summary>Answer</summary>

```sql
CREATE OR REPLACE FUNCTION get_agent_summary(p_stage TEXT)
RETURNS TABLE(data jsonb)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT jsonb_build_object(
        'agent_code', core_attributes->>'agent_code',
        'stage', stage->>'stage_id'
    )
    FROM agent
    WHERE stage->>'stage_id' = p_stage;
END;
$function$;

-- Usage:
SELECT * FROM get_agent_summary('active');
```
</details>

---

**Q14.** Write a query to check whether the `sequence` array inside `document_attributes` **contains** the string `"forms"`, and update it to append `"forms"` only if it doesn't.

<details><summary>Answer</summary>

```sql
UPDATE object_repository
SET document_attributes = jsonb_set(
    document_attributes,
    '{sequence}',
    (document_attributes->'sequence') || '"forms"'::jsonb,
    false
)
WHERE object_id = 'agentindividualdetails'
AND NOT (document_attributes->'sequence' @> '"forms"');
```
</details>

---

**Q15.** What does `jsonb_array_elements(col) WITH ORDINALITY AS arr(elem, ord)` provide, and when would you use it?

<details><summary>Answer</summary>

It unnests a JSONB array into rows, providing:
- `elem` — the JSONB element
- `ord` — a 1-based integer position (ordinality)

Use it when you need the **index** of each element (e.g., to sort, to reference elements by position, or to process first/last elements). Without `WITH ORDINALITY` you only get the element value, not its position.
</details>

---

### Set D — Schema Migration Scenarios (Deployment-Realistic)

**Q16.** You need to add a new attribute `"kyc_verified"` to `object_repository.core_attributes` for `object_id = 'agent'`. The attribute should have structure: `{"datatype":"Boolean","label_id":"KYC Verified","is_pii":false,"attribute_name":"kyc_verified"}`. Write an idempotent query.

<details><summary>Answer</summary>

```sql
UPDATE object_repository
SET core_attributes = jsonb_set(
    core_attributes,
    '{kyc_verified}',
    '{"datatype":"Boolean","label_id":"KYC Verified","is_pii":false,"attribute_name":"kyc_verified"}'::jsonb,
    true
)
WHERE object_id = 'agent'
AND NOT (COALESCE(core_attributes, '{}'::jsonb) ? 'kyc_verified');
```
</details>

---

**Q17.** You have a `service_registry.processor_schema` that looks like:
```json
{"preprocessor": {"type": "lambda", "configuration": {"lambda": "old-lambda"}}}
```
Write an UPDATE to change **only** the lambda name to `"new-lambda"` without touching the rest.

<details><summary>Answer</summary>

```sql
UPDATE service_registry
SET processor_schema = jsonb_set(
    processor_schema,
    '{preprocessor,configuration,lambda}',
    '"new-lambda"'::jsonb,
    false
)
WHERE service_repository_id = 'mysvc';
```
</details>

---

**Q18.** Given a `statuses` JSONB column containing a workflow config, write an idempotent query to add a new status `"pendingkyc"` under the `"onboarding"` key, only if it doesn't exist already.

<details><summary>Answer</summary>

```sql
UPDATE object_repository
SET statuses = CASE
    WHEN statuses->'onboarding' ? 'pendingkyc' THEN statuses
    ELSE jsonb_set(
        statuses,
        '{onboarding,pendingkyc}',
        '{"final":false,"initial":false,"publish":true,
          "label_id":"Pending KYC","status_id":"pendingkyc",
          "status_name":"Pending KYC","mandatory_attributes":[]}'::jsonb,
        true
    )
END
WHERE object_id = 'agent'
AND statuses ? 'onboarding';
```
</details>

---

## Quick Reference Card

```
JSONB OPERATORS:
  ->     Get field as JSONB          col->'key'
  ->>    Get field as TEXT           col->>'key'
  #>     Get path as JSONB           col #> '{a,b,c}'
  #>>    Get path as TEXT            col #>> '{a,b,c}'
  @>     Contains (superset)         col @> '{"k":"v"}'
  <@     Contained by               '{"k":"v"}' <@ col  
  ?      Key exists                  col ? 'key'
  ?|     Any key exists              col ?| ARRAY['k1','k2']
  ?&     All keys exist              col ?& ARRAY['k1','k2']
  ||     Merge/Concat               col1 || col2

JSONB FUNCTIONS:
  jsonb_set(target, path, val, create)  Update nested field
  jsonb_build_object(k,v,k,v,...)       Build object inline
  jsonb_build_array(v1,v2,...)          Build array inline
  jsonb_agg(expr)                       Aggregate to array
  jsonb_array_elements(arr)             Unnest array to rows
  jsonb_array_length(arr)               Count array elements
  jsonb_typeof(val)                     'object'/'array'/etc

COMMON PATTERNS:
  Safe add:   COALESCE(col,'{}'::jsonb) || '{"k":"v"}'::jsonb
  Guard:      WHERE NOT (COALESCE(col,'{}'::jsonb) ? 'key')
  Append arr: col || '["item"]'::jsonb
  Path check: col #> '{a,b}' ? 'key'
  Array has:  col->'arr' @> '"item"'
```
