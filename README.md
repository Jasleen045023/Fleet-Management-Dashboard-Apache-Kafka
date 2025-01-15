# Fleet Management Dashboard | Apache Kafka




<h1>About the Project</h1>

The project focused on creating and visualizing a fleet management dashboard by processing data from three Avro datasets. The aim was to provide key insights into vehicle performance, engine parameters, and driver activity using real-time streaming and visualization tools. The entire pipeline was developed and executed on Ubuntu, employing Kafka for data integration and Kibana for dashboard creation.

To accomplish this, three datasets were combined: Description, Location, and Sensors. Each dataset contributed unique and valuable data points that were integrated using Kafka’s streaming platform. The result was a comprehensive and interactive dashboard that provides meaningful metrics for fleet operations.

Key tasks included defining the schema for each dataset in Avro format, streaming and merging the datasets using Kafka topics, and designing interactive visualizations on Kibana. This approach ensured a seamless and scalable process for analyzing large volumes of real-time data. By utilizing Kafka and Kibana, the project demonstrated how streaming and visualization technologies can enhance fleet management efficiency.



<h1>Learning Objectives</h1>
<ol>
<li>Data Integration with Kafka: Acquired knowledge on how to use Kafka for integrating multiple datasets in Avro format, enabling real-time data processing and streaming.</li>
<li>Avro Schema Design: Learned to define and implement Avro schemas for datasets, ensuring accurate data serialization and compatibility.</li>
<li>Kibana Visualizations: Gained proficiency in creating dashboards using Kibana, with various charts like line graphs, bar graphs, and gauges to display insights.</li>
<li>Real-Time Analytics: Enhanced understanding of setting up real-time analytics pipelines, including data ingestion, processing, and visualization.</li>
<li>Ubuntu CLI Operations: Acquired hands-on experience managing and executing commands on Ubuntu for installing and running Kafka, Kibana, and related tools.</li>
</ol>




<h1>About the Fleet Management Dataset</h1>

The fleet management dashboard is designed to offer real-time insights into vehicle performance, driver activity, and engine health metrics. It combines data from three datasets:
<ol>
<li>Description Dataset: Contains details about vehicles, driver names, and license plates.</li>
<li>Location Dataset: Provides geolocation data of vehicles over time.</li>
<li>Sensors Dataset: Includes metrics such as engine temperature and average RPM.</li>
</ol>

Using these datasets, the dashboard offers insights into:
<ol>
<li>Average RPM: Displays the mean RPM of all vehicles, highlighting potential performance issues.</li>
<li>Maximum RPM: Identifies vehicles operating at peak RPM levels, useful for maintenance planning.</li>
<li>Engine Temperature Trends: Tracks temperature fluctuations over time to detect anomalies.</li>
<li>Driver Activity: Highlights distribution of data by driver name, providing insights into driver performance.</li>
<li>Vehicle Performance Monitoring: Monitors engine temperature performance trends to ensure vehicle health.</li>
</ol>




<h1>Dashboard</h1>


https://github.com/user-attachments/assets/53470cf3-f6a3-40a3-9c47-69f9a7b0d2e6



<h1>Dashboard Objectives</h1>
<ol>
<li>Monitor and analyze fleet performance in real-time.</li>
<li>Track vehicle locations and movements accurately.</li>
<li>Assess engine performance and detect anomalies using sensor data.</li>
<li>Integrate and visualize data across vehicles, drivers, and sensors for operational insights.</li>
<li>Streamline data pipeline for enhanced decision-making.</li>
</ol>



<h1>Pre-Requisites</h1>
<ol>
<li>Operating System: Ubuntu (or any compatible Linux distribution) for running Kafka and Kibana.</li>
<li>Required Tools: Kafka and Zookeeper for real-time data streaming. Kibana for creating visualizations. Java (JDK) and Avro tools for schema creation and data serialization.</li>
<li>System Requirements: Adequate disk space for datasets and streaming logs. Sufficient memory and CPU resources to run Kafka and Kibana smoothly.</li>
<li>Dataset Files: Avro files (Description, Location, Sensors) with properly defined schemas.</li>
<li>Network Configuration: Proper port configuration for Kafka, Kibana, and Elasticsearch.</li>
</ol>



<h1>Implementation Workflow</h1>
**## 1. Transforming Data to JSON Format**

- Script: Use the convert.py script to transform raw data into JSON key-value pairs.

- Command:
  python /home/ashok/Documents/fake/convert.py

- Input: Provide the full path of the JSON file (e.g., desc.json).

Feeding Data into Kafka

Start Docker: Ensure Docker is running.

Command: Use gen_sample.sh to pipe the JSON data into a Kafka topic (e.g., fm1).

