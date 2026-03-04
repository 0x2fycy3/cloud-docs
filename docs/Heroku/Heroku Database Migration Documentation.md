# Heroku Database Migration Documentation

**Application:** `yourappname`  
**Platform:** Heroku  
**Database:** Heroku Postgres
**Migration Date:** 2026-03-01

---

## 1. Objective

Migrate the production database from:

| Parameter | Value |
|-----------|-------|
| Plan | Essential-2 |
| PostgreSQL Version | 15.14 |
| Storage | 32 GB |
| Connections | 40 |

To:

| Parameter | Value |
|-----------|-------|
| Plan | Standard-0 |
| PostgreSQL Version | 17.5 |
| Storage | 64 GB |
| Connections | 200 |
| Continuous Protection | Enabled |

### Migration Goals

- Improve performance and scalability
- Increase connection limits
- Enable continuous protection and enhanced production features
- Upgrade PostgreSQL from v15 to v17

---

## 2. Initial Environment

### Current Production Database

View current database info:

```bash
heroku pg:info --app yourappname
```

View current addons:

```bash
heroku addons --app yourappname
```

**Configuration:**

- **Alias:** `DATABASE_URL`
- **Add-on:** `postgresql-regular-95733`

**Key Metrics:**

- Data Size: 2.34 GB
- Tables: 12
- Active Connections: ~20/40
- Continuous Protection: Disabled

---

## 3. Target Environment

**New database provisioned:**

- **Alias:** `HEROKU_POSTGRESQL_OLIVE_URL`
- **Add-on:** `postgresql-contoured-66774`

**Key Features:**

- Plan: Standard-0
- Storage: 64 GB
- Connections: 200
- PostgreSQL Version: 17.5
- Continuous Protection: Enabled
- Automatic Rollback Support: Enabled

---

## 4. Migration Strategy

Since the Essential plan does not support followers, the migration was executed using:

```bash
heroku pg:copy
```

**Method characteristics:**

- Locks the source database during copy
- Copies schema and data
- Handles PostgreSQL version upgrade automatically
- Ensures transactional consistency

**Estimated downtime:** 3–15 minutes (based on 2.34 GB size)

> **IMPORTANT:** Before beginning, notify your team members about the procedure

---

## 5. Execution Steps

### 5.1 Provision Standard-0 Database

```bash
heroku addons:create heroku-postgresql:standard-0 --app yourappname
```

Wait until status becomes `Available`.

### 5.2 Enable Maintenance Mode

To prevent write operations during migration:

```bash
heroku maintenance:on --app yourappname
```

### 5.3 Copy Database

```bash
heroku pg:copy DATABASE_URL HEROKU_POSTGRESQL_OLIVE_URL \
  --app yourappname \
  --confirm yourappname
```

**Result:**

```
Starting copy of DATABASE to OLIVE... done
Copying... done
```

Completed successfully.

### 5.4 Promote New Database

```bash
heroku pg:promote HEROKU_POSTGRESQL_OLIVE_URL --app yourappname
```

System automatically reassigned the new database as `DATABASE_URL`.

### 5.5 Restart Application

```bash
heroku restart --app yourappname
```

### 5.6 Disable Maintenance Mode

```bash
heroku maintenance:off --app yourappname
```

---

## 6. Post-Migration Verification

### Database Verification

```bash
heroku pg:info --app yourappname
```

**Verified:**

- `DATABASE_URL` now points to Standard-0
- Data size: 2.25 GB
- Tables: 12
- PostgreSQL Version: 17.5
- Connections: 29/200
- Continuous Protection: Enabled
- Rollback capability available

### Application Logs Monitoring

```bash
heroku logs --tail --app yourappname
```

No runtime or connection errors detected.

### 6.1 Functional Validation

**Check website availability.**

Request your team members to perform a full functional validation of the system.

They should verify all business-critical workflows and confirm that no regression or unexpected behavior is observed.

---

## 7. Final Architecture State

### Primary Database

- **Alias:** `DATABASE_URL`
- **Plan:** Standard-0
- **PostgreSQL:** 17.5
- **Continuous Protection:** Enabled
- **Configuration:** Production-grade

### Previous Database (Secondary)

- **Alias:** `HEROKU_POSTGRESQL_GREEN_URL`
- **Plan:** Essential-2
- **PostgreSQL:** 15.14
- **Status:** Retained temporarily for rollback safety

---

## 8. Rollback Plan

If a critical issue occurs, execute the following:

```bash
heroku pg:promote HEROKU_POSTGRESQL_GREEN_URL --app yourappname
heroku restart --app yourappname
```

This will instantly restore the previous database.

---

## 9. Recommended Next Step

After **24–48 hours** of stable production monitoring:

```bash
heroku addons:destroy postgresql-regular-95733 --app yourappname
```

This removes the old Essential-2 database and stops billing.

---

## 10. Improvements Achieved

| Feature | Before | After |
|---------|--------|-------|
| Plan | Essential-2 | Standard-0 |
| PostgreSQL Version | 15.14 | 17.5 |
| Connections | 40 | 200 |
| Continuous Protection | No | Yes |
| Rollback Support | No | Yes |
| Storage | 32 GB | 64 GB |

---

## 11. Risk Assessment

- Data integrity preserved
- Schema fully migrated
- No failed migrations detected
- Application successfully restarted
- Production verified

Downtime was controlled via maintenance mode.

---

## 12. Conclusion

The migration of the `yourappname` production database from **Essential-2** to **Standard-0** was successfully completed using a controlled maintenance window and the `pg:copy` method.

### Benefits Achieved

- Increased scalability
- Higher connection limits (5x increase)
- Continuous data protection
- PostgreSQL version upgrade (15.14 → 17.5)
- Production-grade reliability
- Rollback capabilities enabled

**Status:** Migration completed successfully with zero data loss and minimal downtime.
