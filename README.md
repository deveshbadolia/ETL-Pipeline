

# ETL PIPELINE (POSTGRES + KAFKA + KAFKA UI + DEBEZIUM POSTGRES CONNECTOR)

Kafka Cluster: A Kafka cluster is a system that consists of several Brokers, Topics, and Partitions for both. The key objective is to distribute workloads equally among replicas and Partitions. We will provision 2 kafka nodes first node runs on port 9092 and second on 9093. Internal listener ports are 29092, 29093 and external are 9092, 9093 . This article explains more about kafka listeners.

Kafdrop: Kafdrop is a web UI for viewing Kafka topics and browsing consumer groups. The tool displays information such as brokers, topics, partitions, consumers, and lets you view messages. We will run kafdrop on default 9000 port (Also we use provectus ui image).

Schema Registry: Schema Registry provides a centralized repository for managing and validating schemas for topic message data, and for serialization and deserialization of the data over the network. Producers and consumers to Kafka topics can use schemas to ensure data consistency and compatibility as schemas evolve.

Postgres: We need a database for which we want to capture Data Change Events. We will use debezium/example-postgres:1.9 image to provision postgres database since it has got some default schema & tables created. Postgres docker runs on port 5433

Debezium Connector/Kafka Connect: The Debezium PostgreSQL connector captures row-level changes in the schemas of a PostgreSQL database. This container runs on port 8083.

docker-compose.yaml:
version: '3'

services:

  kafka1:
    image: confluentinc/cp-server:7.5.3
    container_name: kafka1
    hostname: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_LISTENERS: INTERNAL://kafka1:29092,CONTROLLER://kafka1:29093,EXTERNAL://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:29092,EXTERNAL://localhost:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      CLUSTER_ID: ciWo7IWazngRchmPES6q5A==
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
    networks:
       - custom-network

  kafka2:
    image: confluentinc/cp-server:7.5.3
    container_name: kafka2
    hostname: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_LISTENERS: INTERNAL://kafka2:29092,CONTROLLER://kafka2:29093,EXTERNAL://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka2:29092,EXTERNAL://localhost:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      CLUSTER_ID: ciWo7IWazngRchmPES6q5A==
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'

    networks:
      - custom-network
  kafka3:
    image: confluentinc/cp-server:7.5.3
    container_name: kafka3
    hostname: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_LISTENERS: INTERNAL://kafka3:29092,CONTROLLER://kafka3:29093,EXTERNAL://0.0.0.0:9094
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka3:29092,EXTERNAL://localhost:9094
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      CLUSTER_ID: ciWo7IWazngRchmPES6q5A==
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
    networks:
      - custom-network

  schema-registry:
    image: confluentinc/cp-schema-registry
    container_name: schema-registry
    hostname: schema-registry
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka1:29092,kafka2:29092,kafka3:29092
      SCHEMA_REGISTRY_LISTENERS: 'http://0.0.0.0:8081'
    depends_on:
      - kafka1
      - kafka2
      - kafka3


  kconnect:
    image: debezium/connect:2.5
    ports:
      - 8083:8083
    environment:
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
      BOOTSTRAP_SERVERS: kafka1:29092,kafka2:29092,kafka3:29092

    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - custom-network
  postgres:
    image: debezium/example-postgres
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    networks:
      - custom-network


  kafdrop:
    image: obsidiandynamics/kafdrop:3.28.0
    container_name: kafdrop

    restart: "no"
    environment:
       KAFKA_BROKERCONNECT: "kafka1:29092,kafka2:29093,kafka3:29093"
       JVM_OPTS: "-Xms16M -Xmx512M -Xss180K -XX:-TieredCompilation -XX:+UseStringDeduplication -noverify"
    ports:
      - 9000:9000
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - custom-network


networks:
  custom-network:
    driver: bridge



Register Kafka Connect
Once all the containers are up and running, open postman and make a POST call to http://localhost:8083/connectors with below request body.
Note: We are passing hostname as ‘postgres’ which is the container name in docker-compose file for postgres container and the port is internal port 5432 not the exposed one 5433.

{
    "name": "inventory-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "postgres",
        "database.server.name": "dbserver1",
        "table.include.list": "inventory.customers"
    }
}

Save above in register-postgres.json 
Then run this command 
Sudo curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json
Here we want to get all change events (INSERT, UPDATE, DELETE) for customers table in inventory schema. All the changes will be published to our kafka topic.

Visualize change events
Now that we have kafka connect successfully registered, we are good to make some changes in database.
We can use any database client eg. pgAdmin or DBeaver and connect to our database (host: localhost, post: 5433, user: postgres, password: postgres) and we can make an update/insert to existing row in customers table.

After the update statement is executed, we can see in kafdrop UI under topic: ‘dbserver1.inventor.customers’ a new message would be published.

UPDATE change event in Kafdrop UI
Similarly if we make an INSERT/DELETE statement, we will see new messages getting published to the same topic.

# Conclusion
We can write consumers for the above topic (dbserver1.inventor.customers) and trigger various actions(emails etc) based on change events (insert, update, delete).
This is how we get automated messages for various stages of our delivery items (Amazon, Flipkart, Delhivery)