./start_flink_nodatagen.sh
./gen_sample.sh /home/ashok/Documents/gendata/rev_desc.json | kafkacat -b localhost:9092 -t fm1 -K: -P

Validation: Open a new terminal and use the consumer script to check data flow:

./consumer.sh fm1

SQL Table Creation and Data Processing

Table Definitions

Description Table:

CREATE TABLE description (
    vehicle_id BIGINT,
    driver_name STRING,
    license_plate STRING,
    proctime AS PROCTIME() -- Only use processing time
) WITH (
    'connector' = 'kafka',
    'topic' = 'fm1',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);

Location Table:

CREATE TABLE location (
    vehicle_id BIGINT,
    latitude DOUBLE,
    longitude DOUBLE,
    ts BIGINT,
    proctime AS PROCTIME() -- Only use processing time
) WITH (
    'connector' = 'kafka',
    'topic' = 'fm2',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);

Sensor Table:

CREATE TABLE sensor (
    vehicle_id BIGINT,
    engine_temperature INT,
    average_rpm INT,
    proctime AS PROCTIME() -- Only use processing time
) WITH (
    'connector' = 'kafka',
    'topic' = 'fm3',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);

Merged View

CREATE VIEW merged_view_fleet AS
SELECT 
    l.vehicle_id,
    l.latitude,
    l.longitude,
    l.ts,
    d.driver_name,
    d.license_plate,
    s.engine_temperature,
    s.average_rpm
FROM 
    location l
JOIN 
    description d
ON 
    l.vehicle_id = d.vehicle_id
JOIN 
    sensor s
ON 
    l.vehicle_id = s.vehicle_id;

Export to Elasticsearch

Create Elasticsearch Table:
CREATE TABLE merged_view_fleet_es (
    vehicle_id BIGINT,
    latitude DOUBLE,
    longitude DOUBLE,
    ts BIGINT,
    driver_name STRING,
    license_plate STRING,
    engine_temperature INT,
    average_rpm INT,
    proctime TIMESTAMP(3)
) WITH (
    'connector' = 'elasticsearch-7', -- Elasticsearch connector
    'hosts' = 'http://elasticsearch:9200', -- Elasticsearch host
    'index' = 'merged_view_fleet_data', -- Elasticsearch index name
    'document-id.key-delimiter' = '-', -- Delimiter for document IDs
    'format' = 'json', -- Data format
    'sink.bulk-flush.max-actions' = '1' -- Immediate flush for testing (adjust for production)
);

Insert Data:
INSERT INTO merged_view_fleet_es
SELECT 
    vehicle_id,
    latitude,
    longitude,
    ts,
    COALESCE(driver_name, 'Unknown') AS driver_name, -- Replace null with "Unknown"
    COALESCE(license_plate, 'Unknown') AS license_plate, -- Replace null with "Unknown"
    COALESCE(engine_temperature, 0) AS engine_temperature, -- Replace null with 0
    COALESCE(average_rpm, 0) AS average_rpm, -- Replace null with 0
    CURRENT_TIMESTAMP AS proctime
FROM merged_view_fleet;










Data Generation and Transformation:
These commands generate synthetic data for the fleet (vehicles, locations, and sensors) and transform it into a usable format for Kafka topics. They simulate real-world streaming data.

```
./start_flink_nodatagen.sh  # Starts the Flink job without data generation
```

```
./gendata.sh description2.avro desc.json 10000  # Generates vehicle description data.
```

```
python /home/ashok/Documents/fake/convert.py desc.json  # Converts the generated data into the desired format.
```

```
./gen_sample.sh /home/ashok/Documents/gendata/rev_desc.json | kafkacat -b localhost:9092 -t fm1 -K: -P  # Streams data to the Kafka topic "fm1".
```

```
./consumer.sh fm1  # Consumes and validates the streamed data from Kafka topic "fm1".
```

```
./gendata.sh location2.avro loc.json 10000  # Generates vehicle location data.
```

```
python /home/ashok/Documents/fake/convert.py loc.json  # Converts the location data into the desired format.
```

```
./gen_sample.sh /home/ashok/Documents/gendata/rev_loc.json | kafkacat -b localhost:9092 -t fm2 -K: -P  # Streams data to the Kafka topic "fm2".
```

```
./consumer.sh fm2  # Consumes and validates the streamed data from Kafka topic "fm2".
```

```
./gendata.sh sensor2.avro sens.json 10000  # Generates sensor data for the fleet.
```

```
python /home/ashok/Documents/fake/convert.py sens.json  # Converts the sensor data into the desired format.
```

```
./gen_sample.sh /home/ashok/Documents/gendata/rev_sens.json | kafkacat -b localhost:9092 -t fm3 -K: -P  # Streams data to the Kafka topic "fm3".
```

```
./consumer.sh fm3  # Consumes and validates the streamed data from Kafka topic "fm3".
```

