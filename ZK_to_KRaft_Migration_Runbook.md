# Runbook: ZooKeeper to KRaft Migration
# Multi-Region Deployment (MRC)

**Confluent Platform:** 7.7.1  
**CFK Version:** 3.1.0+  
**Init Container:** 2.9.3  
**Platform:** OpenShift (oc CLI)  
**Regions:** NADC | GDC | NATOC  
**Last Updated:** March 2026  

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
   - [1.1 Platform Requirements](#11-platform-requirements)
   - [1.2 Environment Assumptions](#12-environment-assumptions)
2. [Architecture Overview](#2-architecture-overview)
3. [Deploy KRaft Controllers](#3-deploy-kraft-controllers-all-regions)
4. [Migration Execution Steps](#4-apply-kraftmigrationjob-all-regions)
5. [Validation During Dual-Write](#5-validation-after-dual-write)
6. [Finalize KRaft](#6-finalize-all-regions)
7. [Rollback Procedure](#7-rollback-procedure)
8. [Post-Migration Cleanup](#8-post-migration-cleanup)
9. [Critical MRC Rules](#9-critical-mrc-rules)
10. [Reference](#10-reference)

---

## 1. Prerequisites

### 1.1 Platform Requirements

- CFK version: 3.1.0+
- Confluent Platform: 7.7.1
- Init Container: 2.9.3
- Kubernetes/OpenShift supported version
- Each region is an independent cluster
- Sufficient CPU/memory headroom

#### Confirm Namespace per Region

```bash
# NADC
oc config use-context NADC
oc project confluent

# GDC
oc config use-context GDC
oc project confluentgdc

# NATOC
oc config use-context NATOC
oc project confluentnatoc
```

#### Confirm Resources Exist

```bash
oc get kafka
oc get zookeeper
oc get pods
```

#### Backup Current CRs

```bash
oc get kafka kafka -o yaml > kafka-backup.yaml
oc get zookeeper zookeeper -o yaml > zookeeper-backup.yaml
```

#### Validate Cluster Health

```bash
oc exec -it kafka-0 -- \
  kafka-topics \
  --bootstrap-server localhost:9092 \
  --describe | grep UnderReplicated
```

Must return:

```
UnderReplicatedPartitions: 0
```

Check controller stability:

```bash
oc logs kafka-0 | grep -i controller
```

#### Validate IBP Uniformity

```bash
oc exec -it kafka-0 -n confluent -- \
  kafka-configs --bootstrap-server localhost:9092 \
  --entity-type brokers --describe | grep inter.broker.protocol.version
```

> <<>> All brokers must have identical `inter.broker.protocol.version` across ALL regions before proceeding.

---

### 1.2 Environment Assumptions

| Parameter | NADC | GDC | NATOC |
|-----------|------|-----|-------|
| Namespace | `confluent` | `confluentgdc` | `confluentnatoc` |
| Kafka replicas | 4 | 4 | 4 |
| ZooKeeper replicas | 2 | 2 | 1 |
| broker-id-offset | 0 | 10 | 20 |
| zookeeper-myid-offset | 0 | 10 | 20 |
| KRaft controllers | 2 | 2 | 1 |
| dataVolumeCapacity (Kafka) | 1499Gi | 1499Gi | 1499Gi |
| dataVolumeCapacity (ZK) | 100Gi | 100Gi | 100Gi |
| logVolumeCapacity (ZK) | 50Gi | 50Gi | 50Gi |
| CPU limit / request (Kafka) | 10 / 8 | 10 / 8 | 10 / 8 |
| Memory limit / request (Kafka) | 16Gi / 12Gi | 16Gi / 12Gi | 16Gi / 12Gi |
| CPU limit / request (ZK) | 2 / 2 | 2 / 2 | 2 / 2 |
| Memory limit / request (ZK) | 8Gi / 6Gi | 8Gi / 6Gi | 8Gi / 6Gi |
| storageClass | thin-csi-cfk | thin-csi-cfk | thin-csi-cfk |
| client.rack | nadc | gdc | natoc |
| brokerPrefix | kafkanadc | kafkagdc | kafkanatoc |
| bootstrapPrefix | bootstrapnadc | bootstrapgdc | bootstrapnatoc |
| MDS host | mdsnadc.cpenterprise.aepsc.com | mdsgdc.cpenterprise.aepsc.com | mdsnatoc.cpenterprise.aepsc.com |
| MDS nodePortOffset | 30060 | 30070 | 30080 |
| ZK external host | zknadc.cpenterprise.aepsc.com | zkgdc.cpenterprise.aepsc.com | zknatoc.cpenterprise.aepsc.com |
| ZK nodePortOffset | 30010 | 30010 | 30010 |
| LDAP | ldaps://ds.global.aep.com:636 | ldaps://ds.global.aep.com:636 | ldaps://ds.global.aep.com:636 |
| RBAC superUsers | User:cp_kafkaadm | User:cp_kafkaadm | User:cp_kafkaadm |
| TLS secretRef | aep-kafka-tls | aep-kafka-tls | aep-kafka-tls |
| serviceAccountName | kafkamrc | kafkamrc | kafkamrc |

> <<>> UnderReplicatedPartitions must be 0 before migration  
> <<>> No active partition reassignment  
> <<>> No controller instability  

---

## 2. Architecture Overview

### 2.1 Current State

```
Kafka Broker  >>  ZooKeeper (Metadata authority)
```

### 2.2 Migration State (Dual Mode)

```
Kafka Brokers  >>  ZooKeeper + KRaft Controllers (Dual Metadata Write)
```

### 2.3 Final State

```
Kafka Brokers  >>  KRaft Controllers (Metadata Authority) — ZooKeeper removed
```

### 2.4 Multi-Region Layout

```
Region NADC    >>  OpenShift Cluster NADC    (namespace: confluent)       — 2 KRaft controllers
Region GDC     >>  OpenShift Cluster GDC     (namespace: confluentgdc)    — 2 KRaft controllers
Region NATOC   >>  OpenShift Cluster NATOC   (namespace: confluentnatoc)  — 1 KRaft controller
Total                                                                        5 controllers (odd — quorum safe)
```

> Confirmed by Confluent Technical Support (Jay Cruz):  
> "2+2+1 is good for MRC — odd numbered quorum spread across 3 regions.  
> A single region can go down without losing quorum."

---

## 3. Deploy KRaft Controllers (All Regions)

> Confirmed by Confluent Technical Support:  
> Deploy controllers in ALL regions and join into the same quorum BEFORE running any migration job.  
> Bootstrap quorum via NADC first, then GDC, then NATOC.

```bash
# NADC first (bootstrap quorum)
oc config use-context NADC
oc apply -f kraftcontroller-nadc.yaml
oc get pods -n confluent | grep kraftcontroller
# Wait for pods Running before proceeding

# GDC (join quorum)
oc config use-context GDC
oc apply -f kraftcontroller-gdc.yaml
oc get pods -n confluentgdc | grep kraftcontroller

# NATOC (join quorum)
oc config use-context NATOC
oc apply -f kraftcontroller-natoc.yaml
oc get pods -n confluentnatoc | grep kraftcontroller
```

> <<>> Pods must be Running in each region before switching to the next.

### kraftcontroller-nadc.yaml

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluent
  annotations:
    platform.confluent.io/kraft-migration-hold-krc-creation: "true"
    platform.confluent.io/use-log4j1: "true"
spec:
  replicas: 2
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.7.1
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.9.3
  dataVolumeCapacity: 20Gi
  oneReplicaPerNode: true
  authorization:
    type: rbac
    superUsers:
      - User:kraft
      - User:kafka
      - User:cp_kafkaadm
  dependencies:
    mdsKafkaCluster:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
      authentication:
        type: plain
        jaasConfig:
          secretRef: kraftcontroller-credential
      tls:
        enabled: true
        secretRef: aep-kafka-tls
  listeners:
    controller:
      authentication:
        type: plain
      tls:
        enabled: true
  tls:
    secretRef: aep-kafka-tls
  storageClass:
    name: thin-csi-cfk
  license:
    secretRef: confluent-license
```

### kraftcontroller-gdc.yaml

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluentgdc
  annotations:
    platform.confluent.io/kraft-migration-hold-krc-creation: "true"
    platform.confluent.io/use-log4j1: "true"
spec:
  replicas: 2
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.7.1
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.9.3
  dataVolumeCapacity: 20Gi
  oneReplicaPerNode: true
  authorization:
    type: rbac
    superUsers:
      - User:kraft
      - User:kafka
      - User:cp_kafkaadm
  dependencies:
    mdsKafkaCluster:
      bootstrapEndpoint: kafka.confluentgdc.svc.cluster.local:9071
      authentication:
        type: plain
        jaasConfig:
          secretRef: kraftcontroller-credential
      tls:
        enabled: true
        secretRef: aep-kafka-tls
  listeners:
    controller:
      authentication:
        type: plain
      tls:
        enabled: true
  tls:
    secretRef: aep-kafka-tls
  storageClass:
    name: thin-csi-cfk
  license:
    secretRef: confluent-license
```

### kraftcontroller-natoc.yaml

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluentnatoc
  annotations:
    platform.confluent.io/kraft-migration-hold-krc-creation: "true"
    platform.confluent.io/use-log4j1: "true"
spec:
  replicas: 1
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.7.1
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.9.3
  dataVolumeCapacity: 20Gi
  oneReplicaPerNode: true
  authorization:
    type: rbac
    superUsers:
      - User:kraft
      - User:kafka
      - User:cp_kafkaadm
  dependencies:
    mdsKafkaCluster:
      bootstrapEndpoint: kafka.confluentnatoc.svc.cluster.local:9071
      authentication:
        type: plain
        jaasConfig:
          secretRef: kraftcontroller-credential
      tls:
        enabled: true
        secretRef: aep-kafka-tls
  listeners:
    controller:
      authentication:
        type: plain
      tls:
        enabled: true
  tls:
    secretRef: aep-kafka-tls
  storageClass:
    name: thin-csi-cfk
  license:
    secretRef: confluent-license
```

### 3.1 Verify Global Quorum

After all 5 controller pods are Running across all regions:

```bash
oc get pods -n confluent | grep kraftcontroller
oc get pods -n confluentgdc | grep kraftcontroller
oc get pods -n confluentnatoc | grep kraftcontroller
```

Total must equal **5**.

Then verify quorum status:

```bash
oc exec -it kraftcontroller-0 -n confluent -- \
  kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status
```

Must show:
- All 5 voters listed
- One stable LeaderId
- No errors

> <<>> If quorum is unstable → STOP. Do NOT proceed to migration job.

---

## 4. Apply KRaftMigrationJob (All Regions)

> Confirmed by Confluent Technical Support:  
> Run migration in PARALLEL during the SAME maintenance window.  
> Use the SAME KMJ CR across all regions — namespace changes only.  
> Decoupling between regions has caused failures in the past.

First set the IBP annotation on the Kafka CR in each region.  
For CP 7.7.1 the IBP version is **3.7**:

```bash
oc annotate kafka kafka \
  platform.confluent.io/kraft-migration-ibp-version="3.7" \
  --namespace confluent

oc annotate kafka kafka \
  platform.confluent.io/kraft-migration-ibp-version="3.7" \
  --namespace confluentgdc

oc annotate kafka kafka \
  platform.confluent.io/kraft-migration-ibp-version="3.7" \
  --namespace confluentnatoc
```

Now apply the migration job in ALL regions simultaneously.  
Open a **separate terminal per region**:

### kraftmigrationjob.yaml (change namespace only per region)

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftMigrationJob
metadata:
  name: kraftmigrationjob
  namespace: confluent          # change to confluentgdc / confluentnatoc per region
spec:
  dependencies:
    kafka:
      name: kafka
      namespace: confluent      # change to confluentgdc / confluentnatoc per region
    zookeeper:
      name: zookeeper
      namespace: confluent      # change to confluentgdc / confluentnatoc per region
    kRaftController:
      name: kraftcontroller
      namespace: confluent      # change to confluentgdc / confluentnatoc per region
```

Apply — Terminal 1 (NADC):

```bash
oc config use-context NADC
oc apply -f kraftmigrationjob.yaml   # namespace: confluent
```

Apply — Terminal 2 (GDC):

```bash
oc config use-context GDC
oc apply -f kraftmigrationjob.yaml   # namespace: confluentgdc
```

Apply — Terminal 3 (NATOC):

```bash
oc config use-context NATOC
oc apply -f kraftmigrationjob.yaml   # namespace: confluentnatoc
```

### 4.0 Wait for DUAL_WRITE

```bash
oc get kafka kafka -n confluent -o yaml | grep phase
oc get kafka kafka -n confluentgdc -o yaml | grep phase
oc get kafka kafka -n confluentnatoc -o yaml | grep phase
```

Wait until all show:

```
phase: DUAL_WRITE
```

> <<>> All regions must reach DUAL_WRITE before proceeding.  
> <<>> Do NOT finalize one region early.

### 4.1 Verify CFK Migration CR Lock

When a KRaftMigrationJob is created, CFK automatically applies a migration lock to:
- Kafka CR
- ZooKeeper CR
- KRaftController CR

Check for lock (repeat per region):

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
oc get zookeeper zookeeper -n confluent -o yaml | grep kraft-migration-cr-lock
oc get kraftcontroller kraftcontroller -n confluent -o yaml | grep kraft-migration-cr-lock
```

If the lock is present the output will show:

```
platform.confluent.io/kraft-migration-cr-lock: "true"
```

#### Step 1 — Check If Webhooks Are Installed

```bash
oc get validatingwebhookconfigurations | grep confluent
oc get mutatingwebhookconfigurations | grep confluent
```

You should see: `confluent-operator-webhook`  
If nothing appears → webhook is not installed.

#### Step 2 — Check CFK Helm Values (If Installed via Helm)

```bash
helm get values confluent-operator -n confluent
```

Look for:

```yaml
webhook:
  enabled: true
```

#### Step 3 — Enable Webhook (If Disabled)

```bash
helm upgrade confluent-operator confluentinc/confluent-for-kubernetes \
  --namespace confluent \
  --set webhook.enabled=true
```

#### Step 4 — Confirm Lock Works

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
```

Expected: `platform.confluent.io/kraft-migration-cr-lock: "true"`

**Important — enabling webhook:**
- Does NOT restart brokers
- Does NOT affect running cluster
- Does NOT change data
- Only enforces CR validation — safe in sandbox and production

**Enterprise Recommendation:**

For Sandbox:
- Enable webhook
- Validate lock behavior
- Document it

For Production:
- Confirm webhook already enabled
- Do NOT migrate without it
- Verify lock annotation before DUAL_WRITE

---

## 5. Validation After Dual-Write

#### Produce

```bash
oc exec -it kafka-0 -n confluent -- \
  kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic migration-test
```

#### Consume

```bash
oc exec -it kafka-0 -n confluent -- \
  kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic migration-test \
  --from-beginning
```

#### Validate Consumer Groups

```bash
oc exec -it kafka-0 -n confluent -- \
  kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe \
  --all-groups
```

#### Validate ACLs

```bash
oc exec -it kafka-0 -n confluent -- \
  kafka-acls \
  --bootstrap-server localhost:9092 \
  --list
```

#### Confirm Metadata Fully Migrated

```bash
kubectl logs -f -c=kraftcontroller \
  --selector app=kraftcontroller -n confluent \
  | grep "Completed migration of metadata from ZooKeeper to KRaft"
```

> <<>> Observe 30-60 minutes of stability before finalizing.

No:
- Controller election storms
- Metadata load errors
- Broker crash loops

---

## 6. Finalize (All Regions)

Once ALL regions are in DUAL_WRITE and validation passes:

> <<>> Annotate the KRaftMigrationJob CR in each region with the finalize annotation.

### kafka-finalize.yaml (change namespace per region)

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftMigrationJob
metadata:
  name: kraftmigrationjob
  namespace: confluent          # change to confluentgdc / confluentnatoc per region
  annotations:
    platform.confluent.io/kraft-migration-trigger-finalize-to-kraft: "true"
```

Apply per region:

```bash
oc config use-context NADC
oc apply -f kafka-finalize.yaml   # namespace: confluent

oc config use-context GDC
oc apply -f kafka-finalize.yaml   # namespace: confluentgdc

oc config use-context NATOC
oc apply -f kafka-finalize.yaml   # namespace: confluentnatoc
```

Monitor:

```bash
oc get kraftmigrationjob kraftmigrationjob -n confluent -oyaml -w
oc get kraftmigrationjob kraftmigrationjob -n confluentgdc -oyaml -w
oc get kraftmigrationjob kraftmigrationjob -n confluentnatoc -oyaml -w
```

Wait until all regions show:

```
phase: COMPLETE
```

> <<>> All regions must reach COMPLETE before proceeding to cleanup.

---

## 7. Rollback Procedure

> <<>> Rollback is ONLY supported while migration phase is DUAL_WRITE.  
> <<>> Rollback is NOT supported once FINALIZE has been triggered or phase reaches COMPLETE.

### Step 1: Confirm migration phase

```bash
oc get kafka kafka -n confluent -o yaml | grep phase
```

Phase must show: `phase: DUAL_WRITE`  
If phase is `FINALIZING` or `COMPLETE` → STOP. Rollback not supported.

### Step 2: Apply rollback annotation to migration job

```bash
kubectl annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-trigger-rollback-to-zk=true \
  --overwrite \
  --namespace confluent
```

### Step 3: Remove ZooKeeper controller and migration nodes

```bash
zookeeper-shell <zkhost:zkport> deleteall <kafka-cr-name>-<namespace>/controller
zookeeper-shell <zkhost:zkport> deleteall <kafka-cr-name>-<namespace>/migration
```

### Step 4: Trigger node removal process

```bash
kubectl annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/continue-kraft-migration-post-zk-node-removal=true \
  --overwrite \
  --namespace confluent
```

### Step 5: Apply Kafka rollback CR

### kafka-rollback-gdc.yaml

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluentgdc
  annotations:
    platform.confluent.io/pod-overlay-configmap-name: aep-overlay-wait-for-zk
    platform.confluent.io/broker-id-offset: "10"
spec:
  replicas: 4
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.7.1
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.9.3
  dataVolumeCapacity: 1499Gi
  rackAssignment:
    nodeLabels:
      - confluentmrc
  podTemplate:
    podSecurityContext: {}
    serviceAccountName: kafkamrc
    resources:
      limits:
        cpu: "10"
        memory: "16Gi"
      requests:
        cpu: "8"
        memory: "12Gi"
    tolerations:
      - key: "kafkabroker"
        operator: "Equal"
        value: "reserved"
        effect: "NoSchedule"
      - key: "kafkabroker"
        operator: "Equal"
        value: "reserved"
        effect: "NoExecute"
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: kafka
  storageClass:
    name: thin-csi-cfk
  configOverrides:
    server:
      - auto.create.topics.enable=true
      - ldap.search.mode=GROUPS
      - client.rack=gdc
      - confluent.license.topic.replication.factor=4
      - replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
      - confluent.authorizer.init.timeout.ms=1200000
      - confluent.metadata.default.api.timeout.ms=1500000
      - min.insync.replicas=2
      - default.replication.factor=4
      - confluent.tier.metadata.replication.factor=4
      - offsets.topic.replication.factor=4
      - transaction.state.log.replication.factor=4
      - confluent.metrics.reporter.topic.replicas=4
      - confluent.metadata.topic.replication.factor=4
      - confluent.balancer.topic.replication.factor=4
      - confluent.security.event.logger.exporter.kafka.topic.replicas=4
      - event.logger.exporter.kafka.topic.replicas=4
    log4j:
      - "log4j.logger.org.apache.kafka.metadata.migration=TRACE"
  passwordEncoder:
    secretRef: password-encoder-secret
  tls:
    secretRef: aep-kafka-tls
  license:
    secretRef: confluent-license
  listeners:
    internal:
      authentication:
        type: ldap
        jaasConfig:
          secretRef: credential
      tls:
        enabled: true
    external:
      authentication:
        type: ldap
      externalAccess:
        type: route
        route:
          domain: cpenterprise.aepsc.com
          brokerPrefix: kafkagdc
          bootstrapPrefix: bootstrapgdc
      tls:
        enabled: true
    replication:
      authentication:
        type: ldap
        jaasConfigPassThrough:
          secretRef: repcred
      externalAccess:
        type: route
        route:
          domain: cpenterprise.aepsc.com
          brokerPrefix: kafkagdc-rep
          bootstrapPrefix: bootstrapgdc-rep
      tls:
        enabled: true
  metricReporter:
    enabled: true
    bootstrapEndpoint: kafka.confluentgdc.svc.cluster.local:9072,bootstrapnadc.cpenterprise.aepsc.com:443
    authentication:
      type: plain
      jaasConfig:
        secretRef: credential
    tls:
      enabled: true
  authorization:
    type: rbac
    superUsers:
      - User:cp_kafkaadm
  services:
    mds:
      tls:
        enabled: true
      tokenKeyPair:
        secretRef: mds-token
      externalAccess:
        type: nodePort
        nodePort:
          host: mdsgdc.cpenterprise.aepsc.com
          nodePortOffset: 30070
          advertisedURL:
            enabled: true
      provider:
        type: ldap
        ldap:
          address: ldaps://ds.global.aep.com:636
          tls:
            enabled: true
            secretRef: aep-kafka-tls
          authentication:
            type: simple
            simple:
              secretRef: credential
          configurations:
            userSearchScope: 2
            groupNameAttribute: cn
            groupObjectClass: groupOfUniqueNames
            groupMemberAttribute: uniqueMember
            groupMemberAttributePattern: uid=(.*),ou=(.*),ou=users,dc=Global,dc=aep,dc=com
            groupSearchScope: 2
            groupSearchFilter: (&(objectClass=groupOfUniqueNames)(|(cn=scm_eventhub-icoe_*)(cn=icoe-kafka-*)(cn=kafka-*)(cn=scm_ehsb*)(cn=Automatons)(cn=Employees)))
            groupSearchBase: ou=corp,ou=ActiveDirectory,ou=Groups,dc=Global,dc=aep,dc=com
            userNameAttribute: uid
            userSearchFilter: '"(objectclass=inetOrgPerson)"'
            userMemberOfAttributePattern: uid=(.*),ou=Users,dc=Global,dc=aep,dc=com
            userObjectClass: inetOrgPerson
            userSearchBase: ou=Users,dc=Global,dc=aep,dc=com
  dependencies:
    kafkaRest:
      authentication:
        type: bearer
        bearer:
          secretRef: mds-client
      tls:
        enabled: true
        secretRef: aep-kafka-tls
    zookeeper:
      endpoint: zookeeper.confluentgdc.svc.cluster.local:2182,zknatoc.cpenterprise.aepsc.com:30001,zknadc.cpenterprise.aepsc.com:30020,zknadc.cpenterprise.aepsc.com:30023/mrc
      authentication:
        type: digest
        jaasConfig:
          secretRef: credential
      tls:
        enabled: true
```

Apply:

```bash
oc apply -f kafka-rollback-gdc.yaml
# Repeat with equivalent NADC and NATOC versions
```

### Step 6: Delete KRaftMigrationJob

```bash
oc delete kraftmigrationjob kraftmigrationjob -n confluent
oc delete kraftmigrationjob kraftmigrationjob -n confluentgdc
oc delete kraftmigrationjob kraftmigrationjob -n confluentnatoc
```

### Step 7: Monitor Kafka pods

```bash
oc get pods -n confluent -w
oc get pods -n confluentgdc -w
oc get pods -n confluentnatoc -w
```

### Step 8: Validate ZooKeeper Authority

Confirm:
- ZooKeeper pods Running
- Kafka brokers connected to ZooKeeper
- No metadata migration logs present
- Cluster stable

**Important Notes:**
- If migration phase reaches COMPLETE, rollback requires full cluster restore from backup
- Finalize annotation removal does NOT revert migration once FINALIZING has begun
- Do not attempt manual rollback during FINALIZING phase

---

## 8. Post-Migration Cleanup

### 8.1 Switch to Each Region Context

Cleanup must be performed per region.

```bash
# NADC
oc config use-context NADC
oc cluster-info

# GDC
oc config use-context GDC
oc cluster-info

# NATOC
oc config use-context NATOC
oc cluster-info
```

> <<>> Confirm you are in the correct cluster before applying any changes.

After ALL regions are COMPLETE, remove ZooKeeper reference from Kafka CR:

### kafka-patch-gdc.yaml

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluentgdc
  annotations:
    platform.confluent.io/broker-id-offset: "10"
spec:
  replicas: 4
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.7.1
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.9.3
  dataVolumeCapacity: 1499Gi
  rackAssignment:
    nodeLabels:
      - confluentmrc
  podTemplate:
    podSecurityContext: {}
    serviceAccountName: kafkamrc
    resources:
      limits:
        cpu: "10"
        memory: "16Gi"
      requests:
        cpu: "8"
        memory: "12Gi"
    tolerations:
      - key: "kafkabroker"
        operator: "Equal"
        value: "reserved"
        effect: "NoSchedule"
      - key: "kafkabroker"
        operator: "Equal"
        value: "reserved"
        effect: "NoExecute"
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: kafka
  storageClass:
    name: thin-csi-cfk
  kraftController:
    name: kraftcontroller
  configOverrides:
    server:
      - auto.create.topics.enable=true
      - ldap.search.mode=GROUPS
      - client.rack=gdc
      - confluent.license.topic.replication.factor=4
      - replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
      - confluent.authorizer.init.timeout.ms=1200000
      - confluent.metadata.default.api.timeout.ms=1500000
      - min.insync.replicas=2
      - default.replication.factor=4
      - confluent.tier.metadata.replication.factor=4
      - offsets.topic.replication.factor=4
      - transaction.state.log.replication.factor=4
      - confluent.metrics.reporter.topic.replicas=4
      - confluent.metadata.topic.replication.factor=4
      - confluent.balancer.topic.replication.factor=4
      - confluent.security.event.logger.exporter.kafka.topic.replicas=4
      - event.logger.exporter.kafka.topic.replicas=4
  passwordEncoder:
    secretRef: password-encoder-secret
  tls:
    secretRef: aep-kafka-tls
  license:
    secretRef: confluent-license
  listeners:
    internal:
      authentication:
        type: ldap
        jaasConfig:
          secretRef: credential
      tls:
        enabled: true
    external:
      authentication:
        type: ldap
      externalAccess:
        type: route
        route:
          domain: cpenterprise.aepsc.com
          brokerPrefix: kafkagdc
          bootstrapPrefix: bootstrapgdc
      tls:
        enabled: true
    replication:
      authentication:
        type: ldap
        jaasConfigPassThrough:
          secretRef: repcred
      externalAccess:
        type: route
        route:
          domain: cpenterprise.aepsc.com
          brokerPrefix: kafkagdc-rep
          bootstrapPrefix: bootstrapgdc-rep
      tls:
        enabled: true
  metricReporter:
    enabled: true
    bootstrapEndpoint: kafka.confluentgdc.svc.cluster.local:9072,bootstrapnadc.cpenterprise.aepsc.com:443
    authentication:
      type: plain
      jaasConfig:
        secretRef: credential
    tls:
      enabled: true
  authorization:
    type: rbac
    superUsers:
      - User:cp_kafkaadm
  services:
    mds:
      tls:
        enabled: true
      tokenKeyPair:
        secretRef: mds-token
      externalAccess:
        type: nodePort
        nodePort:
          host: mdsgdc.cpenterprise.aepsc.com
          nodePortOffset: 30070
          advertisedURL:
            enabled: true
      provider:
        type: ldap
        ldap:
          address: ldaps://ds.global.aep.com:636
          tls:
            enabled: true
            secretRef: aep-kafka-tls
          authentication:
            type: simple
            simple:
              secretRef: credential
          configurations:
            userSearchScope: 2
            groupNameAttribute: cn
            groupObjectClass: groupOfUniqueNames
            groupMemberAttribute: uniqueMember
            groupMemberAttributePattern: uid=(.*),ou=(.*),ou=users,dc=Global,dc=aep,dc=com
            groupSearchScope: 2
            groupSearchFilter: (&(objectClass=groupOfUniqueNames)(|(cn=scm_eventhub-icoe_*)(cn=icoe-kafka-*)(cn=kafka-*)(cn=scm_ehsb*)(cn=Automatons)(cn=Employees)))
            groupSearchBase: ou=corp,ou=ActiveDirectory,ou=Groups,dc=Global,dc=aep,dc=com
            userNameAttribute: uid
            userSearchFilter: '"(objectclass=inetOrgPerson)"'
            userMemberOfAttributePattern: uid=(.*),ou=Users,dc=Global,dc=aep,dc=com
            userObjectClass: inetOrgPerson
            userSearchBase: ou=Users,dc=Global,dc=aep,dc=com
  dependencies:
    kafkaRest:
      authentication:
        type: bearer
        bearer:
          secretRef: mds-client
      tls:
        enabled: true
        secretRef: aep-kafka-tls
```

> <<>> Ensure `spec.kraftController.name: kraftcontroller` is present.  
> <<>> Ensure NO ZooKeeper dependency remains in the CR.

Apply per region:

```bash
oc apply -f kafka-patch-gdc.yaml
# Repeat with equivalent NADC and NATOC versions
```

#### Delete ZooKeeper CR

```bash
oc delete zookeeper zookeeper -n confluent
oc delete zookeeper zookeeper -n confluentgdc
oc delete zookeeper zookeeper -n confluentnatoc

oc get pods -n confluent | grep zookeeper
oc get pods -n confluentgdc | grep zookeeper
oc get pods -n confluentnatoc | grep zookeeper
```

> <<>> No ZooKeeper pods must remain.

### 8.2 Remove CFK Migration Lock

After migration phase reaches COMPLETE the migration lock must be manually released.

Verify lock is present:

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
oc get kafka kafka -n confluentgdc -o yaml | grep kraft-migration-cr-lock
oc get kafka kafka -n confluentnatoc -o yaml | grep kraft-migration-cr-lock
```

Release lock per region:

```bash
oc annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-release-cr-lock=true \
  --namespace confluent

oc annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-release-cr-lock=true \
  --namespace confluentgdc

oc annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-release-cr-lock=true \
  --namespace confluentnatoc
```

Verify lock removed — should return empty in all regions:

```bash
oc get kafka kafka -n confluent -o yaml | grep kraft-migration-cr-lock
oc get kafka kafka -n confluentgdc -o yaml | grep kraft-migration-cr-lock
oc get kafka kafka -n confluentnatoc -o yaml | grep kraft-migration-cr-lock
```

The annotation should no longer exist in:
- Kafka CR
- ZooKeeper CR
- KRaftController CR

**Important — failure to remove the migration lock may:**
- Prevent future configuration updates
- Block version upgrades
- Interfere with GitOps or CI/CD pipelines (ArgoCD)

---

## 9. Critical MRC Rules

Confirmed and validated by Confluent Technical Support (Jay Cruz):

- **Deploy KRaft controllers in ALL regions first** and join into the same quorum before any migration job is run
- **Bootstrap quorum via NADC first**, add GDC and NATOC, THEN proceed
- **Verify global quorum: 5 voters, 1 stable leader** — no errors before migrating
- **Run KRaftMigrationJob in ALL regions IN PARALLEL** during the SAME maintenance window
- **Use the SAME KMJ CR across all regions** — namespace changes only. Decoupling regions causes failures
- **Enable dual-write in ALL regions** before any validation or finalization
- **Validate ALL regions during DUAL_WRITE** — observe 30-60 minutes of stability
- **Finalize ALL regions** — never leave one region on ZooKeeper while others are on KRaft
- **Never delete ZooKeeper until ALL regions have reached COMPLETE**
- **Never mix Confluent Platform versions** across regions during migration
- **Maintain identical IBP (3.7)** across all clusters throughout migration
- **Remove the migration lock after COMPLETE** — required for future upgrades and CI/CD

---

## 10. Reference

**Official Confluent Documentation:**

- Migrate ZooKeeper to KRaft (CFK):
  https://docs.confluent.io/operator/current/co-migrate-kraft.html

- Multi-Region Cluster configuration:
  https://docs.confluent.io/operator/current/co-multi-region.html

**CFK Example GitHub Repository (KRaft MRC Migration):**
  https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/migration/KRaftMigration/multi-region-cluster

**Confluent Technical Support Input:**  
  Validated by Jay Cruz — Confluent Technical Support
  - 2+2+1 controller quorum confirmed for MRC
  - All controllers must join single quorum before migration jobs
  - Migration jobs must run in parallel in the same maintenance window
  - Same KMJ CR must be used across all regions (namespace changes only)
