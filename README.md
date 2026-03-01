# Create a client properties file for INTERNAL listener
cat > /tmp/client.properties << EOF
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/mnt/sslcerts/aep-kafka-tls/truststore.p12
ssl.truststore.password=$(cat /mnt/sslcerts/aep-kafka-tls/jksPassword.txt)
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$(cat /mnt/secrets/in-kafka-plainldap/plain-jaas.conf | grep username | cut -d'"' -f2)" \
  password="$(cat /mnt/secrets/in-kafka-plainldap/plain-jaas.conf | grep password | cut -d'"' -f2)";
EOF

# Verify the file looks right
cat /tmp/client.properties



BOOTSTRAP="kafka-0.kafka.confluent.svc.cluster.local:9071"

# List topics
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --command-config /tmp/client.properties --list

# List consumer groups
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --command-config /tmp/client.properties --list

# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --command-config /tmp/client.properties \
  --describe --group <group-name>




# Replace YOUR_TRUSTSTORE_PASSWORD and credentials with real values from above
cat > /tmp/client.properties << 'EOF'
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/mnt/sslcerts/aep-kafka-tls/truststore.p12
ssl.truststore.type=PKCS12
ssl.truststore.password=YOUR_TRUSTSTORE_PASSWORD
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="YOUR_USERNAME" password="YOUR_PASSWORD";
EOF






Great question! Here's your complete incident command reference. Save this in VS Code as `kafka-incident-commands.sh`

---

## First — Always Set These Variables

```bash
# Run these first every time you enter the pod
BOOTSTRAP="kafka-0.kafka.confluent.svc.cluster.local:9071"
CONFIG="/tmp/client.properties"

# Recreate client.properties if it's gone (pod restarts clear /tmp)
cat > /tmp/client.properties << 'EOF'
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/mnt/sslcerts/aep-kafka-tls/truststore.p12
ssl.truststore.type=PKCS12
ssl.truststore.password=mystorepassword
ssl.endpoint.identification.algorithm=
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="kafka_adm_d" password="teK2k$xrTgbc%mm";
EOF
```

---

## 🚨 INCIDENT CATEGORY 1 — Cluster Health Check
*"Is the cluster even alive?"*

```bash
# Are all kafka pods running?
oc get pods | grep kafka

# Are all pods healthy? (should say Running, not CrashLoopBackOff)
oc get pods -o wide | grep kafka

# Check pod details if something looks wrong
oc describe pod kafka-0
oc describe pod kafka-1
oc describe pod kafka-2

# Check recent events — shows warnings and errors
oc get events --sort-by='.lastTimestamp' | grep -i kafka

# Check kafka operator is running
oc get pods | grep operator

# Check all nodes are healthy
oc get nodes

# See broker API versions (confirms broker is reachable)
/usr/bin/kafka-broker-api-versions \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG
```

---

## 🚨 INCIDENT CATEGORY 2 — Topic Issues
*"Messages not arriving? Topic problems?"*

```bash
# List all topics
/usr/bin/kafka-topics \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --list

# Describe a specific topic (partitions, replicas, leaders)
/usr/bin/kafka-topics \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --topic <topic-name>

# Check for under-replicated partitions (BAD if any show up)
/usr/bin/kafka-topics \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --under-replicated-partitions

# Check for unavailable partitions (VERY BAD — means data loss risk)
/usr/bin/kafka-topics \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --unavailable-partitions

# Check for partitions with no leader (CRITICAL)
/usr/bin/kafka-topics \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --topics-with-overrides
```

---

## 🚨 INCIDENT CATEGORY 3 — Consumer Group Lag
*"Why are consumers falling behind?"*

This is what your boss specifically mentioned. Lag = how many messages are waiting to be processed.

```bash
# List all consumer groups
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --list

# Check lag for ALL groups at once
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe --all-groups

# Check lag for a specific group
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --group <group-name>

# Check state of a consumer group
# States: Stable=good, Dead=bad, Empty=no consumers
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --group <group-name> \
  --state

# Find groups with high lag (incident alert)
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe --all-groups 2>/dev/null | \
  awk 'NR>1 && $6 > 1000'
  # Shows groups where lag column exceeds 1000
```

**Reading the lag output:**

```
GROUP          TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-consumer    my-topic     0          500             600             100  ← 100 messages behind
my-consumer    my-topic     1          600             600             0    ← all caught up
```

---

## 🚨 INCIDENT CATEGORY 4 — Consume Messages to Investigate
*"What messages are actually in this topic?"*

