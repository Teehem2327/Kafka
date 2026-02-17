Perfect 👍 — I’ll give you a clean, enterprise-style sandbox deployment document you can reuse for testing your ZooKeeper → KRaft migration safely.

Since you’re using:
	•	Confluent for Kubernetes (CFK)
	•	cp-server:7.8.0
	•	OpenShift (oc)
	•	Namespace: confluent
	•	Multi-region model (hub / spoke)
	•	Route-based external access

This will be aligned to your structure.

⸻

📘 Engineer Runbook

Deploy Kafka Sandbox on Kubernetes (OpenShift)

Purpose: Test ZooKeeper → KRaft Migration

⸻

1️⃣ Objective

Deploy an isolated Kafka sandbox cluster in Kubernetes/OpenShift to:
	•	Validate ZooKeeper-based deployment
	•	Test dual-write migration
	•	Validate finalize-to-KRaft
	•	Test rollback safely
	•	Verify MRC configuration logic

This sandbox must not affect production clusters.

⸻

2️⃣ Architecture Overview

Sandbox Components:
	•	1 Kafka cluster (3 brokers)
	•	1 ZooKeeper cluster (3 nodes)
	•	OpenShift route-based external access
	•	LDAP-enabled MDS
	•	CFK-managed resources

Namespace isolation:

confluent-sandbox

(Recommended to avoid conflict with prod)

⸻

3️⃣ Prerequisites

3.1 Platform Requirements
	•	OpenShift cluster access
	•	oc CLI configured
	•	Cluster admin or namespace admin permissions
	•	StorageClass available
	•	CFK operator installed

⸻

3.2 Verify CFK Operator

oc get pods -n confluent

You should see:

confluent-operator-xxxxx   Running


⸻

4️⃣ Create Sandbox Namespace

oc new-project confluent-sandbox

Verify:

oc project confluent-sandbox


⸻

5️⃣ Deploy ZooKeeper (Sandbox)

Create:

zookeeper-sandbox.yaml

apiVersion: platform.confluent.io/v1beta1
kind: Zookeeper
metadata:
  name: zookeeper
  namespace: confluent-sandbox

spec:
  replicas: 3
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-zookeeper:7.8.0
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.10.0

  dataVolumeCapacity: 50Gi
  logVolumeCapacity: 50Gi
  oneReplicaPerNode: true

  podTemplate:
    resources:
      limits:
        cpu: "1"
        memory: "4Gi"
      requests:
        cpu: "1"
        memory: "2Gi"

Apply:

oc apply -f zookeeper-sandbox.yaml

Wait:

oc get pods -w

All ZK pods must be Running.

⸻

6️⃣ Deploy Kafka (ZooKeeper Mode)

kafka-sandbox.yaml

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent-sandbox

spec:
  replicas: 3

  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.8.0
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.10.0

  dataVolumeCapacity: 50Gi
  oneReplicaPerNode: true

  dependencies:
    zookeeper:
      endpoint: zookeeper:2181

  podTemplate:
    resources:
      limits:
        cpu: "2"
        memory: "6Gi"
      requests:
        cpu: "1"
        memory: "4Gi"

  listeners:
    external:
      externalAccess:
        route:
          domain: sandbox.aepsc.com

Apply:

oc apply -f kafka-sandbox.yaml

Wait for brokers:

oc get pods -l platform.confluent.io/type=kafka -w


⸻

7️⃣ Validate Sandbox Health

Check Under Replication

oc exec -it kafka-0 -- \
kafka-topics --bootstrap-server localhost:9092 --describe | grep UnderReplicated

Must return:

UnderReplicatedPartitions: 0


⸻

Produce & Consume Test

Produce:

oc exec -it kafka-0 -- \
kafka-console-producer \
--bootstrap-server localhost:9092 \
--topic sandbox-test

Consume:

oc exec -it kafka-0 -- \
kafka-console-consumer \
--bootstrap-server localhost:9092 \
--topic sandbox-test \
--from-beginning

Messages must flow successfully.

⸻

8️⃣ Enable Migration Testing in Sandbox

Now apply your migration workflow:
	1.	Deploy kraftcontroller
	2.	Enable dual-write annotation
	3.	Validate DUAL_WRITE phase
	4.	Finalize
	5.	Remove ZooKeeper

Because this is sandbox:
	•	No production risk
	•	Can test rollback safely
	•	Can simulate instability scenarios

⸻

9️⃣ Rollback Testing (Sandbox Only)

Before finalize:

oc apply -f kafka-backup.yaml

Delete controller:

oc delete kafka kraftcontroller

This allows you to rehearse recovery steps safely.

⸻

🔟 Post-Test Cleanup

Delete sandbox environment completely:

oc delete project confluent-sandbox

Everything is removed cleanly.

⸻

1️⃣1️⃣ Why This Sandbox Design Is Correct
	•	Namespace isolation
	•	Independent ZK cluster
	•	Independent Kafka cluster
	•	Same CP version as prod
	•	Same image registry
	•	Same resource structure
	•	Route-based external access
	•	CFK-managed lifecycle

It mirrors production behavior without risk.

⸻

🔐 Recommended Testing Checklist

Before certifying migration logic:
	•	Test under load
	•	Test ACL validation
	•	Test consumer group stability
	•	Simulate broker restart during dual-write
	•	Simulate controller restart
	•	Confirm no metadata loss
	•	Confirm topic retention unaffected
	•	Confirm schema registry (if used) reconnects

⸻

🎯 Final Result

After following this:

You will have:
	•	A fully functional ZooKeeper-based Kafka cluster
	•	A safe environment to execute migration
	•	A controlled validation workflow
	•	A repeatable test procedure

⸻

If you want next, I can give you:
	•	A production-grade sandbox GitOps folder structure
	•	Or a visual migration phase diagram
	•	Or a checklist document formatted for submission

You’re building this the right way now.
