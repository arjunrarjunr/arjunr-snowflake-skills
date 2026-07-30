---
name: ingesting-preprocess
description: "Validates and uploads {report_name} CSV to Snowflake stage. Use when: user provides a CSV for {report_name}, mentions {report_name} upload, or wants to stage {report_name} data. Triggers: {report_name} upload, {report_name} CSV, upload {report_name}, stage {report_name}."
---

# {report_name} Ingestion

Guides the user through validating and uploading their CSV file to Snowflake stage
`{snowflake_database}.{snowflake_schema}.{snowflake_stage}`.

## Prerequisites

- User has a CSV file to upload
- Active Snowflake connection with access to `{snowflake_database}.{snowflake_schema}`

## Expected Schema

| # | Column Name |
|---|-------------|
| 1 | column-1 |
| 2 | column-2 |
| 3 | column-3 |
| 4 | column-4 |
| 5 | column-5 |
| 6 | column-6 |
| 7 | column-7 |
| 8 | column-8 |
| 9 | column-9 |
| 10 | column-10 |

## Workflow

### Step 1: File Receipt

**Goal:** Accept and inspect the CSV file.

**Actions:**

1. Accept the CSV file path from the user
2. Read the file and display basic info:
   - Row count (excluding header)
   - Column count
   - First 5 rows as preview

**If file cannot be read:** Inform user of the error (encoding issue, path not found, not a CSV). Stop.

### Step 2: File Format Validation

**Goal:** Validate the CSV structure matches the Snowflake file format.

**Actions:**

1. Retrieve the expected file format definition:
   ```sql
   describe file format {snowflake_database}.{snowflake_schema}.{snowflake_file_format};
   ```
2. Check the CSV against format properties (delimiter, quoting, encoding, header rows, etc.)
3. Report validation result:

**If INVALID:**
- Show which properties don't match (e.g., wrong delimiter, unexpected quoting)
- ⚠️ **MANDATORY STOPPING POINT**: Ask user how they'd like to handle the discrepancy
- Options: fix the file, proceed anyway, abort

**If VALID:** Confirm and proceed.

### Step 3: Column Name Validation

**Goal:** Verify column names match expected schema.

**Actions:**

1. Extract header row from the CSV
2. Compare each column name to the expected schema (case-insensitive, fuzzy match allowed)
3. Categorize columns as: exact match, close match, or no match

**If names differ significantly:**
- Present a comparison table:
  ```
  | # | Expected Column | Actual Column | Status |
  |---|------------------|----------------|--------|
  | 1 | column-1         | Column-1       | Close  |
  | 2 | column-2         | column_2       | Differ |
  ```
- ⚠️ **MANDATORY STOPPING POINT**: For each mismatched column, ask:
  "Do columns [actual_name] and [expected_name] contain the same data?"
- Only rename upon explicit user consent

**If names are similar/exact:** Confirm and proceed.

### Step 4: Column Order Validation

**Goal:** Verify columns are in the expected order.

**Actions:**

1. Compare actual column order against expected order
2. If out of order, display side-by-side:
   ```
   | Position | Expected | Actual |
   |----------|----------|--------|
   | 1        | column-1 | column-2 |
   | 2        | column-2 | column-1 |
   ```

**If out of order:** ⚠️ **MANDATORY STOPPING POINT**: Ask: "The columns are not in the expected order. May I reorder them?"
Only reorder upon explicit user consent

**If in order:** Confirm and proceed.

### Step 5: Upload Confirmation

**Goal:** Final review before upload.

**Actions:**

1. Summarize all validations and any transformations to be applied:
   - File format: PASS/FAIL (with resolution)
   - Column names: matched/renamed (list changes)
   - Column order: correct/reordered
2. Ask: "Would you like to keep the original file name or provide a custom name?"
3. ⚠️ **MANDATORY STOPPING POINT**: Present final confirmation:
   "Ready to upload [filename] to stage `{snowflake_database}.{snowflake_schema}.{snowflake_stage}`. Proceed?"

### Step 6: Upload Execution

**Goal:** Upload the file to the Snowflake stage.

**Actions:**

1. If transformations were applied (rename/reorder), write the modified CSV to a temporary path
2. Determine `PUT` command parameters from the file format retrieved in Step 2:
   - If `COMPRESSION` = `none` → use `auto_compress = FALSE`
   - If `COMPRESSION` = `AUTO` or any named codec (e.g., `GZIP`, `ZSTD`) → use `auto_compress = TRUE`
3. Use forward slashes in the file path to avoid backslash escaping issues on Windows
4. Execute the upload:
   ```sql
   put 'file://<local_path_with_forward_slashes>' @{snowflake_database}.{snowflake_schema}.{snowflake_stage}
       auto_compress = <TRUE|FALSE based on COMPRESSION property>
       overwrite = TRUE;
   ```
5. Verify the upload:
   ```sql
   list @{snowflake_database}.{snowflake_schema}.{snowflake_stage} pattern = '.*<filename>.*';
   ```
6. Report result: file name, size, and stage path

## Error Handling

| Situation | Action |
|-----------|--------|
| File not found / unreadable | Inform user, ask for correct path |
| File is not CSV (binary, Excel, etc.) | Inform user, suggest converting to CSV first |
| File format mismatch | Present differences, ask user how to proceed |
| Column count mismatch (more/fewer than 10) | Flag immediately, ask user before continuing |
| Stage does not exist or access denied | Show error, suggest checking permissions/connection |
| PUT command fails | Show full error, suggest retry or checking file path |
| Network timeout | Suggest retry; if persistent, ask user to check connection |

## Interaction Rules

- Always get EXPLICIT user consent before making changes
- Present differences clearly using tables/comparisons
- Never silently transform data
- If multiple issues found, address them one at a time
- Provide a final summary before upload

## Stopping Points

- ⚠️ After Step 2 if file format is invalid (user decides resolution)
- ⚠️ After Step 3 if column names differ (user approves renaming)
- ⚠️ After Step 4 if column order differs (user approves reordering)
- ⚠️ After Step 5 before upload (final confirmation)

## Success Criteria

- ✅ File passes all validations (or user explicitly approved transformations)
- ✅ File uploaded to stage successfully
- ✅ Upload verified via `LIST` command
- ✅ User informed of final file location and size
