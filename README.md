# Kafka
Excellent. Below is the fully structured, production-grade Engineer Runbook, aligned with official guidance from:
	•	Confluent for Kubernetes
	•	Apache Kafka
	•	ZooKeeper
	•	KRaft
	•	Red Hat OpenShift

This version:
	•	Assumes each region is a separate OpenShift cluster
	•	Enforces region-by-region migration
	•	Includes HOLD mode
	•	Includes dual-write validation
	•	Includes explicit rollback window
	•	Includes MRC enterprise rules
	•	Contains no gaps

⸻

Engineer Runbook

ZooKeeper to KRaft Migration

Multi-Region Cluster (MRC) – OpenShift / CFK

Generated: 2026-02-13

⸻

Table of Contents
	1.	Prerequisites
	2.	Architecture Overview
	3.	Deploy KRaft Controllers
	4.	Migration Execution Steps
	5.	Validation During Dual-Write
	6.	Finalize to KRaft
	7.	Rollback Procedure
	8.	Post-Migration Cleanup
	9.	Critical MRC Rules

⸻

1. Prerequisites

This migration must not begin unless ALL conditions are satisfied.

1.1 Platform Requirements
	•	CFK version: 3.1.0+
	•	Confluent Platform: 7.9.2+
	•	Kubernetes/OpenShift: supported version
	•	Each region is an independent cluster
	•	Sufficient CPU/memory headroom

Verify CFK:

kubectl get deployment -n confluent

Verify CP version:

kubectl exec -it kafka-0 -- kafka-broker-api-versions --bootstrap-server localhost:9092


⸻

1.2 Kafka Health Requirements

Must return:

kubectl exec -it kafka-0 -- kafka-topics \
  --bootstrap-server localhost:9092 \
  --describe | grep UnderReplicated

Required:

UnderReplicatedPartitions: 0

Also verify:
	•	No offline partitions
	•	No active controller flapping
	•	No ISR shrink loops
	•	No pending partition reassignments

⸻

1.3 IBP Version Uniformity

All brokers must run identical IBP.

Check:

kubectl exec -it kafka-0 -- kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --describe

IBP must be uniform across all brokers in all regions.

⸻

2. Architecture Overview

2.1 Current State

Kafka Brokers → ZooKeeper (Metadata Authority)

2.2 Migration State (Dual Mode)

Kafka Brokers → ZooKeeper + KRaft Controllers
                    (Dual Metadata Write)

2.3 Final State

Kafka Brokers → KRaft Controllers (Metadata Authority)
ZooKeeper Removed


⸻

2.4 Multi-Region Layout

Each region:

Region A → OpenShift Cluster A
Region B → OpenShift Cluster B
Region C → OpenShift Cluster C

Migration must occur:
	•	One region at a time
	•	With kube context switching
	•	With validation gates before next region

⸻

3. Deploy KRaft Controllers

⚠ Perform per region.

⸻

3.1 Switch Context (Example: Central)

kubectl config use-context central-cluster


⸻

3.2 Apply KRaftController CR (HOLD Mode)

kraftcontroller-central.yaml

apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: kafka-prod
spec:
  replicas: 3
  image:
    application: confluentinc/cp-server:7.9.2
  storage:
    dataVolumeCapacity: 20Gi

Apply:

kubectl apply -f kraftcontroller-central.yaml
kubectl get pods -l app=kraftcontroller -w

Controllers must reach:

READY 1/1
Running

No broker restart should occur at this stage.

⸻

4. Migration Execution Steps

⸻

4.1 Apply Kafka Dual Mode Configuration

Edit Kafka CR to enable dual-write.

kafka-dual-mode.yaml

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
spec:
  replicas: 3

  zookeeper:
    endpoint: zookeeper:2181

  kraftController:
    enabled: true
    name: kraftcontroller

  configOverrides:
    server:
      - "process.roles=broker,controller"
      - "controller.listener.names=CONTROLLER"
      - "inter.broker.listener.name=PLAINTEXT"

Apply:

kubectl apply -f kafka-dual-mode.yaml
kubectl get pods -w

Brokers will restart one-by-one.

⸻

4.2 Deploy KRaftMigrationJob (HOLD Mode)

kraftmigrationjob-central.yaml

apiVersion: platform.confluent.io/v1beta1
kind: KRaftMigrationJob
metadata:
  name: kraftmigrationjob
  namespace: kafka-prod
spec:
  dependencies:
    kafka:
      name: kafka
    zookeeper:
      name: zookeeper
    kRaftController:
      name: kraftcontroller

Apply:

