# Stored Procedure Patterns

## Validated Create Procedure

```sql
DEFINE PROCEDURE {schema}.create_{entity}(
    p_field1 VARCHAR,
    p_field2 VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    v_id VARCHAR DEFAULT UUID_STRING();
    v_job_id VARCHAR DEFAULT UUID_STRING();
BEGIN
    -- Validation
    IF (:p_field1 IS NULL OR LENGTH(TRIM(:p_field1)) = 0) THEN
        RETURN OBJECT_CONSTRUCT(
            'success', FALSE,
            'errors', ARRAY_CONSTRUCT(
                OBJECT_CONSTRUCT('type', 'validation', 'field', 'field1',
                                 'code', 'required', 'message', 'Field1 is required')
            )
        );
    END IF;

    -- Insert
    INSERT INTO {schema}.{entity} (id, field1, field2)
    VALUES (:v_id, :p_field1, :p_field2);

    -- Dispatch async job (optional)
    INSERT INTO {schema}.jobs (job_id, job_type, entity_name, record_id, payload)
    SELECT :v_job_id, 'on_create', '{entity}', :v_id,
            OBJECT_CONSTRUCT('field1', :p_field1);
    EXECUTE TASK {schema}.job_runner;

    RETURN OBJECT_CONSTRUCT('success', TRUE, 'id', :v_id);
END;
$$;
```

## Validated Update Procedure

```sql
DEFINE PROCEDURE {schema}.update_{entity}(
    p_id VARCHAR,
    p_field1 VARCHAR,
    p_status VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    v_current_status VARCHAR;
BEGIN
    -- Check record exists
    SELECT status INTO :v_current_status
    FROM {schema}.{entity} WHERE id = :p_id;

    IF (:v_current_status IS NULL) THEN
        RETURN OBJECT_CONSTRUCT(
            'success', FALSE,
            'errors', ARRAY_CONSTRUCT(
                OBJECT_CONSTRUCT('type', 'validation', 'field', 'id',
                                 'code', 'not_found', 'message', 'Record not found')
            )
        );
    END IF;

    -- State transition validation
    IF (:p_status IS NOT NULL AND :v_current_status = 'closed') THEN
        RETURN OBJECT_CONSTRUCT(
            'success', FALSE,
            'errors', ARRAY_CONSTRUCT(
                OBJECT_CONSTRUCT('type', 'state', 'field', 'status',
                                 'code', 'invalid_transition',
                                 'message', 'Cannot change status of a closed record')
            )
        );
    END IF;

    -- Update
    UPDATE {schema}.{entity}
    SET field1 = COALESCE(:p_field1, field1),
        status = COALESCE(:p_status, status),
        updated_at = CURRENT_TIMESTAMP()
    WHERE id = :p_id;

    RETURN OBJECT_CONSTRUCT('success', TRUE, 'id', :p_id);
END;
$$;
```

## Read Procedure (Inline — for fast point lookups)

```sql
DEFINE PROCEDURE {schema}.get_{entity}(p_id VARCHAR)
RETURNS TABLE(id VARCHAR, field1 VARCHAR, status VARCHAR, created_at TIMESTAMP_NTZ)
LANGUAGE SQL
AS
$$
BEGIN ATOMIC
    LET res RESULTSET := (
        SELECT id, field1, status, created_at
        FROM {schema}.{entity} WHERE id = :p_id
    );
    RETURN TABLE(res);
END;
$$;
```

## Error Response Structure

All procs return VARIANT:

```json
// Success
{"success": true, "id": "uuid-here"}

// Validation error (field-level — display under the input)
{"success": false, "errors": [{"type": "validation", "field": "title", "code": "required", "message": "..."}]}

// State transition error (display near workflow controls)
{"success": false, "errors": [{"type": "state", "field": "status", "code": "invalid_transition", "message": "..."}]}

// Multi-field constraint (display as form-level banner)
{"success": false, "errors": [{"type": "constraint", "fields": ["start_date", "end_date"], "code": "invalid_range", "message": "..."}]}

// Permission (display as page-level error)
{"success": false, "errors": [{"type": "permission", "code": "unauthorized", "message": "..."}]}
```

## Counter Table Pattern (for short IDs)

Hybrid tables don't support sequences. Use a counter table:

```sql
DEFINE TABLE {schema}.sequences WITH table_type = 'HYBRID' AS (
    name VARCHAR(100) NOT NULL,
    value NUMBER(18,0) NOT NULL DEFAULT 0,
    PRIMARY KEY (name)
);

-- In a proc:
UPDATE {schema}.sequences SET value = value + 1 WHERE name = 'ticket_seq';
SELECT 'OMS-' || value::VARCHAR INTO :v_short_id
    FROM {schema}.sequences WHERE name = 'ticket_seq';
```

## Job Dispatch Pattern

```sql
-- Insert job record
INSERT INTO {schema}.jobs (job_id, job_type, entity_name, record_id, payload)
SELECT :v_job_id, 'job_type_name', 'entity', :v_record_id,
        OBJECT_CONSTRUCT('key', 'value');

-- Kick off the task immediately
EXECUTE TASK {schema}.job_runner;
```

## Job Runner Task

The task runs on a 1-hour schedule as a fallback safety net. Normal job processing
happens immediately via `EXECUTE TASK` called from within stored procedures — the
schedule should rarely fire under normal operation.

```sql
DEFINE TASK {schema}.job_runner
    WAREHOUSE = '{warehouse}'
    SCHEDULE = '60 MINUTE'
    STARTED
AS
    CALL {schema}.process_pending_jobs();
```

## Job Processor

```sql
DEFINE PROCEDURE {schema}.process_job(p_config VARIANT)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_job_id VARCHAR;
    v_job_type VARCHAR;
    v_payload VARIANT;
BEGIN
    v_job_id := p_config:job_id::VARCHAR;

    UPDATE {schema}.jobs SET status = 'running', started_at = CURRENT_TIMESTAMP(),
        attempts = attempts + 1
    WHERE job_id = :v_job_id AND status = 'pending';

    SELECT job_type, payload INTO :v_job_type, :v_payload
    FROM {schema}.jobs WHERE job_id = :v_job_id;

    -- Dispatch by type
    IF (:v_job_type = 'job_type_1') THEN
        CALL {schema}.handle_type_1(:v_job_id, :v_payload);
    ELSEIF (:v_job_type = 'job_type_2') THEN
        CALL {schema}.handle_type_2(:v_job_id, :v_payload);
    END IF;

    UPDATE {schema}.jobs SET status = 'completed', completed_at = CURRENT_TIMESTAMP()
    WHERE job_id = :v_job_id;

    RETURN 'completed';
EXCEPTION
    WHEN OTHER THEN
        UPDATE {schema}.jobs SET status = 'failed', last_error = SQLERRM
        WHERE job_id = :v_job_id;
        RETURN 'failed: ' || SQLERRM;
END;
$$;
```
