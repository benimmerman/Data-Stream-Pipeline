# DE_Bootcamp_Final

# Architecture:

<img width="1411" alt="Screen Shot 2023-03-29 at 7 05 34 PM" src="https://user-images.githubusercontent.com/113261578/228687416-1c60c5b1-1cec-43d4-bb31-88b49327ad21.png">

# Project Summary:

This project builds a real-time application that collects data streamed from airplanes into an API called OpenSky REST API, and streams the data collected using Kafka and Spark Streaming to create live visual models that can be analyzed on Apache Superset.


# Running This Code

First we need to set up a MySQL database to store the streamed data. We will set this MySQL database up in an EC2 instance, and add an extra layer called debezium.
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

Now we will add processors to ingest the data.

--To set up the project, docker containers are used to create a MySQL database on an EC2 instance, as well as an MSK cluster which are both using the same VPC.

--Once the database and MSK cluser are created, we are ready to use the data that is constantly being updated as a JSON document from the API.

--Apache Nifi is set up on the EC2 instance using a docker container.

--The first processor in the NiFi flow chart is used to get the data directly from the API.

--When the data is received, the next processor in the flow chart transforms the data into a JSON dictionary that corresponds to a table that has been created in the MySQL dtabase.

--Next, NiFi puts the data from the transformed JSON document into the table created in the MySQL database.

--Every 30 seconds this process is repeated, streaming new data from the API and loading it into the MySQL table.

--During this process, an EMR cluster executes a Pyspark script (final_pyspark.py)

--The EMR cluster connectes to MSK which runs Kafka and Spark Streaming to transform the MySQL table into a Hudi table to be sent to an S3 bucket location.

--AWS Athena connects the streamed data from the S3 bucket into Apache Superset which is used to create a visualization of the location, speed, and altitude of all flights above an area specified in the API's url.
