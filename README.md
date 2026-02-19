Verify CFK Migration CR Lock

When a `KRaftMigrationJob` is created, CFK automatically applies a migration lock annotation to the following resources:

* Kafka CR
* ZooKeeper CR
* KRaftController CR

This prevents unintended modifications during migration.

### Check for CR Lock

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
oc get zookeeper zookeeper -n confluent -o yaml | grep kraft-migration-cr-lock
oc get kraftcontroller kraftcontroller -n confluent -o yaml | grep kraft-migration-cr-lock
```

If the lock is present, the output will show:

```yaml
platform.confluent.io/kraft-migration-cr-lock: "true"
```

### Interpretation

If present:

* Migration job is actively controlling the resources.
* CR modifications are restricted.
* This is expected behavior.
* No action is required during migration.

---

# 8. Post-Migration Cleanup

## 🔓 Remove CFK Migration Lock

After migration phase reaches:

```
Phase: COMPLETE
```

The migration lock must be manually released to allow future CR updates and upgrades.

### Release Lock

```bash
oc annotate kraftmigrationjob kraftmigrationjob \
platform.confluent.io/kraft-migration-release-cr-lock=true \
--namespace confluent
```

### Verify Lock Removal

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
```

The annotation should no longer exist in:

* Kafka CR
* ZooKeeper CR
* KRaftController CR

### Important

Failure to remove the migration lock may:

* Prevent future configuration updates
* Block version upgrades
* Interfere with GitOps or CI/CD pipelines

---
