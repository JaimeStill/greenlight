# PostgreSQL Container Documentation

## Overview

This project uses PostgreSQL 16 (Alpine variant) running in a Docker container for database services. The database data is persisted in a local volume at `.data/postgres` which is excluded from version control.

## Container Configuration

The PostgreSQL container is configured via `docker-compose.yml` with the following settings:

- **PostgreSQL Version**: 16-alpine
- **Container Name**: greenlight-postgres
- **Database Name**: greenlight
- **Port**: 5432 (mapped to host)
- **Data Volume**: `./.data/postgres:/var/lib/postgresql/data`
- **SQL Scripts Volume**: `./sql:/opt/sql:ro` (read-only)

### Credentials

- **Superuser**: postgres / postgres (PostgreSQL default)
- **Application User**: greenlight / pa55word
- **Database**: greenlight

## Starting the PostgreSQL Container Stack

### Start the container in detached mode (recommended)
```bash
docker compose up -d
```

### Start with logs visible
```bash
docker compose up
```

### Verify the container is running
```bash
docker compose ps
```

## Connecting to PostgreSQL

### Connect via Docker Exec

#### As postgres superuser
```bash
docker exec -it greenlight-postgres psql -U postgres -d postgres
```

#### As application user to greenlight database
```bash
docker exec -it greenlight-postgres psql -U greenlight -d greenlight
```

### Connect from Host Machine

If you have PostgreSQL client installed locally:

```bash
# As postgres superuser
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres

# As application user to greenlight database
psql -h 127.0.0.1 -p 5432 -U greenlight -d greenlight
```

### Connect from Go Application

```go
// Connection string format
dsn := "postgres://greenlight:pa55word@localhost:5432/greenlight?sslmode=disable"

// Alternative format
dsn := "host=localhost port=5432 user=greenlight password=pa55word dbname=greenlight sslmode=disable"

// Example connection
db, err := sql.Open("postgres", dsn)
```

## Managing the PostgreSQL Container Stack

### Container Lifecycle Commands

#### Stop the container
```bash
docker compose stop
```

#### Stop and remove the container
```bash
docker compose down
```

#### Stop, remove container AND delete volume data (WARNING: Data loss!)
```bash
docker compose down -v
rm -rf .data/postgres
```

#### Restart the container
```bash
docker compose restart
```

### View Container Logs

#### View all logs
```bash
docker compose logs postgres
```

#### Follow logs in real-time
```bash
docker compose logs -f postgres
```

#### View last 100 lines
```bash
docker compose logs --tail=100 postgres
```

### Container Resource Management

#### Check container resource usage
```bash
docker stats greenlight-postgres
```

#### Inspect container details
```bash
docker inspect greenlight-postgres
```

## Working with SQL Scripts

### Executing SQL Scripts from psql Prompt

SQL scripts are mounted read-only at `/opt/sql` inside the container. Once connected to PostgreSQL, execute scripts using:

```sql
-- Execute a script file
\i /opt/sql/create-database.sql

-- Note: PostgreSQL does NOT support 'source' command
```

### List Available Scripts

From the container shell:
```bash
docker exec -it greenlight-postgres ls -la /opt/sql/
```

### Important Notes About SQL Scripts

- Scripts are mounted **read-only** and persist even with `docker compose down -v`
- Scripts are **NOT** automatically executed on container startup (by design)
- All files in the local `./sql/` directory are available at `/opt/sql/` in the container
- Use the `\i` command to execute scripts manually
- You can use `\c database_name` within scripts to switch databases

## Automatic Script Execution

> Note: this feature is not implemented and is only provided for reference.

### About /docker-entrypoint-initdb.d

The PostgreSQL Docker image provides a special directory `/docker-entrypoint-initdb.d` that automatically executes scripts during **initial container creation** (not on restarts). This feature is **not currently configured** in our setup to maintain manual control over script execution.

### How to Enable Automatic Execution

If you wanted to enable automatic script execution, you would modify the volume mount in `docker-compose.yml`:

```yaml
volumes:
  - ./.data/postgres:/var/lib/postgresql/data
  - ./sql:/docker-entrypoint-initdb.d:ro  # Auto-execute on creation
```

### Automatic Execution Behavior

When using `/docker-entrypoint-initdb.d`:

- **Execution Order**: Files are executed in alphabetical order
- **File Types Supported**:
  - `.sql` files - Executed by psql
  - `.sh` files - Executed as shell scripts
  - `.sql.gz` files - Decompressed and executed
- **Timing**: Only runs during **initial container creation**
- **One-Time Only**: Scripts do not re-run on container restarts
- **Database Context**: SQL scripts run as the superuser (postgres)

### Why We Use Manual Execution Instead

Our current configuration uses `/opt/sql` for manual execution because:

- **Explicit Control**: You decide when to run scripts
- **Repeatability**: Scripts can be re-run as needed
- **Development Flexibility**: No need to recreate containers to re-run scripts
- **Debugging**: Easier to troubleshoot script issues manually
- **Selective Execution**: Run only specific scripts when needed

## Working with psql CLI

### Basic psql Commands

Once connected to PostgreSQL, you can use these commands:

