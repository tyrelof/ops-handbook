# RDS (DB Triage & Restore Drill)

## Max connections
```sql
SHOW PROCESSLIST;
```

## Fix
- Kill idle
- Pooling
- Reduce concurrency

## Restore drill
1. Identify last good snapshot.
2. Restore to new instance.
3. Update app secret.
4. Restart app pods.
5. Verify data integrity.
