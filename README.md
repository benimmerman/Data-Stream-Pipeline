# DE_Bootcamp_Final

# Architecture:

<img width="1411" alt="Screen Shot 2023-03-29 at 7 05 34 PM" src="https://user-images.githubusercontent.com/113261578/228687416-1c60c5b1-1cec-43d4-bb31-88b49327ad21.png">

# Project Summary:

This project builds a real-time application that collects data streamed from airplanes into an API called OpenSky REST API, and streams the data collected using Kafka and Spark Streaming to create live visual models that can be analyzed on Apache Superset.


# Running This Code
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
