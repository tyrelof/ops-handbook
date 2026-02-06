# Disaster Recovery Playbooks

> DR is not backups.
> DR is **restoring under pressure**.

## RDS Restore Drill
1. Identify last good snapshot
2. Restore to new instance
3. Update app secret
4. Restart app pods
5. Verify data integrity

## Secrets Restore Drill
1. Validate secret source (SSM/Secrets Manager)
2. Force ExternalSecret refresh
3. Restart dependent pods

## Ingress / LB Restore
1. Verify DNS
2. Verify certificate
3. Reapply ingress from Git
4. Validate routing
