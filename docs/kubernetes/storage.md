# Storage & Backups (PVC, Velero, Snapshots)

## Check PVCs
```bash
kubectl get pvc -n <ns>
```

## Check Velero backups
```bash
velero backup get
```

## Restore from snapshot
1. Identify snapshot ID.
2. Create volume from snapshot.
3. Restore PVC from volume.
