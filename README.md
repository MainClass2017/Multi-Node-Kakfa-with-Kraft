# Multi-Node-Kakfa-with-Kraft

Step 1: Install Java 17
Kafka 8.x runs best on Java 17.

2)	Download tar
curl -O https://packages.confluent.io/archive/8.2/confluent-8.2.0.tar.gz


Bash
apt update
apt install -y openjdk-17-jdk
# Verify installation
java -version
Step 2: Create Directory Structure
Bash
mkdir -p /app/data/kafka-data/broker
mkdir -p /app/data/kafka-data/controller

mkdir -p /app/data/certs
mkdir -p /app/data/control-center

********************************************************************************************************************

sudo mkdir -p /app/data/certs
cd /app/data/certs

# Generate CA
sudo openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem -days 3650 -nodes \
-subj "/CN=KafkaCA/OU=Kafka/O=MyOrg/L=City/ST=State/C=IN"

# Generate Keystore
sudo keytool -genkeypair -noprompt \
-alias kafka-broker \
-keyalg RSA \
-keysize 2048 \
-validity 3650 \
-keystore /app/data/certs/pocv4kafka_server_ks.jks \
-storepass 'pocv4kafka#2025' \
-keypass 'pocv4kafka#2025' \
-dname "CN=kafka-cluster, OU=Kafka, O=MyOrg, L=City, ST=State, C=IN"

# Generate CSR
sudo keytool -certreq \
-alias kafka-broker \
-keystore /app/data/certs/pocv4kafka_server_ks.jks \
-storepass 'pocv4kafka#2025' \
-file /app/data/certs/broker.csr

# Create SAN extension for all 3 nodes
sudo tee /app/data/certs/san.ext > /dev/null <<EOF
subjectAltName=IP:192.168.8.187,DNS:d-as-db-cmn-kfka-8-187,IP:192.168.8.209,DNS:d-as-db-cmn-kfka-8-209,IP:192.168.8.240,DNS:d-he-db-cmn-kfka-8-240
EOF

# Sign CSR with CA
sudo openssl x509 -req \
-CA /app/data/certs/ca-cert.pem \
-CAkey /app/data/certs/ca-key.pem \
-in /app/data/certs/broker.csr \
-out /app/data/certs/broker-signed.crt \
-days 3650 \
-CAcreateserial \
-extfile /app/data/certs/san.ext

# Import CA cert into keystore
sudo keytool -import -noprompt \
-alias CARoot \
-keystore /app/data/certs/pocv4kafka_server_ks.jks \
-storepass 'pocv4kafka#2025' \
-file /app/data/certs/ca-cert.pem

# Import signed broker cert into keystore
sudo keytool -import -noprompt \
-alias kafka-broker \
-keystore /app/data/certs/pocv4kafka_server_ks.jks \
-storepass 'pocv4kafka#2025' \
-file /app/data/certs/broker-signed.crt

# Create Truststore
sudo keytool -import -noprompt \
-alias CARoot \
-keystore /app/data/certs/pocv4kafka_server_ts.jks \
-storepass 'pocv4kafka#2025' \
-file /app/data/certs/ca-cert.pem

# Verify
sudo openssl x509 -in /app/data/certs/broker-signed.crt -noout -text | grep -A2 "Subject Alternative"
openssl req -in broker.csr -noout -text


****************************************************************************************************************************************
JAAS Configuration (All Nodes)-

Create two separate JAAS files to allow the Broker and Controller to authenticate as the "admin" superuser.

Controller JAAS: vi /app/data/confluent-8.2.0/etc/kafka/jaas/controller_jaas.conf

KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};

Broker JAAS: vi /app/data/confluent-8.2.0/etc/kafka/jaas/broker_jaas.conf

KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};



****************************************************************************

Controller an broker put all properties  in three server 

****************************************************************************
Format storage all three server (controller and broker ) - 

/app/data/confluent-8.2.0/bin/kafka-storage format \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag \
  --config /app/data/confluent-8.2.0/etc/kafka/controller.properties \
  --add-scram 'SCRAM-SHA-256=[name=admin,password=admin@123]'

