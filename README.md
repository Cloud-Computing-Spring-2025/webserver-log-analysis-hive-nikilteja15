# Web Server Log Analysis(with Apache Hive)

## Author
Sai Nikil Teja Swarna

## Project Overview
This project focuses on analyzing web server logs using Apache Hive. The objective is to process and extract meaningful insights from web server logs stored in CSV format. The analysis includes counting total requests, analyzing HTTP status codes, identifying most visited pages, detecting suspicious activity, and analyzing traffic trends.

## Implementation Approach
The analysis is implemented using Apache Hive, and the following tasks are performed:

1. **Setup Hive Environment:** Create a Hive database and an external table for web server logs.
2. **Load Data:** Load the CSV data into HDFS and import it into the Hive table.
3. **Execute Queries:** Perform various analytical queries to extract insights.
4. **Implement Partitioning:** Partition the data by status code to improve query performance.
5. **Generate Output:** Export query results for further analysis.

## Execution Steps

### **Step 1: Setup Hive Environment**
```sh
docker exec -it namenode /bin/bash  # Open an interactive bash shell inside the namenode container.
hdfs dfs -ls /  # List all directories in the root of HDFS.
hdfs dfs -mkdir -p /data/web_logs  # Create a directory in HDFS for storing web logs.
mkdir -p /data/web_logs  # Create a local directory for log files.
ls -l /data/web_logs  # List files and directories inside /data/web_logs.
exit  # Exit the container shell.
```

### **Step 2: Load Data into HDFS**
```sh
docker cp /workspaces/webserver-log-analysis-hive-nikilteja15/web_server_logs.csv namenode:/data/web_logs/web_server_logs.csv  # Copy CSV file to namenode container.

docker exec -it namenode /bin/bash  # Open a shell inside namenode container.
ls -l /data/web_logs/  # Verify the copied file in the container.
hdfs dfs -mkdir -p /data/web_logs  # Ensure HDFS directory exists.
hdfs dfs -put /data/web_logs/web_server_logs.csv /data/web_logs/  # Upload CSV to HDFS.
hdfs dfs -ls /data/web_logs/  # List files in HDFS to confirm upload.
exit  # Exit the namenode container.
```

### **Step 3: Create Hive Table and Load Data**
```sh
docker exec -it hive-server /bin/bash  # Open a shell inside the hive-server container.
hive  # Start the Hive CLI.
```
#### **Hive Queries:**
```sql
-- Create an external table for web logs
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs (
    ip STRING,
    timestamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/data/web_logs';

-- Load data into the table
LOAD DATA INPATH '/data/web_logs/web_server_logs.csv' INTO TABLE web_logs;
```

### **Step 4: Execute Queries**
```sql
-- Retrieve first 5 records
SELECT * FROM web_logs LIMIT 5;

-- Count total web requests
SELECT COUNT(*) AS total_requests FROM web_logs;

-- Analyze status codes
SELECT status, COUNT(*) AS count FROM web_logs GROUP BY status;

-- Identify the most visited pages
SELECT url, COUNT(*) AS visits FROM web_logs GROUP BY url ORDER BY visits DESC LIMIT 3;

-- Identify the most common user agents
SELECT user_agent, COUNT(*) AS count FROM web_logs GROUP BY user_agent ORDER BY count DESC;

-- Detect suspicious activity (IPs with more than 3 failed requests)
SELECT ip, COUNT(*) AS failed_requests FROM web_logs WHERE status IN (404, 500) GROUP BY ip HAVING COUNT(*) > 3;

-- Analyze traffic trends (requests per minute)
SELECT substr(timestamp, 0, 16) AS minute, COUNT(*) AS requests FROM web_logs GROUP BY substr(timestamp, 0, 16) ORDER BY minute;
```

### **Step 5: Automate Execution with HQL File**
```sh
touch hql_queries.hql  # Create an empty file to store Hive queries.
code hql_queries.hql  # Open the file in an editor.
```

#### **Save the above queries in `hql_queries.hql` and execute:**
```sh
docker cp hql_queries.hql hive-server:/opt/hql_queries.hql  # Copy the HQL file to the hive-server container.
docker exec -it hive-server ls -l /opt/hql_queries.hql  # Verify the file exists inside the container.
docker exec -it hive-server /bin/bash  # Open a shell inside hive-server.
hive -f /opt/hql_queries.hql | tee /opt/hql_output.txt  # Execute queries from the HQL file and save output.
exit  # Exit the hive-server container.
```

### **Step 6: Retrieve Output**
```sh
docker cp hive-server:/opt/hql_output.txt hql_output.txt  # Copy the query output to the local machine.
ls -l  # List files in the current directory to confirm output file exists.
```

## Challenges Faced
1. **Data Loading Issues:** Some fields contained unexpected NULL values, requiring manual data validation.
2. **Query Performance:** Large datasets required optimization via partitioning and indexing.
3. **HDFS Path Issues:** Ensured correct file locations before executing Hive queries.
4. **Docker Permissions:** Adjusted file permissions to allow data transfer between local machine and containers.

## Sample Input and Output
### **Sample Input (CSV Format)**
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

### **Sample Output**
#### **Total Web Requests**
```
Total Requests: 101
```
#### **Status Code Analysis**
```
200: 57
404: 21
500: 22
```
#### **Most Visited Pages**
```
/products: 24
/contact: 24
/home: 20
```
#### **Traffic Source Analysis**
```
Edge/88.0: 23
Chrome/90.0: 23
Opera/74.0: 21
Safari/13.1: 17
Mozilla/5.0: 16
```
#### **Suspicious IP Addresses**
```
192.168.1.1: 5 failed requests
192.168.1.15: 7 failed requests
192.168.1.2: 6 failed requests
192.168.1.20: 6 failed requests
192.168.1.25: 4 failed requests
192.168.1.3: 9 failed requests
```
#### **Traffic Trend Over Time**
```
2024-02-01 10:15: 7 requests
2024-02-01 10:16: 2 requests
2024-02-01 10:17	3
2024-02-01 10:18	2
2024-02-01 10:19	1
2024-02-01 10:20	4
2024-02-01 10:21	1
2024-02-01 10:22	2
2024-02-01 10:23	2
2024-02-01 10:24	1
2024-02-01 10:25	1
2024-02-01 10:26	6
2024-02-01 10:27	3
2024-02-01 10:29	1
2024-02-01 10:30	4
2024-02-01 10:31	3
2024-02-01 10:32	3
2024-02-01 10:34	2
2024-02-01 10:35	5
2024-02-01 10:36	1
2024-02-01 10:37	1
2024-02-01 10:38	7
2024-02-01 10:39	2
2024-02-01 10:40	3
2024-02-01 10:41	2
2024-02-01 10:42	1
2024-02-01 10:43	3
2024-02-01 10:46	1
2024-02-01 10:47	1
2024-02-01 10:49	1
2024-02-01 10:50	1
2024-02-01 10:51	4
2024-02-01 10:52	1
2024-02-01 10:53	5
2024-02-01 10:55	3
2024-02-01 10:56	3
2024-02-01 10:57	1
2024-02-01 10:58	3
2024-02-01 10:59	3
timestamp	1
```