```bash
# Read latest 10 messages from a topic
/usr/bin/kafka-console-consumer \
  --bootstrap-server $BOOTSTRAP \
  --consumer.config $CONFIG \
  --topic <topic-name> \
  --max-messages 10

# Read from beginning (careful on large topics!)
/usr/bin/kafka-console-consumer \
  --bootstrap-server $BOOTSTRAP \
  --consumer.config $CONFIG \
  --topic <topic-name> \
  --from-beginning \
  --max-messages 10

# Read from specific partition
/usr/bin/kafka-console-consumer \
  --bootstrap-server $BOOTSTRAP \
  --consumer.config $CONFIG \
  --topic <topic-name> \
  --partition 0 \
  --offset earliest \
  --max-messages 5

# Read with timestamp shown (very useful for incidents)
/usr/bin/kafka-console-consumer \
  --bootstrap-server $BOOTSTRAP \
  --consumer.config $CONFIG \
  --topic <topic-name> \
  --property print.timestamp=true \
  --property print.key=true \
  --max-messages 10
```

---

## 🚨 INCIDENT CATEGORY 5 — Broker & Partition Health
*"Is a broker down? Partitions without leaders?"*

```bash
# Check broker configs (who is the controller?)
/usr/bin/kafka-configs \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --entity-type brokers \
  --all | grep -i leader

# List all broker IDs that are active
/usr/bin/kafka-broker-api-versions \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG 2>&1 | grep "id:"

# Check log dirs (disk usage per broker)
/usr/bin/kafka-log-dirs \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe | grep -i error

# Check pod logs for a specific broker
oc logs kafka-0 --tail=100
oc logs kafka-1 --tail=100

# Watch logs in real time during incident
oc logs kafka-0 -f

# Check logs for specific error
oc logs kafka-0 --tail=500 | grep -i "error\|warn\|exception"
```

---

## 🚨 INCIDENT CATEGORY 6 — Authentication & Authorization Issues
*"Someone can't connect? Access denied errors?"*

```bash
# Check if a user has ACL permissions
/usr/bin/kafka-acls \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --list

# Check ACLs for a specific topic
/usr/bin/kafka-acls \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --list \
  --topic <topic-name>

# Check RBAC role bindings (from outside pod)
oc get confluentrolebinding
oc describe confluentrolebinding <name>

# Check secrets are still valid and not expired
oc get secrets | grep -E "kafka|tls|oauth|mds"

# Check MDS (the OAuth badge office) is running
oc get pods | grep mds
curl -k https://kafka.confluent.svc.cluster.local:8090/security/1.0/authenticate
```

---

## 🚨 INCIDENT CATEGORY 7 — Resource & Performance Issues
*"Cluster is slow? Running out of disk?"*

```bash
# Check pod resource usage
oc top pods | grep kafka

# Check disk usage inside broker pod
df -h

# Check log directory sizes
du -sh /var/log/kafka/
du -sh /var/lib/kafka/data/

# Check if broker is under memory pressure
oc describe pod kafka-0 | grep -A 10 "Limits\|Requests\|OOMKilled"

# Check JVM metrics via JMX
curl -s http://localhost:7778/metrics | grep -i "UnderReplicated\|OfflinePartitions\|ActiveControllerCount"
```

---

## 🚨 INCIDENT CATEGORY 8 — Reset & Recovery
*"Consumer group stuck? Need to reset offsets?"*

```bash
# See where a consumer group currently is
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --describe \
  --group <group-name>

# DRY RUN reset to latest (see what would happen — safe)
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --group <group-name> \
  --topic <topic-name> \
  --reset-offsets \
  --to-latest \
  --dry-run

# ACTUAL reset — only do this if told to by your boss!
/usr/bin/kafka-consumer-groups \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG \
  --group <group-name> \
  --topic <topic-name> \
  --reset-offsets \
  --to-latest \
  --execute
```

---

## 🚨 INCIDENT CATEGORY 9 — Alerts Investigation
*"Alert fired — what do I check?"*

```bash
# Alert: UnderReplicatedPartitions
/usr/bin/kafka-topics --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG --describe --under-replicated-partitions

# Alert: ConsumerLag high
/usr/bin/kafka-consumer-groups --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG --describe --all-groups

# Alert: Broker down
oc get pods | grep kafka
oc logs kafka-<number> --tail=200 | grep -i error

# Alert: Disk full
oc exec -it kafka-0 -- df -h
oc exec -it kafka-0 -- du -sh /var/lib/kafka/data/*

# Alert: Authentication failure
oc logs kafka-0 --tail=200 | grep -i "authentication\|sasl\|ldap"

# Alert: No active controller
/usr/bin/kafka-broker-api-versions \
  --bootstrap-server $BOOTSTRAP \
  --command-config $CONFIG 2>&1 | grep -i controller
```

---

## 📋 Incident Response Checklist

When an alert fires, always follow this order:

```
1. oc get pods | grep kafka          ← Are all brokers up?
2. Check under-replicated partitions  ← Is data at risk?
3. Check consumer group lag           ← Are consumers stuck?
4. oc logs kafka-0 --tail=200         ← What does the broker say?
5. oc get events --sort-by=timestamp  ← What happened recently?
6. Escalate to team with findings
```

---

