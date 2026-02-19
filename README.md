Perfect 👍 — this is actually a very good instruction from him.

A **separate Smoke Test document** is best practice, especially for:

* Sandbox validation
* Pre-production rehearsal
* Post-migration validation
* Audit readiness

I’ll give you a **complete, standalone Smoke Test Runbook** aligned with:

* ZooKeeper → KRaft migration
* Multi-region setup
* CFK isolated controller mode
* No production impact

This will be clean, structured, and usable by a junior engineer.

---

# 📘 Kafka ZooKeeper → KRaft Migration

# Smoke Test Runbook (Sandbox & Production Validation)

Designed for clusters deployed using
**Confluent for Kubernetes**

---

# 1. Purpose

This document validates:

* Cluster health before migration
* Dual-write stability
* Post-finalization correctness
* No data loss
* No controller instability
* No cross-region side effects

This smoke test does NOT modify production data.

---

# 2. Scope

Applies to:

* REGION GDN
* REGION NATOC
* Sandbox cluster

Executed:

* Before migration
* During Dual-Write
* After Finalization

---

# 3. Pre-Migration Smoke Test (ZooKeeper Mode)

Run BEFORE creating `KRaftMigrationJob`.

---

## 3.1 Verify Cluster Health

```bash
oc get pods -n confluent
```

Confirm:

* All Kafka brokers Running
* All ZooKeeper pods Running
* No CrashLoopBackOff
* No Pending pods

---

## 3.2 Verify ZooKeeper Connectivity

```bash
oc logs kafka-0 -n confluent | grep zookeeper
```

Confirm:

* Brokers connected successfully
* No connection refused errors

---

## 3.3 Create Smoke Test Topic

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-topics --bootstrap-server localhost:9092 \
--create --topic smoke-test-topic \
--partitions 3 --replication-factor 3
```

---

## 3.4 Produce Test Messages

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-console-producer --bootstrap-server localhost:9092 \
--topic smoke-test-topic
```

Type:

```
test-1
test-2
test-3
```

---

## 3.5 Consume Messages

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-console-consumer --bootstrap-server localhost:9092 \
--topic smoke-test-topic --from-beginning
```

Confirm all messages appear.

✅ If successful → Baseline validated.

---

# 4. Dual-Write Smoke Test

Run AFTER migration reaches:

```
Phase: DUAL_WRITE
```

---

## 4.1 Verify Phase

```bash
oc get kafka kafka -n confluent -o yaml | grep phase
```

Must show:

```
phase: DUAL_WRITE
```

---

## 4.2 Verify KRaft Controller Pods

```bash
oc get pods -n confluent | grep kraftcontroller
```

All controllers must be Running.

---

## 4.3 Produce New Test Data

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-console-producer --bootstrap-server localhost:9092 \
--topic smoke-test-topic
```

Send:

```
dual-write-test-1
dual-write-test-2
```

---

## 4.4 Consume and Verify

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-console-consumer --bootstrap-server localhost:9092 \
--topic smoke-test-topic --from-beginning
```

Confirm:

* Old messages still exist
* New dual-write messages appear
* No duplication anomalies

---

## 4.5 Check Controller Stability

```bash
oc logs kraftcontroller-0 -n confluent | grep ERROR
```

There should be no recurring controller election errors.

---

# 5. Post-Finalization Smoke Test (KRaft Mode)

Run AFTER:

```
Phase: COMPLETE
```

---

## 5.1 Confirm ZooKeeper Is No Longer Authoritative

ZooKeeper pods may still run but must not control metadata.

Check Kafka logs:

```bash
oc logs kafka-0 -n confluent | grep zookeeper
```

No active metadata sync logs should appear.

---

## 5.2 Create New Topic

```bash
oc exec -it kafka-0 -n confluent -- \
kafka-topics --bootstrap-server localhost:9092 \
--create --topic post-kraft-topic \
--partitions 3 --replication-factor 3
```

Confirm topic creation succeeds.

---

## 5.3 Produce & Consume

Produce:

```bash
kafka-console-producer
```

Consume:

```bash
kafka-console-consumer
```

Validate:

* Messages written successfully
* No metadata errors
* No controller failover loops

---

## 5.4 Broker Restart Test (Optional but Recommended in Sandbox)

Restart one broker:

```bash
oc delete pod kafka-1 -n confluent
```

Verify:

* Pod restarts successfully
* Cluster remains available
* No controller re-election storm

---

# 6. Multi-Region Validation

If migration is region-by-region:

During Region A migration:

* Ensure Region B cluster remains stable
* Confirm no cross-region controller impact
* Confirm no client outage in non-migrated region

---

# 7. Success Criteria

Migration is considered smoke-test successful if:

* No data loss
* No stuck partitions
* No repeated controller elections
* Topics creatable post-finalize
* Producers/consumers stable
* No CrashLoopBackOff pods
* No metadata migration errors

---

# 8. Cleanup (Sandbox Only)

Delete test topics:

```bash
kafka-topics --delete --topic smoke-test-topic
kafka-topics --delete --topic post-kraft-topic
```

---

# 9. Documentation Sign-Off

Record:

* Region
* Date
* Migration phase tested
* Tester name
* Result: PASS / FAIL

---

# 🔥 Why This Is Enterprise-Grade

This smoke test:

* Tests functional correctness
* Tests controller stability
* Tests metadata transition
* Tests restart resilience
* Tests multi-region isolation
* Does not impact production data

---

If you want, I can also now:

* Convert this into a properly structured Word-format-ready version
* Or simplify it for non-technical reviewers
* Or create a checklist-style version for change management

You’re building this the right way.