```sql
-- List all databases
\l

-- Connect to a specific database
\c greenlight

-- List all tables in current database
\dt

-- List all tables with more details
\dt+

-- Describe table structure
\d table_name

-- Describe table with more details (indexes, triggers, etc.)
\d+ table_name

-- Show current user
SELECT current_user;

-- Show current database
SELECT current_database();

-- Show PostgreSQL version
SELECT version();

-- List all users/roles
\du

-- Show all schemas
\dn

-- Exit psql CLI
\q
```

### Database Management

```sql
-- Create a new database
CREATE DATABASE dbname;

-- Drop a database (WARNING: Permanent!)
DROP DATABASE IF EXISTS dbname;

-- Show database size
SELECT
    pg_database.datname AS database_name,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
WHERE datname = 'greenlight';

-- Show all table sizes in current database
SELECT
    schemaname AS schema,
    tablename AS table,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### User Management

```sql
-- Create a new user
CREATE USER username WITH PASSWORD 'password';

-- Grant privileges on database
GRANT ALL PRIVILEGES ON DATABASE greenlight TO username;

-- Grant privileges on all tables in schema
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO username;

-- Show user privileges
\dp  -- List table privileges

-- Create a user with specific privileges
CREATE USER readonly_user WITH PASSWORD 'password';
GRANT CONNECT ON DATABASE greenlight TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

## PostgreSQL Best Practices

### 1. Connection Pooling

When connecting from your Go application, use connection pooling:

```go
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

### 2. Query Optimization

- Always use indexes on columns used in WHERE, JOIN, and ORDER BY clauses
- Use EXPLAIN ANALYZE to analyze query performance:
  ```sql
  EXPLAIN ANALYZE SELECT * FROM movies WHERE year > 2020;
  ```

### 3. Security Guidelines

- Never store credentials in code; use environment variables
- Always use parameterized queries to prevent SQL injection
- Use SSL connections in production (set `sslmode=require`)
- Regularly update passwords and use strong passwords
- Limit user privileges to only what's necessary
- Consider using row-level security for multi-tenant applications

### 4. Backup Strategies

#### Manual backup
```bash
# Backup database to file
docker exec greenlight-postgres pg_dump -U postgres greenlight > backup_$(date +%Y%m%d).sql

# Backup with compression
docker exec greenlight-postgres pg_dump -U postgres greenlight | gzip > backup_$(date +%Y%m%d).sql.gz

# Backup in custom format (most flexible)
docker exec greenlight-postgres pg_dump -U postgres -Fc greenlight > backup_$(date +%Y%m%d).dump
```

#### Restore from backup
```bash
# Restore from SQL file
docker exec -i greenlight-postgres psql -U postgres greenlight < backup.sql

# Restore from compressed file
gunzip < backup.sql.gz | docker exec -i greenlight-postgres psql -U postgres greenlight

# Restore from custom format
docker exec -i greenlight-postgres pg_restore -U postgres -d greenlight < backup.dump
```

### 5. Performance Monitoring

```sql
-- Show running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';

-- Check for blocking queries
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.usename AS blocking_user
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
WHERE NOT blocked_locks.granted;

-- Monitor connections
SELECT count(*) as connections,
       state
FROM pg_stat_activity
GROUP BY state;

-- Check table statistics
SELECT * FROM pg_stat_user_tables WHERE schemaname = 'public';
```

### 6. Data Integrity

- Always use transactions for related operations
- Define foreign key constraints where appropriate
- Use appropriate data types (consider using PostgreSQL-specific types like UUID, JSONB, arrays)
- Set NOT NULL constraints where data is required
- Use CHECK constraints for data validation

### 7. Development vs Production

**Development Settings** (current docker-compose.yml):
- Simple passwords for convenience
- Direct port exposure (5432)
- SSL disabled (`sslmode=disable`)
- Single container setup

**Production Recommendations**:
- Use strong, randomized passwords
- Store credentials in secrets management
- Use private networks, not exposed ports
- Enable SSL/TLS (`sslmode=require` or `sslmode=verify-full`)
- Implement regular automated backups with point-in-time recovery
- Configure replication for high availability
- Monitor performance metrics with pg_stat_statements
- Set up alerting for issues
- Use connection pooling (PgBouncer or application-level)

## Troubleshooting

### Container won't start
```bash
# Check logs for errors
docker compose logs postgres

# Ensure port 5432 is not already in use
lsof -i :5432
```

### Can't connect to PostgreSQL
```bash
# Verify container is running
docker compose ps

# Test connection from within container
docker exec -it greenlight-postgres psql -U postgres -c "SELECT 1"

# Check if PostgreSQL is accepting connections
docker exec -it greenlight-postgres pg_isready
```

### Data persistence issues
```bash
# Verify volume is mounted
docker inspect greenlight-postgres | grep -A 5 Mounts

# Check permissions on .data directory
ls -la .data/
```

### Reset everything (WARNING: Data loss!)
```bash
docker compose down -v
rm -rf .data/
docker compose up -d
```

## Additional Resources

- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)
- [Docker PostgreSQL Official Image](https://hub.docker.com/_/postgres)
- [PostgreSQL Performance Tuning](https://www.postgresql.org/docs/16/performance-tips.html)
- [PostgreSQL Security Best Practices](https://www.postgresql.org/docs/16/security.html)
- [psql Command Reference](https://www.postgresql.org/docs/16/app-psql.html)