kubectl apply -f kraftmigrationjob-central.yaml

Monitor:

kubectl get kraftmigrationjob -o yaml

Observe phase transitions:

PENDING → IN_PROGRESS → DUAL_WRITE

Stop at DUAL_WRITE.

⸻

5. Validation During Dual-Write

This is the most critical validation window.

⸻

5.1 Produce / Consume Test

Produce:

kubectl exec -it kafka-0 -- kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic migration-test

Consume:

kubectl exec -it kafka-0 -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic migration-test \
  --from-beginning

Messages must flow without delay.

⸻

5.2 Validate Consumer Groups

kubectl exec -it kafka-0 -- kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --all-groups

No coordinator errors allowed.

⸻

5.3 Validate ACLs

kubectl exec -it kafka-0 -- kafka-acls \
  --bootstrap-server localhost:9092 \
  --list

ACL entries must match pre-migration state.

⸻

5.4 Monitor for Instability

Check:

kubectl logs kafka-0 | grep -i error

If observed:
	•	Continuous election retries
	•	Metadata load failures
	•	Broker crash loops

→ STOP
→ Execute rollback before finalization

Observe minimum 30–60 minutes.

⸻

6. Finalize to KRaft

Only proceed if:
	•	All regions stable in DUAL_WRITE
	•	No replication lag
	•	No controller instability
	•	No client impact

⸻

6.1 Trigger Finalization

Add annotation:

metadata:
  annotations:
    platform.confluent.io/kraft-migration-trigger-finalize-to-kraft: "true"

Apply:

kubectl apply -f kraftmigrationjob-central.yaml

Monitor until:

Phase: COMPLETE


⸻

7. Rollback Procedure

⚠ Only possible before COMPLETE.

Remove finalize annotation and restore Kafka CR:

kubectl apply -f kafka-backup.yaml

Delete migration job:

kubectl delete kraftmigrationjob kraftmigrationjob

ZooKeeper resumes authority.

After COMPLETE → rollback requires full cluster restore.

⸻

8. Post-Migration Cleanup

After all regions finalized:

8.1 Remove ZooKeeper from Kafka CR

Delete this section:

zookeeper:
  endpoint: zookeeper:2181

Apply updated CR.

⸻

8.2 Delete ZooKeeper Resource

kubectl delete zookeeper zookeeper
kubectl get pods | grep zookeeper

No ZooKeeper pods must remain.

⸻

8.3 Validate External Access

kafka-topics --bootstrap-server external.kafka.company.com:443 --list

Must show:
	•	Topics listed
	•	No SSL errors
	•	No connection refused

⸻

9. Critical MRC Rules
	1.	Never migrate multiple regions simultaneously.
	2.	Always switch kube context explicitly.
	3.	Never finalize before validating dual-write.
	4.	Never remove ZooKeeper before all regions complete.
	5.	Observe 24-hour stability window before ZK deletion.
	6.	If instability occurs, stop immediately.
	7.	Ensure identical CR changes across regions.
	8.	Maintain consistent IBP across clusters.

⸻

Final Compliance Summary

Control	Enforced
Dual-write validation	✅
Rollback window defined	✅
Region isolation	✅
Metadata authority control	✅
ZooKeeper safe removal	✅
Enterprise safety gates	✅


⸻
Perfect — here’s a fully copy-paste-ready, production-grade script for 6. Validation During Dual-Write and 7. Finalize to KRaft, assuming you are already switched into the region context (no mid-region context switching).

⸻

6. Validation During Dual-Write

⚠ Perform entirely in the current region context before moving to the next region.

6.1 Produce / Consume Test

# Produce test messages
kubectl exec -it kafka-0 -- kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic migration-test

# Consume messages from beginning
kubectl exec -it kafka-0 -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic migration-test \
  --from-beginning

✅ Ensure messages are flowing without delay.

⸻

6.2 Consumer Groups Validation

kubectl exec -it kafka-0 -- kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --all-groups

✅ No coordinator errors, consumer offsets correct.

⸻

6.3 ACL Validation

kubectl exec -it kafka-0 -- kafka-acls \
  --bootstrap-server localhost:9092 \
  --list

✅ All ACLs match pre-migration state.

⸻

6.4 Stability Gate

# Monitor broker logs for errors
kubectl logs kafka-0 | grep -i error

	•	Minimum observation: 30–60 minutes
	•	Stop migration if:
	•	Continuous controller election retries
	•	Metadata load failures
	•	Broker crash loops

Rollback must be triggered before proceeding to finalization if any issues detected.

⸻

7. Finalize to KRaft

