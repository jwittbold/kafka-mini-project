# Kafka + Python Streaming Fraud Detector in Docker

## Table of Contents
* [Project Info](#info)
* [Technologies](#technologies)
* [Set-up](#set-up)
* [Execution](#execution)


## Project Info
This project implements a real-time fraud detection system utilizing Kafka, Python, and Docker. It generates and processes a stream of transactions, identifying which transactions are fraudulent and which are legitimate, while writing results out to respective fraudulent / legitimate Kafka topics. 


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


### Dockerfiles
First we will create containerized versions of our Python generator and detector apps using the same Dockerfile for each. 
Our identical Dockerfiles should read as follows:
```Dockerfile
# Dockerfile
FROM python:3.8 

WORKDIR /usr/app

ADD ./requirements.txt ./
RUN pip install -r requirements.txt 
ADD ./ ./

CMD ["python", "app.py"]
```


### Dependencies
Since we will be utilizing the kafka-python client, both the generator and detector folders will need to include our (identical) dependencies, in the file 'requirements.txt' 

```
# requirements.txt
kafka-python
```


### docker-compose files (Isolating the Kafka cluster)
Because we would like to use Kafka as a service, running independently from the applications that are using it, we will need to isolate the Kafka cluster.
This is accomplished by by creating two seperate docker-compose files, ```docker-compose.kafka.yml``` containing our Zookeeper and broker services, and ```docker-compose.yml``` which provides our application services.  

Our docker-compose files should read as follows:

```yml
# docker-compose.yml
version: "3"

services:
  generator:
    build: ./generator
    environment: 
      KAFKA_BROKER_URL: broker:9092
      TRANSACTIONS_TOPIC: queueing.transactions
      TRANSACTIONS_PER_SECOND: 1000

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
```yml
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


### Create Kafka network
To allow both docker-compose compositions to access the same network, we must create an external Docker network. We can accomplish this by running:  
```Shell
docker network create kafka-network
```



### Generator App
Our generator app utilizes the kafka-python client, and employs the KafkaProducer module and our python script ```transactions.py``` in order to emulate an endless stream of transactions produced to the ```TRANSACTIONS_TOPIC```  

Our generator ```app.py``` reads as follows:
```python
# generator/app.py

import os
import json
from time import sleep
from kafka import KafkaProducer
from transactions import create_random_transaction


KAFKA_BROKER_URL = os.environ.get('KAFKA_BROKER_URL')
TRANSACTIONS_TOPIC = os.environ.get('TRANSACTIONS_TOPIC')
TRANSACTIONS_PER_SECOND = float(os.environ.get('TRANSACTIONS_PER_SECOND'))
SLEEP_TIME = 1 / TRANSACTIONS_PER_SECOND


if __name__ == '__main__':

    producer = KafkaProducer(
        bootstrap_servers=KAFKA_BROKER_URL, 
        # encode all values as JSON
        value_serializer=lambda value: json.dumps(value).encode('utf-8')
        )

    while True:
        transaction: dict = create_random_transaction()
        producer.send(TRANSACTIONS_TOPIC, value=transaction)
        print(transaction) # DEBUG
        sleep(SLEEP_TIME)
```


### Transactions script 
Our ```transactions.py``` script is called within our generator ```app.py``` and is used to generate a series of random transactions and reads as follows:  

```python
# generator/transactions.py

from random import choices, randint
from string import ascii_letters, digits

account_chars: str = digits + ascii_letters

def _random_account_id() -> str:
    """Return a random account number made of 12 characters.""" 
    return "".join(choices(account_chars, k=12))

def _random_amount() -> float:
    """Return a random amount between 1.00 and 1000.00.""" 
    return randint(100, 1000000) / 100

def create_random_transaction() -> dict: 
    """Create a fake, randomised transaction.""" 
    return {
        'source': _random_account_id(), 
        'target': _random_account_id(), 
        'amount': _random_amount(),
        # Keep it simple: it's all dollars 
        'currency': 'USD'
    }    
```


### Detector App
Our detector app utilizes the kafka-python KafkaConsumer module to consume transactions from the ```TRANSACTIONS_TOPIC```. Our detector app's custom logic stipulates that if any transaction is greater than or equal to $900 USD, it is to be considered fraud. Transactions read in by our consumer are then written out to either ```FRAUD_TOPIC``` or ```LEGIT_TOPIC``` depending on the transaction amount.  

Our detector ```app.py``` reads as follows:
```python
# detector/app.py

import os
import json
from kafka import KafkaConsumer, KafkaProducer


KAFKA_BROKER_URL = os.environ.get('KAFKA_BROKER_URL')
TRANSACTIONS_TOPIC = os.environ.get('TRANSACTIONS_TOPIC')
LEGIT_TOPIC = os.environ.get('LEGIT_TOPIC')
FRAUD_TOPIC = os.environ.get('FRAUD_TOPIC')


def is_suspicious(transaction: dict) -> bool:
    """Determine whether a transaction is suspicious."""
    return transaction['amount'] >= 900

if __name__ == '__main__':

    consumer = KafkaConsumer(
        TRANSACTIONS_TOPIC,
        bootstrap_servers=KAFKA_BROKER_URL,
        value_deserializer=lambda value: json.loads(value)
    )

    producer = KafkaProducer(
        bootstrap_servers=KAFKA_BROKER_URL,
        value_serializer=lambda value: json.dumps(value).encode('utf-8')
    )

    for message in consumer:
        transaction: dict = message.value
        topic = FRAUD_TOPIC if is_suspicious(transaction) else LEGIT_TOPIC
        producer.send(topic, value=transaction)
        print(topic, transaction) # DEBUG
```


## Execution

In order to run our fraud detection app we can navigate to the project folder and run the following commands:

### Step 1
First, spin up the Kafka cluster:  
```docker-compose -f docker-compose.kafka.yml up```  

![start_cluster](/screenshots/start_kafka_cluster.png)


### Step 2
Next, in a seperate terminal windown, we will start our generator and detector services:    
```docker-compose -f docker-compose.yml up```

![start_services](/screenshots/docker_compose_up.png)


### Step 3
Next, in a seperate terminal window, using the kafka-console-consumer within our broker container, we can consume our legitimate transactions:  
```docker-compose -f docker-compose.kafka.yml exec broker kafka-console-consumer --bootstrap-server localhost:9092 --topic streaming.transactions.legit```

![legit_transactions](/screenshots/legit.png)


### Step 4
Finally, in a seperate terminal window, also using the kafka-console-consumer within our broker container, we can consume our fraudulent transactions:
```docker-compose -f docker-compose.kafka.yml exec broker kafka-console-consumer --bootstrap-server localhost:9092 --topic streaming.transactions.fraud```

![fraud_transactions](/screenshots/fraud.png)


Notice that all 'legit' amounts are less than $900 while all 'fraud' amounts are greater than $900.  
We now have a working fraud detection pipeline!
