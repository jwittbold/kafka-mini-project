# Kafka + Python Streaming Fraud Detector in Docker

## Table of Contents
* [Project Info](#info)
* [Technologies](#technologies)
* [Set-up](#set-up)
* [Execution](#execution)


## Project Info
This project implements a real-time fraud detection system utilizing Kafka, Python, and Docker. It generates and processes a stream of transactions, identifying which transactions are fraudulent and which are legitimate while writing results out to respective fraudulent / legitimate Kafka topics. 

## Technologies
* Docker
* Python 3.8
* kafka-python client

## Set-up 

Our final directory structure should appear as follows:  
```
.
├── detector
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── docker-compose.kafka.yml
├── docker-compose.yml
└── generator
    ├── Dockerfile
    ├── app.py
    ├── requirements.txt
    └── transactions.py

2 directories, 9 files
```

First we will create containerized versions of our Python generator and detector apps using the same Dockerfile for each.  
Our identical Dockerfiles should read as follows:
```
# Dockerfile
FROM python:3.8 

WORKDIR /usr/app

ADD ./requirements.txt ./
RUN pip install -r requirements.txt 
ADD ./ ./

CMD ["python", "app.py"]
```

Since we will be utilizing the kafka-python client, both the generator and detector folders will need to include our dependencies, in the file 'requirements.txt' 

```
# requirements.txt
kafka-python
```
Because we would like to use Kafka as a service, running independently from the applications that are using it, we will need to isolate the Kafka cluster.
We accomplish this by creating two seperate docker-compose files, ```docker-compose.kafka.yml``` containing our Zookeeper and broker services, and ```docker-compose.yml``` which provides our application services.  

To allow both docker-compose compositions to access the same network, we must create an external Docker network. We can accomplish this by running:  
```docker network create kafka-network```

Our docker-compose files should read as follows:

```
# docker-compose.yml
version: "3"

services:
  generator:
    build: ./generator
    environment: 
      KAFKA_BROKER_URL: broker:9092
      TRANSACTIONS_TOPIC: queueing.transactions
      TRANSACTIONS_PER_SECOND: 10

  detector:
    build: ./detector
    environment: 
      KAFKA_BROKER_URL: broker:9092
      TRANSACTIONS_TOPIC: queueing.transactions
      LEGIT_TOPIC: streaming.transactions.legit
      FRAUD_TOPIC: streaming.transactions.fraud

networks: 
  default:
    name: kafka-network
```
AND
```
# docker-compose.kafka.yml
version: "3"

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      
  broker:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper 
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

networks:
  default:
    name: kafka-network
```






