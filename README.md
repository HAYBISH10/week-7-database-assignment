# week-7-database-assignment

# PostgreSQL Audit Logging, Hierarchical Data, Migrations and Security

## Step 1: Build a Reusable Audit Log

First, I created the audit log table:

```sql
CREATE TABLE audit_log (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tbl TEXT,
    op TEXT,
    old_row JSONB,
    new_row JSONB,
    changed_by TEXT DEFAULT current_user,
    at TIMESTAMPTZ DEFAULT now()
);
```

I then created the audit trigger function:

```sql
CREATE FUNCTION audit() 
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log(tbl, op, old_row, new_row)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        to_jsonb(OLD),
        to_jsonb(NEW)
    );

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

Finally, I attached the trigger to the `students` table:

```sql
CREATE TRIGGER trg_audit
AFTER INSERT OR UPDATE OR DELETE
ON students
FOR EACH ROW
EXECUTE FUNCTION audit();
```

The trigger automatically records changes made to the `students` table.

---

## Step 2: Test the Audit Log

I updated one student and deleted another:

```sql
UPDATE students
SET name = 'Kofi M.'
WHERE id = 1;

DELETE FROM students
WHERE id = 3;
```

I then checked the audit trail:

```sql
SELECT
    tbl,
    op,
    old_row->>'name' AS was,
    new_row->>'name' AS now,
    at
FROM audit_log
ORDER BY at DESC;
```

The audit log records the table name, operation performed, old data, new data, user who made the change, and timestamp.

---

## Step 3: Model a Category Tree

I created a self-referencing `categories` table:

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    parent_id INT REFERENCES categories(id)
);
```

I inserted hierarchical category data:

```sql
INSERT INTO categories (name, parent_id)
VALUES
    ('Electronics', NULL),
    ('Computers', 1),
    ('Laptops', 2),
    ('Phones', 1);
```

I used a recursive CTE to display the hierarchy:

```sql
WITH RECURSIVE tree AS (
    SELECT
        id,
        name,
        parent_id,
        0 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.id,
        c.name,
        c.parent_id,
        t.depth + 1
    FROM categories c
    JOIN tree t
        ON c.parent_id = t.id
)
SELECT
    repeat('  ', depth) || name AS tree
FROM tree
ORDER BY depth, name;
```

The recursive query walks from the parent category to its child categories.

---

## Step 4: Run Versioned Migrations with Flyway

I organized the database migrations in the following order:

```text
migrations/
├── V1__core_tables.sql
├── V2__audit_log.sql
└── V3__categories.sql
```

I then applied the migrations:

```bash
flyway -url=jdbc:postgresql://localhost/bootcamp -user=postgres migrate
```

I checked the migration status using:

```bash
flyway info
```

Flyway records which migrations have already been applied, allowing database schema changes to be managed in a controlled and versioned way.

---

## Step 5: Apply Least-Privilege Security

I created separate roles for reading and writing:

```sql
CREATE ROLE app_read;
CREATE ROLE app_write;
```

I granted database connection access:

```sql
GRANT CONNECT
ON DATABASE bootcamp
TO app_read, app_write;
```

I granted access to the public schema:

```sql
GRANT USAGE
ON SCHEMA public
TO app_read, app_write;
```

The read role was given only `SELECT` permission:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA public
TO app_read;
```

The write role was given the required data modification permissions:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO app_write;
```

Finally, I created an API user and assigned it the write role:

```sql
CREATE USER api
LOGIN PASSWORD 'strong-secret'
IN ROLE app_write;
```

## Conclusion

In this lab, I implemented a PostgreSQL audit logging system using triggers and JSONB, modeled hierarchical categories using a self-referencing table and recursive CTEs, managed database schema changes using Flyway migrations, and applied role-based least-privilege security. These techniques help improve database auditing, organization, maintainability, and security.