/app/data/confluent-8.2.0/bin/kafka-storage format \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag \
  --config /app/data/confluent-8.2.0/etc/kafka/broker.properties \
  --add-scram 'SCRAM-SHA-256=[name=admin,password=admin@123]'
  
******************************************************************************

cat > /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf <<EOF
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.endpoint.identification.algorithm=
EOF

************************************************************************************
after controller running but broker not running nee to reformat the the data and storage 
On all 3 controller nodes:

systemctl stop kafka-controller.service
rm -rf /app/data/kafka-data/controller/*

Then run on each controller:

/app/data/confluent-8.2.0/bin/kafka-storage format \
  --config /app/data/confluent-8.2.0/etc/kafka/controller.properties \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag \
  --add-scram 'SCRAM-SHA-256=[name=admin,password=admin@123]'
  
3) Start controllers first
After formatting all 3 controllers:
 
systemctl start kafka-controller.service

journalctl -u kafka-controller.service -n 100 --no-pager

4) Then start brokers

Once controllers are healthy:

systemctl stop kafka-broker.service
rm -rf /app/data/kafka-data/broker/*

/app/data/confluent-8.2.0/bin/kafka-storage format \
  --config /app/data/confluent-8.2.0/etc/kafka/broker.properties \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag
systemctl start kafka-broker.service

*****************************************************************************************************
Create User- 
root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-configs \
  --bootstrap-server 192.168.8.187:9093 \
  --alter \
  --add-config 'SCRAM-SHA-256=[password=samyak@123]' \
  --entity-type users \
  --entity-name samyak \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
Completed updating config for user vivek.


List User-

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-configs \
  --bootstrap-server 192.168.8.187:9093 \
  --describe \
  --entity-type users \
  --entity-name vivek \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
SCRAM credential configs for user-principal 'vivek' are SCRAM-SHA-256=iterations=4096


Create topics-

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-topics \
  --bootstrap-server 192.168.8.187:9093 \
  --create \
  --topic test-kafka \
  --partitions 3 \
  --replication-factor 3 \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
Created topic test-kafka.

ACL for vivek-topic-test read + write:

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-acls \
  --bootstrap-server 192.168.8.187:9093 \
  --add \
  --allow-principal User:confluent \
  --operation READ \
  --operation WRITE \
  --topic confluent \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=vivek-topic-test, patternType=LITERAL)`:
        (principal=User:vivek, host=*, operation=WRITE, permissionType=ALLOW)
        (principal=User:vivek, host=*, operation=READ, permissionType=ALLOW)

ACL for test-kafka all permissions:

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-acls \
  --bootstrap-server 192.168.8.187:9093 \
  --add \
  --allow-principal User:vivek \
  --operation ALL \
  --topic test-kafka \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test-kafka, patternType=LITERAL)`:
        (principal=User:vivek, host=*, operation=ALL, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test-kafka, patternType=LITERAL)`:
        (principal=User:vivek, host=*, operation=ALL, permissionType=ALLOW)
		
Consumer group permission:

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/bin# ./kafka-acls \
  --bootstrap-server 192.168.8.187:9093 \
  --add \
  --allow-principal User:vivek \
  --operation READ \
  --group '*' \
  --command-config /app/data/confluent-8.2.0/etc/kafka/admin-properties.conf
Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=*, patternType=LITERAL)`:
        (principal=User:vivek, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=*, patternType=LITERAL)`:
        (principal=User:vivek, host=*, operation=READ, permissionType=ALLOW)

Produce and Consume :

/app/data/confluent-8.2.0/bin# ./kafka-console-producer   --bootstrap-server 192.168.8.187:9093   --topic vivek-topic-test   --producer.config /app/data/confluent-8.2.0/etc/kafka/vivek.properties
  
/app/data/confluent-8.2.0/bin# ./kafka-console-consumer   --bootstrap-server 192.168.8.187:9093   --topic vivek-topic-test    --from-beginning   --consumer.config /app/data/confluent-8.2.0/etc/kafka/vivek.properties


************************************************************************************************************************************************


