
# ELK Stack Setup in AWS EC2 with Apache NiFi Log Collection Using Filebeat

## Table of Contents
1. [Introduction](#introduction)
2. [What is ELK Stack?](#what-is-elk-stack)
   - 2.1. [Elasticsearch](#elasticsearch)
   - 2.2. [Logstash](#logstash)
   - 2.3. [Kibana](#kibana)
   - 2.4. [Filebeat](#filebeat)
3. [Purpose and Use Cases](#purpose-and-use-cases)
4. [Setup Guide](#setup-guide)
   - 4.1. [ELK Stack on Monitoring EC2](#elk-stack-on-monitoring-ec2)
   - 4.2. [Filebeat on Apache NiFi Nodes](#filebeat-on-apache-nifi-nodes)
5. [Access and Validation](#access-and-validation)
6. [Conclusion](#conclusion)

---

## Introduction

The ELK stack (Elasticsearch, Logstash, Kibana) is a robust and flexible platform for centralized logging and analytics. It is ideal for monitoring distributed systems, collecting logs from various sources, and visualizing them for insights. This guide explains how to set up the ELK stack on an AWS EC2 instance and configure Filebeat to collect logs from Apache NiFi clusters.

---

## What is ELK Stack?

### Elasticsearch
- **Definition:** A distributed search and analytics engine.
- **Purpose:** Indexes, stores, and retrieves log data for fast search and analysis.
- **Use Case:** Searching through logs from Apache NiFi systems for troubleshooting and monitoring.

### Logstash
- **Definition:** A server-side data processing pipeline.
- **Purpose:** Collects, processes, and forwards log data to Elasticsearch.
- **Use Case:** Parsing raw logs into structured data before storing them in Elasticsearch.

### Kibana
- **Definition:** A visualization and analytics platform for Elasticsearch.
- **Purpose:** Provides dashboards and tools for analyzing log data.
- **Use Case:** Visualizing Apache NiFi logs and system health metrics.

### Filebeat
- **Definition:** A lightweight log shipper.
- **Purpose:** Ships log files from servers to Logstash or Elasticsearch.
- **Use Case:** Forwarding logs from NiFi nodes to the centralized ELK stack.

---

## Purpose and Use Cases

### Purpose
- Centralized logging for distributed systems.
- Real-time troubleshooting and error resolution.
- Historical analysis of log trends.
- Compliance with auditing and regulatory requirements.

### Use Cases
1. **Monitoring NiFi Clusters:** Track and troubleshoot logs across multiple nodes.
2. **Real-Time Analytics:** Analyze application performance and detect issues proactively.
3. **Compliance:** Store logs securely for regulatory compliance.
4. **Visualization:** Create visual dashboards to interpret system metrics and trends.

---

## Setup Guide

### 4.1. ELK Stack on Monitoring EC2

#### Step 1: Prepare the Environment
```bash
mkdir elk
cd elk
touch docker-compose.yml
```

#### Step 2: Create Environment Variables
```bash
cat > .env <<EOL
ELK_VERSION=7.14.0

# Elasticsearch credentials
ELASTIC_USERNAME=elastic
ELASTIC_PASSWORD=j9ID7B7072eX

# Logstash memory settings
LS_JAVA_OPTS=-Xms1g -Xmx1g

# Kibana settings
SERVER_NAME=kibana
SERVER_PORT=5601
EOL
```

#### Step 3: Create the `docker-compose.yml` File
```yaml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${ELK_VERSION}
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ELASTIC_USERNAME=${ELASTIC_USERNAME}
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "7300:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - internal

  logstash:
    image: docker.elastic.co/logstash/logstash:${ELK_VERSION}
    container_name: logstash
    ports:
      - "6200:5044"
    volumes:
      - ./logstash-pipeline:/usr/share/logstash/pipeline
    environment:
      - xpack.security.enabled=true
      - ELASTICSEARCH_HOST=http://elasticsearch:9200
      - ELASTICSEARCH_USER=${ELASTIC_USERNAME}
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      - LS_JAVA_OPTS=${LS_JAVA_OPTS}
    networks:
      - internal

  kibana:
    image: docker.elastic.co/kibana/kibana:${ELK_VERSION}
    container_name: kibana
    ports:
      - "5100:${SERVER_PORT}"
    environment:
      - xpack.security.enabled=true
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=${ELASTIC_USERNAME}
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      - SERVER_NAME=${SERVER_NAME}
      - SERVER_PORT=${SERVER_PORT}
    networks:
      - internal

volumes:
  es-data:

networks:
  internal:
    driver: bridge
```

#### Step 4: Configure Logstash Pipeline
```bash
mkdir logstash-pipeline
vi logstash-pipeline/nifi.conf
```
Add the following:
```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [application] == "nifi" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:loglevel} %{GREEDYDATA:log_message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nifi-logs-%{+YYYY.MM.dd}"
  }
}
```

#### Step 5: Deploy ELK Stack
```bash
docker compose up -d
```

### 4.2. Filebeat on Apache NiFi Nodes

#### Step 1: Install Filebeat
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.14.0-x86_64.rpm
sudo rpm -vi filebeat-7.14.0-x86_64.rpm
```

#### Step 2: Configure Filebeat
Edit `/etc/filebeat/filebeat.yml`:
```yaml
output.elasticsearch:
  hosts: ["<Monitoring-EC2-IP>:7300"]
  username: "elastic"
  password: "j9ID7B7072eX"

setup.kibana:
  host: "<Monitoring-EC2-IP>:5100"

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /path/to/nifi/logs/*.log
    fields:
      application: nifi
    fields_under_root: true
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after
```

#### Step 3: Enable Modules
```bash
sudo filebeat modules list
sudo filebeat modules enable system
```

#### Step 4: Validate and Start Filebeat
```bash
sudo filebeat test config
sudo systemctl start filebeat
sudo journalctl -u filebeat -f
```

### Repeat the same steps in all Apache Nifi Nodes
---

## Access and Validation

- **Elasticsearch:** `http://<Monitoring-EC2-IP>:7300`
- **Kibana:** `http://<Monitoring-EC2-IP>:5100`

In Kibana:
1. Navigate to **Stack Management > Index Patterns** and create an index pattern (`nifi-logs-*`).
2. Use **Discover** to view logs in real time.

---

## Conclusion

By following this guide, you can deploy a scalable and efficient ELK stack in AWS and centralize logs from distributed systems like Apache NiFi. This setup provides a robust solution for monitoring, troubleshooting, and analyzing system logs.
