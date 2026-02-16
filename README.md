Engineer Runbook

ZooKeeper to KRaft Migration


Table of Contents
	1.	Prerequisites
	2.	Architecture Overview
	3.	Deploy KRaft Controllers
	4.	Migration Execution Steps
	5.	Validation During Dual-Write
	6.	Finalize KRaft
	7.	Rollback Procedure
	8.	Post-Migration Cleanup
	9.	Critical MRC Rules

⸻

1. Prerequisites

This migration must not begin unless all conditions are satisfied.

1.1 Platform Requirements
	•	CFK version 3.1.0+
	•	Confluent Platform 7.9.2+
	•	Kubernetes/OpenShift supported version
	•	Each region is an independent cluster
	•	Sufficient CPU/memory headroom

Confirm Namespace

oc project kafka-prod

Confirm Kafka & ZooKeeper Pods

oc get kafka
oc get zookeeper
oc get pods

Backup Kafka & ZooKeeper CRs

oc get kafka kafka -o yaml > kafka-backup.yaml
oc get zookeeper zookeeper -o yaml > zk-backup.yaml

Verify CFK

oc get pods -n confluent

Verify CP Version

oc exec -it kafka-0 -- kafka-broker-api-versions --bootstrap-server localhost:9092

Kafka Health Requirements

oc exec -it kafka-0 -- kafka-topics --bootstrap-server localhost:9092 --describe | grep UnderReplicated
oc logs kafka-0 | grep controller

Required:

UnderReplicatedPartitions: 0

Also verify:
	•	No Offline partitions
	•	No active controller flapping
	•	No ISR shrink loops
	•	No pending partition reassignments

⸻

1.2 IBP Version Uniformity

All brokers must run identical IBP.

oc exec -it kafka-0 -- kafka-configs --bootstrap-server localhost:9092 --entity-type brokers --describe

IBP must be uniform across all brokers in all regions.

⸻

2. Architecture Overview

2.1 Current State

Kafka Brokers → ZooKeeper (Metadata authority)

2.2 Migration State (Dual Mode)

Kafka Brokers → ZooKeeper + KRaft Controllers (Dual Metadata Write)

2.3 Final State

Kafka Brokers → KRaft Controllers (Metadata Authority)
ZooKeeper removed

2.4 Multi-Region Layout

Each region:
	•	Region A → OpenShift Cluster A
	•	Region B → OpenShift Cluster B
	•	Region C → OpenShift Cluster C

⸻

3. Deploy KRaft Controllers

Performed per region.

⸻

REGION 1

3.1 Switch Context (Central)

oc config use-context central-cluster
oc cluster-info

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

oc apply -f kraftcontroller-central.yaml
oc get pods -l app=kraftcontroller -w

Controller must reach:

READY 1/1 Running

No broker restart should occur at this stage.

⸻

4. Migration Execution Steps

4.1 Apply Kafka Dual Mode Configuration

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

oc apply -f kafka-dual-mode.yaml
oc get pods -w

Brokers will restart one-by-one.

⸻

5. Deploy KRaftMigrationJob (HOLD Mode)

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

oc apply -f kraftmigrationjob-central.yaml
oc get kraftmigrationjob -o yaml

Observe:

PENDING → IN_PROGRESS → DUAL_WRITE

Stop at DUAL_WRITE.

⸻

6. Validation During Dual-Write

Produce

oc exec -it kafka-0 -- \
kafka-console-producer \
--bootstrap-server localhost:9092 \
--topic migration-test

Consume

oc exec -it kafka-0 -- \
kafka-console-consumer \
--bootstrap-server localhost:9092 \
--topic migration-test \
--from-beginning

Message must flow without delay.

⸻

Validate Consumer Groups

oc exec -it kafka-0 -- \
kafka-consumer-groups \
--bootstrap-server localhost:9092 \
--describe \
--all-groups

No coordinator errors allowed.

⸻

Validate ACLs

oc exec -it kafka-0 -- \
kafka-acls \
--bootstrap-server localhost:9092 \
--list

ACL entries must match pre-migration state.

⸻

Monitor for Instability

oc logs kafka-0 | grep -i error

If observed:
	•	Continuous election retries
	•	Metadata load failures
	•	Broker crash loops

STOP and execute rollback before finalization.

Observe minimum 30–60 minutes.

⸻

7. Finalize to KRaft

Performed per region after validation passes.

kraftmigrationjob-finalize.yaml

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

Apply:

oc apply -f kraftmigrationjob-finalize.yaml
oc get kraftmigrationjob -o yaml -w

Wait until:

Phase: COMPLETE

Then move to next region.

⸻

8. Rollback Procedure

Only possible before COMPLETE.

oc apply -f kafka-backup.yaml
oc delete kraftmigrationjob kraftmigrationjob

After COMPLETE → rollback requires full cluster restore.

⸻

9. Post-Migration Cleanup

Performed per region.

Switch Context

oc config use-context central-cluster
oc cluster-info

Repeat for east and west.

⸻

Update Kafka CR – Remove ZooKeeper

kafka-kraft-only.yaml

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: kafka-prod
spec:
  replicas: 3
  kraftController:
    enabled: true
    name: kraftcontroller
  configOverrides:
    server:
      - "process.roles=broker,controller"
      - "controller.listener.names=CONTROLLER"
      - "inter.broker.listener.name=PLAINTEXT"

Apply:

oc apply -f kafka-kraft-only.yaml
oc get pods -w


⸻

Delete ZooKeeper

oc delete zookeeper zookeeper -n kafka-prod
oc get pods -n kafka-prod | grep zookeeper

Confirm no ZooKeeper pods remain.

⸻

Monitor Stability

oc logs kafka-0 | grep -i error

Brokers must remain stable.
No under-replicated partitions.
Controller quorum stable.

⸻

10. Critical MRC Rules

i. Never migrate multiple regions simultaneously.
ii. Always switch cluster context explicitly.
iii. Never finalize before validating dual-write.
iv. Never remove ZooKeeper before all regions complete.
v. Observe 24-hour stability window before ZooKeeper deletion.
vi. If instability occurs, stop immediately.
vii. Ensure identical CR changes across regions.
viii. Maintain consistent IBP across clusters.

⸻

Migrate from ZooKeeper to KRaft (migrating existing metadata)
https://docs.confluent.io/operator/current/co-migrate-kraft.html
https://docs.confluent.io/operator/current/co-multi-region.html#configure-active-passive-sr-in-mrc

Git Repo for KRaft MRC migration from ZooKeeper:
https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/migration/KRaftMigration/multi-region-cluster/kraft-based-cluster/confluent-platform/kraft
