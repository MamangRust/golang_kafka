## Kafka Producer and Worker with golang

We will create a simple application using Apache Kafka with Golang. This application consists of two main components: Producer and Worker. The Producer is responsible for sending messages to Kafka, while the Worker will consume those messages.

## Preparation

Make sure you have a running Kafka broker. For development ease, you can use Docker Compose with the following configuration:

```yaml
version: '3.8'

services:
  kafka:
    image: docker.io/bitnami/kafka:3.6
    ports:
      - '9092:9092'
    volumes:
      - 'kafka_data:/bitnami'
    environment:
      # KRaft settings
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      # Listeners
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT
    networks:
      - my-kafka

  producer:
    build:
      context: ./producer
    restart: always
    container_name: producer
    depends_on:
      - kafka
    ports:
      - '3000:3000'
    networks:
      - my-kafka

  worker:
    build:
      context: ./worker
    restart: always
    container_name: worker
    depends_on:
      - kafka
    networks:
      - my-kafka

volumes:
  kafka_data:
    driver: local

networks:
  my-kafka:
    driver: bridge
```

### Create Kafka Producer

```go
// producer/main.go

package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"

	"github.com/Shopify/sarama"
)

func main() {
	brokers := []string{"localhost:9092"}
	topic := "comments"

	producer, err := connectProducer(brokers)
	if err != nil {
		panic(err)
	}
	defer producer.Close()

	// Trap signals to gracefully shutdown producer
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM)

	// Produce messages to Kafka
	go produceMessages(producer, topic)

	// Wait for termination signal
	<-signals
	fmt.Println("Shutting down producer...")
}

func connectProducer(brokers []string) (sarama.SyncProducer, error) {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	config.Producer.RequiredAcks = sarama.WaitForAll
	config.Producer.Retry.Max = 5

	producer, err := sarama.NewSyncProducer(brokers, config)
	if err != nil {
		return nil, err
	}

	return producer, nil
}

func produceMessages(producer sarama.SyncProducer, topic string) {
	for i := 1; i <= 10; i++ {
		message := fmt.Sprintf("Message %d", i)
		msg := &sarama.ProducerMessage{
			Topic: topic,
			Value: sarama.StringEncoder(message),
		}

		partition, offset, err := producer.SendMessage(msg)
		if err != nil {
			fmt.Printf("Failed to send message: %s\n", err)
		} else {
			fmt.Printf("Message sent to topic(%s)/partition(%d)/offset(%d): %s\n", topic, partition, offset, message)
		}
	}
}
```

#### Docker producer

```Dockerfile
FROM golang:1.21-alpine

COPY . /app

WORKDIR /app

RUN go mod download

RUN go build -o main .

CMD ["./main"]
```

## Create Kafka Worker

```go
// worker/main.go

package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"

	"github.com/Shopify/sarama"
)

func main() {
	brokers := []string{"localhost:9092"}
	topic := "comments"

	consumer, err := connectConsumer(brokers)
	if err != nil {
		panic(err)
	}
	defer consumer.Close()

	// Trap signals to gracefully shutdown consumer
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM)

	// Consume messages from Kafka
	go consumeMessages(consumer, topic)

	// Wait for termination signal
	<-signals
	fmt.Println("Shutting down consumer...")
}

func connectConsumer(brokers []string) (sarama.Consumer, error) {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true

	consumer, err := sarama.NewConsumer(brokers, config)
	if err != nil {
		return nil, err
	}

	return consumer, nil
}

func consumeMessages(consumer sarama.Consumer, topic string) {
	partitionConsumer, err := consumer.ConsumePartition(topic, 0, sarama.OffsetOldest)
	if err != nil {
		fmt.Printf("Failed to start consumer: %s\n", err)
		return
	}

	fmt.Println("Consumer started")

	// Process messages
	for {
		select {
		case err := <-partitionConsumer.Errors():
			fmt.Printf("Error: %s\n", err)
		case msg := <-partitionConsumer.Messages():
			fmt.Printf("Received message | Topic(%s) | Partition(%d) | Offset(%d) | Value(%s)\n",
				string(msg.Topic), msg.Partition, msg.Offset, string(msg.Value))
		}
	}
}

```

## Running on Docker

```sh
docker-compose up --build -d


docker-compose logs producer

docker-compose logs worker
```
