# capture-data-change
Commands and steps to install kafka,debezium and zookeeper to capture mysql db changes

### Step 1: Create a README.md File
Below is a well-formatted version of your steps and commands, written in Markdown (GitHub’s preferred format). You can copy this into a file named `README.md`.

# Capture Data Change (CDC) Setup with MySQL, Debezium, Kafka, and Zookeeper

This repository documents the steps to set up Change Data Capture (CDC) using MySQL, Debezium, Kafka, and Zookeeper on a Linux system (tested on aarch64 architecture). 
Follow these steps to replicate MySQL data changes to Kafka topics.

---

## Step 1: Configure MySQL for CDC

### Log into MySQL
Connect to your MySQL instance (replace `192.168.64.6` with your MySQL server IP):
```bash
mysql -u root -p -h 192.168.64.6
```

### Create Debezium User
Create a user for Debezium and grant necessary privileges:
```sql
CREATE USER 'debezium'@'%' IDENTIFIED BY 'xxxx!';
ALTER USER 'debezium'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'xxxx!';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

### Enable Binary Logging
Check if binary logging is enabled:
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
```
Expected output: `log_bin=ON`, `binlog_format=ROW`.

If not enabled, edit the MySQL config file (e.g., `/etc/mysql/my.cnf` or `/etc/my.cnf`):
```os cmd
sudo vi /etc/mysql/my.cnf
```
Add under `[mysqld]`:
```
log_bin=ON
log-bin=mysql-bin
binlog_format=ROW
```
Restart MySQL:
```os cmd
systemctl restart mysqld
```
Verify again:
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
```

---

## Step 2: Install Java (JDK 8)
Kafka and Zookeeper require Java. Install JDK 8 for your OS architecture (example uses Linux aarch64).

### Download JDK
Download from Oracle (adjust for your architecture):
```
https://www.oracle.com/ng/java/technologies/downloads/#java8
```
Example file: `jdk-8u441-linux-aarch64.rpm`

### Install Java
Navigate to your download folder (e.g., `/home/sandbox/Downloads`):
```os cmd
cd /home/sandbox/Downloads
rpm -ivf jdk-8u441-linux-aarch64.rpm
```
Verify installation:
```os cmd
java -version
```
Expected output:
```
java version "1.8.0_441"
Java(TM) SE Runtime Environment (build 1.8.0_441-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.441-b07, mixed mode)
```

---

## Step 3: Install Zookeeper
Kafka includes Zookeeper for coordination.

### Download Zookeeper
```os cmd
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2.tar.gz
```

### Extract Zookeeper
```os cmd
cd /home/sandbox/Downloads
tar -xvf apache-zookeeper-3.6.2.tar.gz
```

### Move Zookeeper
```os cmd
mv apache-zookeeper-3.6.2 /home/zookeeper
```

---

## Step 4: Install Kafka

### Download Kafka
```os cmd
wget https://archive.apache.org/dist/kafka/2.6.3/kafka_2.12-2.6.3.tgz
```

### Extract Kafka
```os cmd
cd /home/sandbox/Downloads
tar -xvf kafka_2.12-2.6.3.tgz
```

### Move Kafka
```os cmd
mv kafka_2.12-2.6.3 /home/kafka
```

---

## Step 5: Set Up Debezium MySQL Connector

### Download Debezium
```os cmd
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.8.0.Final/debezium-connector-mysql-1.8.0.Final-plugin.tar.gz
```

### Extract Debezium
```os cmd
tar -xvzf debezium-connector-mysql-1.8.0.Final-plugin.tar.gz
```

### Move Debezium
```os cmd
mv debezium-connector-mysql-1.8.0.Final-plugin /home/kafka/plugins
```

---

## Step 6: Configure Debezium
Create a configuration file for the Debezium MySQL connector:
```os cmd
nano /home/kafka/config/connect-debezium-mysql.properties
```
Add (replace `xxxx!` with your password and adjust IPs):
```
name=mysql-connector-02
connector.class=io.debezium.connector.mysql.MySqlConnector
tasks.max=1
database.hostname=192.168.64.6
database.port=3306
database.user=debezium
database.password=xxxx!
database.server.id=223344
database.history.kafka.topic=msql.history
database.server.name=mysql-connector-02
database.include.list=classicmodels
database.history.kafka.bootstrap.servers=192.168.64.10:9092
database.jdbc.url=jdbc:mysql://192.168.64.6:3306/classicmodels?useSSL=false
```
Save and exit (`:wq`).

---

## Step 7: Start Zookeeper
Navigate to Kafka directory:
```os cmd
cd /home/kafka
```
Start Zookeeper:
```os cmd
./bin/zookeeper-server-start.sh config/zookeeper.properties &
```

### Fix JVM Option Error (if encountered)
If you see `Error: VM option 'UseG1GC' is experimental`:
1. Edit `kafka-run-class.sh`:
   ```os cmd
   vi /home/kafka/bin/kafka-run-class.sh
   ```
2. Modify `JVMFLAGS` (e.g., remove `-XX:+UseG1GC` or add `-XX:+UnlockExperimentalVMOptions` before it):
   ```
   JVMFLAGS="-Xmx512M -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC"
   ```
3. Save and restart Zookeeper.

---

## Step 8: Start Kafka Broker
```os cmd
./bin/kafka-server-start.sh config/server.properties &
```

---

## Step 9: Create Kafka Topic
Create a topic for Debezium history:
```os cmd
./bin/kafka-topics.sh --create --bootstrap-server 192.168.64.10:9092 --replication-factor 1 --partitions 1 --topic msql.history
```
Verify:
```os cmd
./bin/kafka-topics.sh --list --bootstrap-server 192.168.64.10:9092
```

---

## Step 10: Start Kafka Connect with Debezium
```os cmd
./bin/connect-standalone.sh config/connect-standalone.properties config/connect-debezium-mysql.properties
```

---

## Step 11: Verify Replication

### Check Topics
```os cmd
./bin/kafka-topics.sh --list --bootstrap-server 192.168.64.10:9092
```
Expect: `msql.history`, `mysql-connector-02.classicmodels.<table>`.

### Consume Table Data
Test a table (e.g., `mytab`):
```os cmd
./bin/kafka-console-consumer.sh --topic mysql-connector-02.classicmodels.mytab --bootstrap-server 192.168.64.10:9092 --from-beginning
```
Insert data in MySQL:
```sql
INSERT INTO testdb.mytab VALUES ('test');
```

### Check Schema Changes
```os cmd
./bin/kafka-console-consumer.sh --topic msql.history --bootstrap-server 192.168.64.10:9092 --from-beginning
```

---

## Troubleshooting

### Challenge 1: Public Key Retrieval Not Allowed
Error: `java.sql.SQLNonTransientConnectionException: Public Key Retrieval is not allowed`.
Fix:
```sql
ALTER USER 'debezium'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'xxxx!';
```

### Challenge 2: Unrecognized Time Zone 'WAT'
Error: `The server time zone value 'WAT' is unrecognized`.
Fix:
```sql
SET GLOBAL time_zone = 'Africa/Lagos';
FLUSH PRIVILEGES;
```

### Challenge 3: Kafka Host Mismatch
Error: No history topic due to wrong Kafka IP.
Fix: Update `database.history.kafka.bootstrap.servers` to `192.168.64.10:9092` in `connect-debezium-mysql.properties`.

---

## Next Steps
The next phase will involve setting up a secondary MySQL database to consume changes published to Kafka, completing the CDC pipeline.


