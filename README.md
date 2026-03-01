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
