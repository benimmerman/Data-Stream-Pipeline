# DE_Bootcamp_Final

# Architecture:

<img width="1411" alt="Screen Shot 2023-03-29 at 7 05 34 PM" src="https://user-images.githubusercontent.com/113261578/228687416-1c60c5b1-1cec-43d4-bb31-88b49327ad21.png">

# Project Summary:

This project builds a real-time application that collects data streamed from airplanes into an API called OpenSky REST API, and streams the data collected using Kafka and Spark Streaming to create live visual models that can be analyzed on Apache Superset.

Link to the video presentation of this project: https://youtu.be/YLg45TL1GPA

# Running This Code

First we need to set up a MySQL database to store the streamed data. We will set this MySQL database up in an EC2 instance, and add an extra layer called debezium. Note: when setting up EC2, MSK, and EMR be sure that these services all use the same VPC.
In the terminal run the followng:

  docker run -dit --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.6
  docker exec -it '<container_id>' bash
  mysql -u root -p 
  debezium # root password for MySWQL
  CREATE DATABASE final;
  
As shown abovem, we will call this database final. Next we will create a table in this data called flights, and the script used to cread this table is found in create_flights.txt.

An example of the API url we will be using to gather the data is https://opensky-network.org/api/states/all?lamin=45.8389&lomin=5.9962&lamax=47.8229&lomax=10.5226, where the numerical values for lamin, lomin, lamax, and lomax represent the minimum and maximum boundary values for the latitudes and longitudes used to create a box surrounding the desired area of analysis. These values can be changed to correspond to whatever geographical area the user desires. For more information on the API go to https://openskynetwork.github.io/opensky-api/rest.html.
Now that the MyQSL database is set up and we have our API url, we can create our Apache Nifi Flow. We are going to set this up using a docker container running on an EC2 instance.

To run Nifi in your EC2 instance:

  docker run --name nifi -p 8080:8080 -p 8443:8443 --link mysql:mysql -d apache/nifi:1.12.0
  then visit http://:8080/nifi/

Now we will add processors to ingest the data. We will be using the processors InvokeHTTP, JoltTransformJSON, ConvertJSONtoSQL, and PutSQL in that order. For more information on the setup of this flow diagram reference the link to the video presentation of the project at around the 3:20 mark in the "Project Summary" section.
The first processor in the NiFi flow chart is used to get the data directly from the API. When the data is received, the next processor in the flow chart transforms the data into a JSON dictionary that corresponds to a table that has been created in the MySQL dtabase. Next, NiFi puts the data from the transformed JSON document into the table created in the MySQL database. Every 30 seconds this process is repeated, streaming new data from the API and loading it into the MySQL table. During this process, the EMR cluster you'll create executes the Pyspark script final_pyspark.py.

To use Kafka this project takes advantage of AWS's MSK service which fully manages Kafka. Again, remember to use the same VPC as your EC2 instance. This project's MSK cluster uses Kafka version 2.6.2, with 3 zones, and 200 GiB storage per broker. Once the MSK cluster is up and running, you can get the boostrap server endpoints by selecting "view client information" in "cluster summary". Copy the endpoints and paste them in the specified location in final_pyspark.py.

Now we need to install Kafka client to interact with the MSK cluster.
From the EC2 instance run:

sudo yum install java-1.8.0
wget https://archive.apache.org/dist/kafka/2.6.2/kafka_2.12-2.6.2.tgz
tar -xzf kafka_2.12-2.6.2.tgz

Go to the kafka_2.12-2.6.2/bin directory and create file client.properties and add the following line:

security.protocol=PLAINTEXT

Go to MKS cluster -> Client information to find the private endpoint and zookeeper connection. Copy it in to variables as below:

ZOOKEEPER_CONNECTION_STRING=z-3.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:2182,z-1.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:2182,z-2.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:2182
BOOTSTRAP_SERVERS=b-2.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:9092,b-3.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:9092,b-1.finalproject.34zas9.c3.kafka.ca-central-1.amazonaws.com:9092

Run the following command:

path-to-your-kafka-installation/bin/kafka-topics.sh --list --zookeeper 
$ZOOKEEPER_CONNECTION_STRING

You should see a few default topics, which means you were able successfully connect to MSK cluster.
Now we need to create a new Debezium connect container to establish connection between MySQL and MSK cluster. Run this also from your EC2 VM:

docker run -dit --name connect-msk -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my-connect-configs -e OFFSET_STORAGE_TOPIC=my-connect-offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses -e BOOTSTRAP_SERVERS=$BOOTSTRAP_SERVERS -e KAFKA_VERSION=2.6.2 -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=2 -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=2 -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=2 --link mysql:mysql debezium/connect:1.8.0.Final

Now if you start the ingestion pipeline in Nifi again and run kafka command to list all topics you should see new topics.
We can check what messages those topics are receiving:

cd kafka_2.12-2.6.2/bin/./kafka-console-consumer.sh  --bootstrap-server $BOOTSTRAP_SERVERS --topic my-connect-configs --from-beginning

You should see similar messages in my-connect-configs topic:

{"key":"FneNUqVfSeyCzUxKRdVqar9/Wsg7bt3jFJfV/T/FGz8=","algorithm":"HmacSHA256","creation-timestamp":1656810587172}
./kafka-console-consumer.sh --bootstrap-server BootstrapConnectString --topic my-connect-offsets --from-beginning
./kafka-console-consumer.sh --bootstrap-server BootstrapConnectString --topic my-connect-statuses --from-beginning

Other two topics should be empty since the connection with MySQL has not been established yet. Next, we need to establish the connection by running the following command:

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "final", "database.history.kafka.bootstrap.servers": "b-2.finalmskcluster.0lcx26.c13.kafka.us-east-1.amazonaws.com:9092,b-3.finalmskcluster.0lcx26.c13.kafka.us-east-1.amazonaws.com:9092,b-1.finalmskcluster.0lcx26.c13.kafka.us-east-1.amazonaws.com:9092", "database.history.kafka.topic": "dbhistory.final" } }'



--To set up the project, docker containers are used to create a MySQL database on an EC2 instance, as well as an MSK cluster which are both using the same VPC.

--Once the database and MSK cluser are created, we are ready to use the data that is constantly being updated as a JSON document from the API.

--Apache Nifi is set up on the EC2 instance using a docker container.

--

--

--

--

--During this process, an EMR cluster executes a Pyspark script (final_pyspark.py)

--The EMR cluster connectes to MSK which runs Kafka and Spark Streaming to transform the MySQL table into a Hudi table to be sent to an S3 bucket location.

--AWS Athena connects the streamed data from the S3 bucket into Apache Superset which is used to create a visualization of the location, speed, and altitude of all flights above an area specified in the API's url.
