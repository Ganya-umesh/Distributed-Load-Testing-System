# Load Testing

- **Installing The Go  compiler, tool chain, and runtime environment**
    
    ```bash
    #!/bin/bash
    
    # shifting to home directory
    start_dir=$(pwd)
    cd
    
    echo "======================"
    echo " Installing GO "
    echo "======================"
    
    sudo apt-get update -y
    sudo apt-get upgrade -y
    
    echo "======================"
    echo " Downloading GO installation file "
    echo "======================"
    
    wget https://go.dev/dl/go1.21.3.linux-amd64.tar.gz
    
    tar -xvf go1.21.3.linux-amd64.tar.gz
    
    if [ -d "/usr/local/go" ]; then
    	echo "======================"
    	echo "Deleting previous installation"
    	echo "======================"
    	sudo rm -rf /usr/local/go
    fi
    
    sudo mv go /usr/local
    
    echo "export GOROOT=/usr/local/go" >> ~/.bashrc
    echo "export GOPATH=\$HOME/go" >> ~/.bashrc
    echo "export PATH=\$PATH:\$GOPATH/bin:\$GOROOT/bin" >> ~/.bashrc
    source ~/.bashrc
    
    echo "======================"
    echo " Installation Complete "
    go version
    echo "======================"
    ```
    

## Producer and Consumer Code

- consumer.go
    
    ```go
    package main
    
    import (
    	"log"
    	"github.com/IBM/sarama"
    	"os"
    	"os/signal"
    	"encoding/json"
    )
    
    type message struct {
            Key string `json:"key"`
            Value string `json:"value"`
    }
    
    func main() {
    	brokers := []string{"localhost:9092"}
    	config := sarama.NewConfig()
    	topic := "test"
    
    	consumer, err := sarama.NewConsumer(brokers, config)
    	if err != nil {
    		log.Fatalln(err)
    	}
    
    	defer func() {
    		if err := consumer.Close(); err != nil {
    			log.Fatalln(err)
    		}
    	}()
    
    	partitionConsumer, err := consumer.ConsumePartition(topic, 0, sarama.OffsetNewest)
    	if err != nil {
    		log.Panic(err)
    	}
    
    	// trap SIGINT to trigger a shutdown
    	signals := make(chan os.Signal, 1)
    	signal.Notify(signals, os.Interrupt)
    
    	consumed := 0
    	ConsumerLoop:
    	for {
    		select {
    			case msg := <-partitionConsumer.Messages():
    				var decodedMsg message
    				err := json.Unmarshal(msg.Value, &decodedMsg)
    				if err != nil {
    					log.Panic(err)
    				}
    
    				log.Println("Consumed message:", decodedMsg)
    				consumed++
    			case <-signals:
    				break ConsumerLoop
    		}
    	}
    
    	log.Printf("Consumed: %d\n", consumed)
    }
    ```
    
- producer.go
    
    ```go
    package main
    
    import (
    	"log"
    	"github.com/IBM/sarama"
    	"os"
    	"os/signal"
    	"encoding/json"
    	// "time"
    )
    
    type message struct {
    	Key string `json:"key"`
    	Value string `json:"value"`
    }
    
    func main() {
    	brokers := []string{"localhost:9092"}
    	topic := "test"
    	config := sarama.NewConfig()
    	config.Producer.Return.Successes = true
    
    	producer, err := sarama.NewAsyncProducer(brokers, config)
    
    	if( err != nil) {
    		log.Panic(err)
    	}
    
    	defer func() {
    		if err := producer.Close(); err != nil {
    			log.Panic(err)
    		}
    	} ()
    
    	signals := make(chan os.Signal, 1)
    	signal.Notify(signals, os.Interrupt)
    
    	var enqueued, errors int
    
    	messages := []message{{Key: "message", Value: "test 1"}, {Key: "message", Value: "test 2"}}
    
    	encodedMessages := make([][]byte, len(messages))
    	for i, v := range messages {
    		encodedMessage, err := json.Marshal(v)
    		if err != nil {
    			log.Panic(err)
    		}
    		encodedMessages[i] = encodedMessage
    	}
    
    	ProducerLoop:
    	for i, v := range encodedMessages {
    		select {
    			case producer.Input() <- &sarama.ProducerMessage{Topic: topic, Key: nil, Value: sarama.StringEncoder(v)}:
    				enqueued++
    				log.Println("Message Produced:", messages[i])
    
    			case err := <-producer.Errors():
    				log.Println("Failed to produce message", err)
    				errors++
    			case <-signals:
    				break ProducerLoop
    		}
    	}
    
    	log.Printf("Enqueued: %d; errors: %d\n", enqueued, errors)
    }
    ```
    

## Creating Kafka Topics

### Start Kafka

```bash
# To start Kafka
sudo systemctl start kafka
# To check the status of Kafka
sudo systemctl status kafka
# To stop kafka
sudo systemctl stop kafka
```

- Creating Topics
    
    ```bash
    #!/usr/bin/bash
    
    # List Topics
    echo "=================================="
    echo "          Available Topics	"
    echo "=================================="
    /usr/local/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
    
    # Create Topics
    echo "=================================="
    echo "        Creating Given Topic      "
    echo "=================================="
    
    /usr/local/kafka/bin/kafka-topics.sh --create --topic $1 --bootstrap-server localhost:9092
    
    # Get topic description
    echo "=================================="
    echo "        Description of Topic      "  
    echo "=================================="
    /usr/local/kafka/bin/kafka-topics.sh --describe --topic $1 --bootstrap-server localhost:9092
    ```
    

## Setting up the Go dependency management file

This must be run in the folder which consists of your go files

```go
go mod init example/folder
```

## Installing  Sarama by IBM

Sarama is a Go library for working with Apache Kafka, which is a distributed streaming platform. Apache Kafka is commonly used for building real-time data pipelines and streaming applications.

Its a a Go implementation of the Kafka protocol, allowing Go developers to produce and consume messages to and from Kafka topics. It supports various features of Kafka, including different message serialisation formats, partitioning strategies, and authentication mechanisms.

```bash
go get -u github.com/IBM/sarama
```

## Running the code

Build the executable and run using

```bash
#Building the executable
go build file.go
#Running the executable
./file
```