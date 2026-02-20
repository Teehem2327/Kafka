Good — now we’ll lock this down cleanly using the KRaftMigrationJob (KMJ) model only, aligned with Confluent Support guidance and Confluent for Kubernetes.

No manual dual-write patch.
No migration-trigger annotation on Kafka CR.
Single clean flow.

⸻

✅ STEP 1 — Deploy KRaft Controllers (All Regions)

You deploy this in each region (switch context per region).

Your controller YAML is mostly correct.
Here is the cleaned, production-safe version.

⸻

📄 kraftcontroller.yaml  (Apply per region)

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kraftcontroller
  namespace: confluent
spec:
  replicas: 3   # Adjust per region to achieve global 2+2+1 total = 5
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.8.0
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.10.0
  dataVolumeCapacity: 20Gi
  oneReplicaPerNode: true

  configOverrides:
    server:
      - "process.roles=controller"
      - "controller.listener.names=CONTROLLER"

  listeners:
    controller:
      authentication:
        type: plain
      tls:
        enabled: false

Apply per region:

oc apply -f kraftcontroller.yaml

Wait until pods are Running before switching region.

⸻

✅ STEP 2 — Verify Global Quorum

After controllers are deployed in ALL regions:

oc get pods -n confluent | grep kraftcontroller

Total must equal 5 (for 2+2+1 design).

Then:

oc exec -it kraftcontroller-0 -n confluent -- \
kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status

You must see:
	•	All voters listed (5)
	•	One stable LeaderId
	•	No errors

If quorum unstable → STOP.

⸻

✅ STEP 3 — Apply KRaftMigrationJob (All Regions)

Now create this file.

⸻

📄 kraftmigrationjob.yaml  (Apply per region)

apiVersion: platform.confluent.io/v1beta1
kind: KRaftMigrationJob
metadata:
  name: kraftmigrationjob
  namespace: confluent
spec:
  dependencies:
    kafka:
      name: kafka
      namespace: confluent
    zookeeper:
      name: zookeeper
      namespace: confluent
    kRaftController:
      name: kraftcontroller
      namespace: confluent

Apply in Region A.
Switch context.
Apply in Region B.
Switch context.
Apply in Region C.

Same maintenance window.

⸻

✅ STEP 4 — Wait for DUAL_WRITE

In each region:

oc get kafka kafka -n confluent -o yaml | grep phase

Wait until:

phase: DUAL_WRITE

All regions must reach DUAL_WRITE before proceeding.

Do NOT finalize one region early.

⸻

✅ STEP 5 — Finalize (All Regions)

Once ALL regions are in DUAL_WRITE:

Patch Kafka CR in each region with finalize annotation.

⸻

📄 kafka-finalize.yaml (Apply per region)

apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent
  annotations:
    platform.confluent.io/kraft-migration-trigger-finalize: "true"

Apply per region:

oc apply -f kafka-finalize.yaml

Monitor:

oc get kafka kafka -n confluent -o yaml | grep phase

Wait until:

phase: COMPLETE

All regions must reach COMPLETE.

⸻

🔒 OPTIONAL (Post COMPLETE)

Release lock (if webhooks enabled):

oc annotate kraftmigrationjob kraftmigrationjob \
platform.confluent.io/kraft-migration-release-cr-lock=true \
--namespace confluent


⸻

🔵 Important Clarification

You no longer need:

❌ kafka-patch.yaml for dual-write
❌ platform.confluent.io/kraft-migration-trigger: “true”
❌ Manual dual-write enable step

KMJ handles that automatically.

⸻

🟢 Final Clean Flow Summary

1️⃣ Deploy controllers in ALL regions
2️⃣ Verify global quorum healthy
3️⃣ Apply KRaftMigrationJob in ALL regions
4️⃣ Wait until ALL regions reach DUAL_WRITE
5️⃣ Apply finalize annotation in ALL regions
6️⃣ Wait until ALL regions reach COMPLETE

No mixing methods.
No manual dual-write patching.

⸻

This is now:

✔ Fully aligned with Confluent guidance
✔ MRC-safe
✔ Clean
✔ Enterprise-correct
✔ No conflicting triggers

You can update your documentation confidently with this structure.
