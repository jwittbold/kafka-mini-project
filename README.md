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
First we will create containerized versions of our Python generator and detector apps using the same Dockerfile for each. 
Our Dockerfile should look like this:
```
# Dockerfile
FROM python:3.8 

WORKDIR /usr/app

ADD ./requirements.txt ./
RUN pip install -r requirements.txt 
ADD ./ ./

CMD ["python", "app.py"]
```

Since we will be utilizing the kafka-python client, both the generator and detector folders will need to include our dependencies, ```requirements.txt```, containing ```kafka-python```

Our directory structure should appear as follows:  
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
