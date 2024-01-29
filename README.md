

# ETL PIPELINE (POSTGRES + KAFKA + KAFKA UI + DEBEZIUM POSTGRES CONNECTOR)


![image](https://github.com/deveshbadolia/ETL-Pipeline-/assets/124484199/3a71f4e6-b549-4a94-8585-47e98dc72f49)







Kafka Cluster: A Kafka cluster is a system that consists of several Brokers, Topics, and Partitions for both. The key objective is to distribute workloads equally among replicas and Partitions. We will provision 2 kafka nodes first node runs on port 9092 and second on 9093. Internal listener ports are 29092, 29093 and external are 9092, 9093 . This article explains more about kafka listeners.

Kafdrop: Kafdrop is a web UI for viewing Kafka topics and browsing consumer groups. The tool displays information such as brokers, topics, partitions, consumers, and lets you view messages. We will run kafdrop on default 9000 port (Also we use provectus ui image).

Schema Registry: Schema Registry provides a centralized repository for managing and validating schemas for topic message data, and for serialization and deserialization of the data over the network. Producers and consumers to Kafka topics can use schemas to ensure data consistency and compatibility as schemas evolve.

Postgres: We need a database for which we want to capture Data Change Events. We will use debezium/example-postgres:1.9 image to provision postgres database since it has got some default schema & tables created. Postgres docker runs on port 5433

Debezium Connector/Kafka Connect: The Debezium PostgreSQL connector captures row-level changes in the schemas of a PostgreSQL database. This container runs on port 8083.

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

# Output
![Screenshot (7)](https://github.com/deveshbadolia/ETL-Pipeline-/assets/124484199/34da1352-ff67-4155-aa20-c7dd174cf521)

![Screenshot (6)](https://github.com/deveshbadolia/ETL-Pipeline-/assets/124484199/cf7ff0c7-44c6-41ff-9d98-108cbc4cfe92)

![Screenshot (5)](https://github.com/deveshbadolia/ETL-Pipeline-/assets/124484199/f483ddc7-ccd7-4e53-87ad-13cdbfab41f1)

![Screenshot (4)](https://github.com/deveshbadolia/ETL-Pipeline-/assets/124484199/8ed65032-90c2-4861-acce-073947006ac9)




# Conclusion
We can write consumers for the above topic (dbserver1.inventor.customers) and trigger various actions(emails etc) based on change events (insert, update, delete).
This is how we get automated messages for various stages of our delivery items (Amazon, Flipkart, Delhivery)