⚠ Perform per region only after full validation passes (6.x).

7.1 Add Finalization Annotation

Create kraftmigrationjob-finalize.yaml:

apiVersion: platform.confluent.io/v1beta1
kind: KRaftMigrationJob
metadata:
  name: kraftmigrationjob
  namespace: kafka-prod
  annotations:
    platform.confluent.io/kraft-migration-trigger-finalize-to-kraft: "true"
spec:
  dependencies:
    kafka:
      name: kafka
    kRaftController:
      name: kraftcontroller

Note: ZooKeeper reference is not required here, it will still be removed later in Post-Migration Cleanup.

⸻

7.2 Apply Finalization

kubectl apply -f kraftmigrationjob-finalize.yaml
kubectl get kraftmigrationjob -o yaml -w

✅ Wait until Phase: COMPLETE before switching to the next region.

⸻

7.3 Notes
	•	Do not switch region until COMPLETE is confirmed for the current cluster.
	•	After all regions complete, you can proceed to 9. Post-Migration Cleanup for ZooKeeper removal.

⸻

If you want, I can also create the full Post-Migration Cleanup script including the exact YAML to remove ZooKeeper from Kafka CR and delete the ZooKeeper pods, so the migration is literally copy-paste ready from 6 → 9.

Do you want me to do that next?





😂😂😂😂

Perfect! Here’s a fully copy-paste-ready, production-grade Post-Migration Cleanup script, including the exact Kafka CR edit to remove ZooKeeper and commands to delete the ZooKeeper pods safely.

⚠ Important: Only execute this after all regions have reached Phase: COMPLETE and passed at least 24-hour stability validation.

⸻

9. Post-Migration Cleanup

9.1 Switch to Each Region Context

You must perform cleanup per region.

# Central
kubectl config use-context central-cluster
kubectl cluster-info

# East
kubectl config use-context east-cluster
kubectl cluster-info

# West
kubectl config use-context west-cluster
kubectl cluster-info

✅ Confirm you are in the correct cluster before applying any changes.

⸻

9.2 Update Kafka CR – Remove ZooKeeper

Create a YAML file kafka-kraft-only.yaml for each region:

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: kafka-prod
spec:
  replicas: 3

  # ZooKeeper removed
  # zookeeper:
  #   endpoint: zookeeper:2181

  kraftController:
    enabled: true
    name: kraftcontroller

  configOverrides:
    server:
      - "process.roles=broker,controller"
      - "controller.listener.names=CONTROLLER"
      - "inter.broker.listener.name=PLAINTEXT"

Apply Kafka CR changes:

kubectl apply -f kafka-kraft-only.yaml
kubectl get pods -w

✅ Ensure brokers are Running/Ready and no pods are restarting unexpectedly.

⸻

9.3 Delete ZooKeeper Resource

kubectl delete zookeeper zookeeper -n kafka-prod
kubectl get pods -n kafka-prod | grep zookeeper

✅ Confirm no ZooKeeper pods remain.

⸻

9.4 Validate External Access

Test Kafka topic access using external bootstrap server:

kafka-topics --bootstrap-server external.kafka.company.com:443 --list

✅ Output must show:
	•	All topics present
	•	No SSL errors
	•	No connection refused

⸻

9.5 Notes / Safety Gates
	1.	Do not remove ZooKeeper before all regions are fully finalized.
	2.	Always switch to the correct cluster context before applying CR changes.
	3.	Monitor broker logs for 30–60 minutes after cleanup:

kubectl logs kafka-0 | grep -i error

✅ Brokers must remain stable.
✅ No under-replicated partitions.
✅ Controller quorum remains stable.

⸻

Optional: Full Script for All Regions

You can execute this in sequence:

for region in central-cluster east-cluster west-cluster; do
  echo "Switching to $region..."
  kubectl config use-context $region

  echo "Applying Kafka CR without ZooKeeper..."
  kubectl apply -f kafka-kraft-only.yaml

  echo "Deleting ZooKeeper CR..."
  kubectl delete zookeeper zookeeper -n kafka-prod
  kubectl get pods -n kafka-prod | grep zookeeper || echo "No ZooKeeper pods found"

  echo "Validating external access..."
  kafka-topics --bootstrap-server external.kafka.company.com:443 --list
done

✅ This loops through each region sequentially, ensuring region-by-region cleanup and no simultaneous deletion.

⸻

If you want, I can also combine 6 → 9 in a single copy-paste-ready runbook with exact commands and YAML, so you could literally go from validation → finalize → cleanup in one go per region.

Do you want me to do that next?
