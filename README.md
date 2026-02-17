# kafka-rollback.yaml
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent

spec:
  image:
    application: docker.rtfx.aepsc.com/confluentinc/cp-server:7.8.0
    init: docker.rtfx.aepsc.com/confluentinc/confluent-init-container:2.10.0

  dataVolumeCapacity: 125Gi
  oneReplicaPerNode: true

  podTemplate:
    resources:
      limits:
        cpu: "2"
        memory: "8Gi"
      requests:
        cpu: "1"
        memory: "4Gi"

  configOverrides:
    log4j:
      - "log4j.logger.org.apache.kafka.metadata.migration=TRACE"

  listeners:
    external:
      externalAccess:
        route:
          domain: ehsbhub.aepsc.com

  authorization:
    superUsers:
      - User:kafka_adm_d
      - User:kafka_sadm_d

  services:
    mds:
      externalAccess:
        route:
          domain: ehsbhub.aepsc.com
      provider:
        ldap:
          address: "ldap://ds-q-global.aep.com:389"





# Apply the rollback
oc apply -f kafka-rollback.yaml

# Monitor the rollback
oc get pods -n confluent -w

# Verify it completed
oc get kafka -n confluent

		  
