---
name: creating-scd2-versioning
description: "Generates a complete Snowflake SCD Type-2 implementation (Table, Stream, Stored Procedure, Task). Use when: user wants SCD2, slowly changing dimension, type-2 versioning, historical tracking. Triggers: SCD2, SCD type 2, slowly changing dimension, type-2 versioning, Create SCD, historical versioning."
---

# SCD Type-2 Versioning Generator

Generates four Snowflake objects from user-provided parameters:

1. **Output Table** - DDL for the SCD2 dimension table (all input columns + SCD2 versioning columns)
2. **Stream** - CDC on the output dimension table
3. **Stored Procedure** - Insert/Update/Soft-delete logic with hash-based change detection
4. **Task** - Scheduled execution of the procedure

## Workflow

### Step 1: Gather Parameters

**Ask** user for required inputs:

```
To generate your SCD Type-2 implementation, provide:

1. DATABASE_NAME - Fully qualified database name
2. SCHEMA_NAME - Schema where objects reside
3. INPUT_TABLE_NAME - Source/staging table name
4. OUTPUT_TABLE_NAME - Target dimension table name
5. PRIMARY_KEY_COLUMNS - One or more business key columns (comma-separated)
6. INPUT_COLUMNS - All columns from the input table (comma-separated)
7. TIMESTAMP_COLUMN - Source timestamp column (e.g., UPDATED_AT_SOURCE)
8. SCHEDULE_INTERVAL - CRON expression (e.g., 'USING CRON 0 */2 * * * Europe/Madrid')
9. WAREHOUSE_NAME - Warehouse for the task
```

If user provides an existing DDL or table reference, parse it to extract columns, data types, and keys automatically.

**STOP**: Confirm all parameters with user before generating SQL.

### Step 2: Compute Derived Values

From the gathered parameters, derive:

- **NON_KEY_NON_TS_COLUMNS** = INPUT_COLUMNS minus PRIMARY_KEY_COLUMNS minus TIMESTAMP_COLUMN
- **HASH_EXPRESSION** = `md5(concat_ws('|', coalesce(<col1>, ''), coalesce(<col2>, ''), ...))` over NON_KEY_NON_TS_COLUMNS
- **PK_JOIN_CONDITION** = `tgt.<PK1> = src.<PK1> and tgt.<PK2> = src.<PK2> ...`

Present derived values for confirmation.

### Step 3: Generate SQL

Generate the four SQL blocks following the templates below.

#### 3a. Output Table

```sql
create or alter table {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} (
    {-- All input columns with exact names and data types from INPUT_COLUMNS --}
    {COL1}    {DATATYPE1},
    {COL2}    {DATATYPE2},
    ...
    {COLn}    {DATATYPEn},
    SCD_ID          varchar       not null,
    UPDATED_AT      timestamp_ntz not null,
    VALID_FROM      timestamp_ntz not null,
    VALID_TO        timestamp_ntz not null,
    IS_ACTIVE       boolean       not null,
    IS_DELETED      boolean       not null
)
comment = 'SCD Type-2 dimension table for {OUTPUT_TABLE_NAME}. Tracks historical versions of records from {INPUT_TABLE_NAME}.';
```

**Rules for Output Table DDL:**
- Preserve **all** input columns with their **exact column names and data types** as defined in the input table.
- Append the six SCD2 versioning columns (`SCD_ID`, `UPDATED_AT`, `VALID_FROM`, `VALID_TO`, `IS_ACTIVE`, `IS_DELETED`) after the input columns.
- Column order: input columns first (in the same order as INPUT_COLUMNS), then SCD2 columns.

#### 3b. Stream

```sql
create stream if not exists {DATABASE_NAME}.{SCHEMA_NAME}.STREAM_{OUTPUT_TABLE_NAME}
    on table {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME}
    show_initial_rows = FALSE
    comment = 'Tracks all inserts, updates, and deletes on {OUTPUT_TABLE_NAME} for change data capture.';
```

#### 3c. Stored Procedure

