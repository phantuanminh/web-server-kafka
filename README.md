# Centralized Logging System for Docker Containers

## I. General Structure

```
            Mechanism        Mechanism            Mechanism
                A                B                    C
                |                |                    |
> Service 1 --------  Zookeeper  |                    |
                    \    &       |                    |
> Service 2 ---------> Kafka --------> ClickHouse -------> Superset
                    /(Rsyslog)
> Service n --------

```

Notes:

- Each service represents a Docker Container.
- Zookeeper is a MUST for Kafka. [Why?](https://stackoverflow.com/questions/23751708/is-zookeeper-a-must-for-kafka)
- To save space, we can try to run Zookeeper, Kafka, ClickHouse, Superset in a single container.

## II. General Mechanism Discussion

<b>1. Mechanism A:</b> Transmits all logging messages from multiple services (Docker Containers) to Kafka.

- First, we need to figure out how Docker handle its logging -> [Docker Logging Driver](https://docs.docker.com/config/containers/logging/configure/).
- I propose 2 possible solutions:
  - [Moby Kafka Logdriver](https://github.com/MickayG/moby-kafka-logdriver): A Docker plugin that works as a logdriver, transmit all log messages to Kafka.
  - Rsyslog: Transmit all log messages to a rsyslog server, then rsyslog decides where the messages go.

<b>2. Mechanism B:</b> Configure ClickHouse to automatically receive data from Kafka.

> Check out this really helpful tutorial: [Link](https://altinity.com/blog/2020/5/21/clickhouse-kafka-engine-tutorial)

![Structure](https://altinity.com/wp-content/uploads/2020/07/Screenshotfrom2020-05-1420-53-15.png)

<b>3. Mechanism C:</b> Configure Superset to visualize ClickHouse data

- Setup Superset User Interface. Use the interface to connect to ClickHouse URI -> Then it's all done!

## III. Demo

## IGNORE TEXT BELOW

## Run Kafka, ClickHouse and other services in the tutorial

```bash
$ sudo docker-compose up
```

## Create a Kafka topic

```bash
$ sudo docker exec -it kafka-broker kafka-topics \
--zookeeper 192.168.1.40:2181 \
--create \
--topic readings \
--partitions 6 \
--replication-factor 1
```

```bash
$ sudo docker exec -it kafka-broker kafka-topics \
--zookeeper 192.168.1.40:2181 \
--describe readings
```

## Configure ClickHouse to receive data from Kafka

![Structure](https://altinity.com/wp-content/uploads/2020/07/Screenshotfrom2020-05-1420-53-15.png)

### Open ClickHouse CLI

```bash
$ sudo docker exec -it clickhouse bin/bash -c "clickhouse-client --multiline"
```

1. Create a MergeTree Table

```bash
CREATE TABLE readings (
    readings_id Int32 Codec(DoubleDelta, LZ4),
    time DateTime Codec(DoubleDelta, LZ4),
    date ALIAS toDate(time),
    temperature Decimal(5,2) Codec(T64, LZ4)
) Engine = MergeTree
PARTITION BY toYYYYMM(time)
ORDER BY (readings_id, time);
```

2. Create Kafka Table Engine

```bash
CREATE TABLE readings_queue (
    readings_id Int32,
    time DateTime,
    temperature Decimal(5,2)
)
ENGINE = Kafka
SETTINGS kafka_broker_list = '192.168.1.40:9091',
       kafka_topic_list = 'readings',
       kafka_group_name = 'readings_consumer_group1',
       kafka_format = 'CSV',
       kafka_max_block_size = 1048576;
```

3. Create a materialized view to transfer data between Kafka and the merge tree table

```bash
CREATE MATERIALIZED VIEW readings_queue_mv TO readings AS
SELECT readings_id, time, temperature
FROM readings_queue;
```

4. Test the setup by producing some messages

```bash
sudo docker exec -it kafka-broker kafka-console-producer \
--broker-list 192.168.1.40:9091 \
--topic readings

# Data
1,"2020-05-16 23:55:44",14.2
2,"2020-05-16 23:55:45",20.1
3,"2020-05-16 23:55:51",12.9
```

## Configure ClickHouse on Superset

1. Register a root ClickHouse account

```bash
$ sudo docker exec -it superset superset-init
```

2. Connect to ClickHouse

- Use the UI at localhost:8080
- Add Clickhouse URI to Superset: clickhouse://clickhouse:8123

## Source

https://altinity.com/blog/2020/5/21/clickhouse-kafka-engine-tutorial