SQL Table Creation: 
Defines structured tables for each data stream (description, location, and sensor) in Kafka. These tables enable querying and joining data using SQL.
```
CREATE TABLE description (
     vehicle_id BIGINT,
     driver_name STRING,
     license_plate STRING,
     proctime AS PROCTIME() -- Only use processing time
 ) WITH (
     'connector' = 'kafka',
     'topic' = 'fm1',
     'scan.startup.mode' = 'earliest-offset',
     'properties.bootstrap.servers' = 'kafka:9094',
     'format' = 'json'
 );
```

```
SELECT * FROM description;
```

```
CREATE TABLE location (
     vehicle_id BIGINT,
     latitude DOUBLE,
     longitude DOUBLE,
     ts BIGINT,
     proctime AS PROCTIME() -- Only use processing time
 ) WITH (
     'connector' = 'kafka',
     'topic' = 'fm2',
     'scan.startup.mode' = 'earliest-offset',
     'properties.bootstrap.servers' = 'kafka:9094',
     'format' = 'json'
 );
```

```
SELECT * FROM location;
```

```
CREATE TABLE sensor (
     vehicle_id BIGINT,
     engine_temperature INT,
     average_rpm INT,
     proctime AS PROCTIME() -- Only use processing time
 ) WITH (
     'connector' = 'kafka',
     'topic' = 'fm3',
     'scan.startup.mode' = 'earliest-offset',
     'properties.bootstrap.servers' = 'kafka:9094',
     'format' = 'json'
 );
```

```
SELECT * FROM sensor;
```

Data Integration and Transformation:
Combines data from all three tables into a unified view for better insights and seamless analysis.
```
CREATE VIEW merged_view_fleet AS
 SELECT 
     l.vehicle_id,
     l.latitude,
     l.longitude,
     l.ts,
     d.driver_name,
     d.license_plate,
     s.engine_temperature,
     s.average_rpm
 FROM 
     location l
 JOIN 
     description d
 ON 
     l.vehicle_id = d.vehicle_id
 JOIN 
     sensor s
 ON 
     l.vehicle_id = s.vehicle_id;
```

```
SELECT * FROM merged_view_fleet;
```

Data Sink to Elasticsearch:
Exports the unified data view to Elasticsearch for advanced analytics and visualization in Kibana.
```
CREATE TABLE merged_view_fleet_es (
     vehicle_id BIGINT,
     latitude DOUBLE,
     longitude DOUBLE,
     ts BIGINT,
     driver_name STRING,
     license_plate STRING,
     engine_temperature INT,
     average_rpm INT,
     proctime TIMESTAMP(3)
 ) WITH (
     'connector' = 'elasticsearch-7', -- Elasticsearch connector
     'hosts' = 'http://elasticsearch:9200', -- Elasticsearch host
     'index' = 'merged_view_fleet_data', -- Elasticsearch index name
     'document-id.key-delimiter' = '-', -- Delimiter for document IDs
     'format' = 'json', -- Data format
     'sink.bulk-flush.max-actions' = '1' -- Immediate flush for testing (adjust for production)
 );
```

```
INSERT INTO merged_view_fleet_es
 SELECT 
     vehicle_id,
     latitude,
     longitude,
     ts,
     COALESCE(driver_name, 'Unknown') AS driver_name, -- Replace null with "Unknown"
     COALESCE(license_plate, 'Unknown') AS license_plate, -- Replace null with "Unknown"
     COALESCE(engine_temperature, 0) AS engine_temperature, -- Replace null with 0
     COALESCE(average_rpm, 0) AS average_rpm, -- Replace null with 0
     CURRENT_TIMESTAMP AS proctime
 FROM merged_view_fleet;
```

Dashboard Creation:
Utilize Kibana at localhost:5601 to create interactive and visual dashboards for monitoring fleet data in real-time.



<h1>Key Insights from the Dashboard</h1>
<ol>
<li>Vehicle Performance Monitoring: Average engine temperature: ~199.68°C.</li>
<li>Top 5 vehicles with maximum engine temperature identified for priority maintenance.</li>
<li>Driver Activity Analysis: Key drivers (e.g., Frieda Pirrone, Anatollo Beckwith) contributed significantly to fleet operations.</li>
<li>Engine Metrics: Average RPM is ~3,449 whereas Maximum RPM is 4,998, indicating potential overuse or high-performance operations.</li>
</ol>



<h1>Conclusion</h1>
The project successfully demonstrated how real-time streaming and visualization tools can be used to monitor and optimize fleet management. By integrating Avro datasets using Kafka and visualizing insights through Kibana, it provided a scalable solution for tracking vehicle performance and driver activity. This approach can be extended to other domains requiring real-time analytics and visualization.