```sql
create or replace procedure {DATABASE_NAME}.{SCHEMA_NAME}.SP_{OUTPUT_TABLE_NAME}()
returns varchar
language sql
execute as caller
as
$$
declare
    v_load_ts       timestamp_ntz default current_timestamp();
    v_deleted       integer       default 0;
    v_expired       integer       default 0;
    v_inserted      integer       default 0;
begin
    -- STEP 1: INSERT brand-new records (PK not yet in dimension)
    insert into {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} (
        {ALL_INPUT_COLUMNS},
        SCD_ID, UPDATED_AT, VALID_FROM, VALID_TO, IS_ACTIVE, IS_DELETED
    )
    select
        {ALL_INPUT_COLUMNS_PREFIXED_SRC},
        {HASH_EXPRESSION}              as SCD_ID,
        :v_load_ts                     as UPDATED_AT,
        :v_load_ts                     as VALID_FROM,
        '9999-12-31'::timestamp_ntz    as VALID_TO,
        TRUE                           as IS_ACTIVE,
        FALSE                          as IS_DELETED
    from {DATABASE_NAME}.{SCHEMA_NAME}.{INPUT_TABLE_NAME} as src
    where not exists (
        select 1
        from {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} as tgt
        where {PK_JOIN_CONDITION}
        and tgt.IS_ACTIVE = TRUE
    );
    v_inserted := SQLROWCOUNT;

    -- STEP 2: UPDATE changed records (PK exists, hash differs)
    update {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} as tgt
    set
        {SET_ALL_NON_KEY_COLUMNS_FROM_SRC},
        tgt.SCD_ID      = src.DIFF_HASH,
        tgt.UPDATED_AT  = :v_load_ts,
        tgt.VALID_FROM  = :v_load_ts,
        tgt.VALID_TO    = '9999-12-31'::timestamp_ntz,
        tgt.IS_ACTIVE   = TRUE,
        tgt.IS_DELETED  = FALSE
    from (
        select *, {HASH_EXPRESSION} as DIFF_HASH
        from {DATABASE_NAME}.{SCHEMA_NAME}.{INPUT_TABLE_NAME}
    ) as src
    where {PK_JOIN_CONDITION}
    and tgt.IS_ACTIVE = TRUE
    and tgt.SCD_ID != src.DIFF_HASH;
    v_expired := SQLROWCOUNT;

    -- STEP 3: RE-INSERT historical versions from stream (before-image of updated rows)
    insert into {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} (
        {ALL_INPUT_COLUMNS},
        SCD_ID, UPDATED_AT, VALID_FROM, VALID_TO, IS_ACTIVE, IS_DELETED
    )
    select
        {ALL_INPUT_COLUMNS_PREFIXED_STRM},
        strm.SCD_ID,
        :v_load_ts                          as UPDATED_AT,
        strm.VALID_FROM,
        dateadd(second, -1, NEW_TO_DTS)     as VALID_TO,
        FALSE                                as IS_ACTIVE,
        FALSE                                as IS_DELETED
    from {DATABASE_NAME}.{SCHEMA_NAME}.STREAM_{OUTPUT_TABLE_NAME} as strm
    inner join (
        select max({TIMESTAMP_COLUMN}) as NEW_TO_DTS
        from {DATABASE_NAME}.{SCHEMA_NAME}.{INPUT_TABLE_NAME}
    ) on TRUE
    where strm.METADATA$ACTION = 'DELETE'
    and strm.METADATA$ISUPDATE = TRUE;

    -- STEP 4: SOFT-DELETE records no longer in source
    update {DATABASE_NAME}.{SCHEMA_NAME}.{OUTPUT_TABLE_NAME} as tgt
    set
        tgt.UPDATED_AT  = :v_load_ts,
        tgt.VALID_TO    = dateadd(second, -1, :v_load_ts),
        tgt.IS_ACTIVE   = FALSE,
        tgt.IS_DELETED  = TRUE
    where tgt.IS_ACTIVE = TRUE
    and not exists (
        select 1
        from {DATABASE_NAME}.{SCHEMA_NAME}.{INPUT_TABLE_NAME} as src
        where {PK_JOIN_CONDITION}
    );
    v_deleted := SQLROWCOUNT;

    return 'SCD2 load completed. Timestamp: ' || :v_load_ts::varchar
        || ' | Soft-deleted: ' || :v_deleted::varchar
        || ' | Expired: ' || :v_expired::varchar
        || ' | Inserted: ' || :v_inserted::varchar;

exception
    when OTHER then
        return 'SCD2 load FAILED. SQLCODE: ' || SQLCODE
            || ' | SQLERRM: ' || SQLERRM
            || ' | SQLSTATE: ' || SQLSTATE;
end;
$$;
```

#### 3d. Task

```sql
alter task if exists {DATABASE_NAME}.{SCHEMA_NAME}.TASK_{OUTPUT_TABLE_NAME} suspend;

create or alter task {DATABASE_NAME}.{SCHEMA_NAME}.TASK_{OUTPUT_TABLE_NAME}
    warehouse = {WAREHOUSE_NAME}
    schedule = '{SCHEDULE_INTERVAL}'
    comment = 'Executes SP_{OUTPUT_TABLE_NAME} on schedule for SCD Type-2 processing.'
as
    call {DATABASE_NAME}.{SCHEMA_NAME}.SP_{OUTPUT_TABLE_NAME}();

alter task {DATABASE_NAME}.{SCHEMA_NAME}.TASK_{OUTPUT_TABLE_NAME} resume;
```

**STOP**: Present generated SQL to user for review before writing files.

### Step 4: Write Files

If user approves, write the SQL to the appropriate location. Follow repo conventions:
- One object per file
- Naming: `<order>_<layer>_<ObjectClass>.sql`
- Respect the project's SQL style rules (lowercase keywords, UPPERCASE identifiers)

If the repo structure is `sql/<deliverable>/`, create files like:
```
sql/<deliverable>/
  01_PUB_Table.sql
  02_PUB_Stream.sql
  03_PUB_Procedure.sql
  04_PUB_Task.sql
```

## SQL Style Rules (from `.sqlfluff`)

All generated SQL **must** follow these rules:

- **Keywords:** lowercase (`create`, `select`, `insert`, `update`, `delete`, `from`, `where`, `as`, `set`, `returns`, `begin`, `end`, `declare`, `default`)
- **Identifiers:** UPPERCASE (`COLUMN_NAME`, `TABLE_NAME`, alias references like `NEW_TO_DTS`)
- **Functions:** lowercase (`current_timestamp()`, `dateadd()`, `md5()`, `concat_ws()`, `coalesce()`, `max()`)
- **Literals:** UPPERCASE (`NULL`, `TRUE`, `FALSE`)
- **Types:** lowercase (`timestamp_ntz`, `integer`, `varchar`, `boolean`)
- **Indent:** 4 spaces
- **Max line:** 190 chars
- **Not-equal operator:** `!=` (not `<>`)

## Composite Key Handling

When PRIMARY_KEY_COLUMNS contains multiple columns:
- JOIN conditions become multi-predicate: `tgt.COL1 = src.COL1 and tgt.COL2 = src.COL2`
- Hash excludes ALL PK columns and the timestamp column
- NOT EXISTS subqueries include all PK predicates

## Stopping Points

- After Step 1: Parameters confirmed
- After Step 3: SQL reviewed before writing files

## Output

Complete SCD Type-2 SQL implementation (Table + Stream + Procedure + Task) written to specified location.
