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
