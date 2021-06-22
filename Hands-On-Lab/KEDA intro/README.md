# KEDA Intro Hands-On-Lab

**Content**

- [KEDA Intro Hands-On-Lab](#keda-intro-hands-on-lab)
- [KEDA Intro](#keda-intro)
  - [Abstract and learning objectives](#abstract-and-learning-objectives)
  - [Overview](#overview)
  - [Exercise 1: Create and run the application in Docker](#exercise-1-create-and-run-the-application-in-docker)
    - [Task 1: Deploy a local RabbitMq container](#task-1-deploy-a-local-rabbitmq-container)
    - [Task 2: Deploy the pub sub application](#task-2-deploy-the-pub-sub-application)
  - [Exercise 2: Install KEDA and RabbitMq in a Kubernetes Cluster:](#exercise-2-install-keda-and-rabbitmq-in-a-kubernetes-cluster)
    - [Task 1: Deploy KEDA into our local cluster](#task-1-deploy-keda-into-our-local-cluster)
  - [Exercise 3: Deploy application using KEDA to scale the app:](#exercise-3-deploy-application-using-keda-to-scale-the-app)

# KEDA Intro

## Abstract and learning objectives

This hands-on lab is designed to guide you through the process of building and deploying a scalable application that is able to grow dynamically by taking account of the amount of traffic present in a external source.

At the end of this lab you will be better at building and deploying highly scalable applications in Kubernetes.

This Lab assumes you have Docker Desktop installed with Kubernetes orchestration turned on and Kubernetes Lens IDE.

## Overview

Your company charges you with building an application that will deal with a certain batch workload. This batch workload only happens once a day, your application should quickly complete all the tasks and use the minimum amount of resources when it's not processing any requests.

In this hands on lab we will build an event based application that will simulate your batch workload using RabbitMQ and KEDA to only scale when there are messages or tasks to complete.

## Exercise 1: Create and run the application in Docker

In this task we will build and test the application as a simple pub sub app

But before we start open a terminal window and navigate to the same directory that this readme file is located

```powershell
cd ".\Hands-on-Lab\KEDA intro"
```

### Task 1: Deploy a local RabbitMq container

Before we can build the application and test our code we need to deploy the pubsub system for this we will use [RabbitMq](https://www.rabbitmq.com/) a very popular message broker that is simple to use and to deploy both on cloud and on-premise.

To start the RabbitMq container we need to run the following command

```powershell
docker run -d --hostname my-rabbit -p 15672:15672 -p 5672:5672 rabbitmq:3-management
```
This will start the docker contaner and we can validate this by running 

```powershell
docker ps
```

where we will see the running container 
![running container](./media/content1.png)

We can also validate the Container by visiting the [RabbitMq management dashboard](http://localhost:15672/) and use the username guest and password guest

![Login RabbitMQ management Dashboard](./media/content2.png)

Here we can see the RabbitMq dashboard that provides metrics for the RabbitMq message broker

![RabbitMq Dashboard](./media/content3.png)

So now we have a RabbitMq message brocker deployed we can start to develop our application.

### Task 2: Deploy the pub sub application

To deploy the pubsub application we have to build the Docker Images and them run the Containers.

To do this we travel to the materials folder and we can open it in a text editor, we recommend VSCode.

This can be done by running the following commands:

```powershell
code .\materials
```

Now we can explore the code, this application will use a single docker image to facilitate the demonstration but it will use 2 important files:

The first is the receiver.go file that has the subscriber applications, this is the application that we will scale up using KEDA 

```go
package main

import (
	"log"
	"os"
	"time"

	"github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}

func main() {
	url := os.Args[1]
	conn, err := amqp.Dial(url)
	failOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	ch, err := conn.Channel()
	failOnError(err, "Failed to open a channel")
	defer ch.Close()

	failOnError(err, "Failed to set QoS")

	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	failOnError(err, "Failed to declare a queue")

	err = ch.Qos(
		1,     // prefetch count
		0,     // prefetch size
		false, // global
	)

	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		false,  // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	failOnError(err, "Failed to register a consumer")

	forever := make(chan bool)

	go func() {
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
			time.Sleep(1 * time.Second)
			d.Ack(false)
		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever
}
```

The second important file is the send.go file that has a simple implementation of a publisher that will send messages to the queue.

```go
package main

import (
	"fmt"
	"log"
	"os"
	"strconv"

	"github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}

func main() {
	url := os.Args[1]
	messageCount, err := strconv.Atoi(os.Args[2])
	failOnError(err, "Failed to parse second arg as messageCount : int")
	conn, err := amqp.Dial(url)
	failOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	ch, err := conn.Channel()
	failOnError(err, "Failed to open a channel")
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	failOnError(err, "Failed to declare a queue")

	for i := 0; i < messageCount; i++ {
		body := fmt.Sprintf("Hello World: %d", i)
		err = ch.Publish(
			"",     // exchange
			q.Name, // routing key
			false,  // mandatory
			false,  // immediate
			amqp.Publishing{
				ContentType: "text/plain",
				Body:        []byte(body),
			})
		log.Printf(" [x] Sent %s", body)
		failOnError(err, "Failed to publish a message")
	}
}
```

Now that we have our application we need to build it's Docker container, to do this navigate to the directory that has the dockerfile

```docker
FROM golang:1.12.1 as builder

# Copy the code from the host and compile it
WORKDIR $GOPATH/src/pub-sub-app

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go install ./...

FROM scratch
COPY --from=builder /go/bin/receive /go/bin/send /usr/local/bin/
CMD ["receive"]
```

And run the following command:


```powershell
docker build -t pub-sub-app .
```

Before proceding use ipconfig or ifconfig to find your machines local IP address (rabbitmq doesn't like localhost connections).

```powershell
ipconfig
```

Now that we have our container lets execute the container to see the application running.

THe best way to do this is to open 2 terminals side by side as such:

![Terminals in VScode side by side](./media/content4.png)

Then on one side run the subscriber command:

```powershell
docker run pub-sub-app receive "amqp://guest:guest@MYMACHINESIP:5672"
```

and on the other side run the publisher command and send 10 messages

```powershell
docker run pub-sub-app send "amqp://guest:guest@MYMACHINESIP:5672" 10
```

and we can now see our application running and catching the messages of the other side

![Execution Result](./media/content5.png)

## Exercise 2: Install KEDA and RabbitMq in a Kubernetes Cluster:

Now that our application is ready we can start to prepare our Cluster by adding KEDA and RabbitMq to our local cluster.

### Task 1: Deploy KEDA into our local cluster

First login into your kubernetes cluster via lens so we can visualize our changes as they are happening.

Now that we have the cluster open we have to install KEDA to do this run in Lens integrated shell:

```powershell
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.3.0/keda-2.3.0.yaml
```

And now that KEDA is installed we can install RabbitMq using Helm by running:

```powershell
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install rabbitmq --set auth.username=user --set auth.password=PASSWORD bitnami/rabbitmq
```

If you don't have helm installed run the following:

1. Install [Chocolatey](https://chocolatey.org/install)

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; 

[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

2. Use Chocolatey to install Helm 3

    ```code
    choco install kubernetes-helm
    ```

And repeat the command:

```powershell
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install rabbitmq --set auth.username=user --set auth.password=PASSWORD bitnami/rabbitmq
```

Now we have RabbitMq and KEDA running we can move on to the final exercise.

## Exercise 3: Deploy application using KEDA to scale the app:

Now that we have the environment prepared we have to deploy our application.

To do this we can apply the manifests in the materials/deploy directory.
But before we can do that we lets analysis how KEDA will control our application's scale.

To do this we can open the deploy-consumer.yaml file, since that is where the definitions to run KEDA and our subscriber application are defined.

Here we can see multiple Kubernetes objects:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-consumer-secret
data:
  RabbitMqHost: YW1xcDovL3VzZXI6UEFTU1dPUkRAcmFiYml0bXEuZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbDo1Njcy
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer
  namespace: default
spec:
  scaleTargetRef:
    name: rabbitmq-consumer
  pollingInterval: 5 # Optional. Default: 30 seconds
  cooldownPeriod: 30 # Optional. Default: 300 seconds
  maxReplicaCount: 30 # Optional. Default: 100
  triggers:
    - type: rabbitmq
      metadata:
        queueName: hello
        queueLength: "5"
      authenticationRef:
        name: rabbitmq-consumer-trigger
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-consumer-trigger
  namespace: default
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-consumer-secret
      key: RabbitMqHost
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq-consumer
  namespace: default
  labels:
    app: rabbitmq-consumer
spec:
  selector:
    matchLabels:
      app: rabbitmq-consumer
  template:
    metadata:
      labels:
        app: rabbitmq-consumer
    spec:
      containers:
        - name: rabbitmq-consumer
          image: pub-sub-app
          imagePullPolicy: IfNotPresent
          command:
            - receive
          args:
            - "amqp://user:PASSWORD@rabbitmq.default.svc.cluster.local:5672"
---
```

THe first object is a secret that contains the connections string to the queue we want to monitor, this is the authentication that KEDA will use to access the queue.

The second object is the most important one since it's this object that defines the behavior of KEDA and therefore the behavior of our application.

```yaml
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer
  namespace: default
spec:
  scaleTargetRef:
    name: rabbitmq-consumer
  pollingInterval: 5 # Optional. Default: 30 seconds
  cooldownPeriod: 30 # Optional. Default: 300 seconds
  maxReplicaCount: 30 # Optional. Default: 100
  triggers:
    - type: rabbitmq
      metadata:
        queueName: hello
        queueLength: "5"
      authenticationRef:
        name: rabbitmq-consumer-trigger
```

In this object we see lots of data that we can edit:

- The first one is the scale target, this is the name of the deployment we want to control via KEDA
- The second one is the polling interval, this defines how often KEDA will check our triggers to validate the conditions
- The third is the max number of replicas our deployment can reach
- The final one is the trigger definition where we can see the type of trigger "rabbitmq" the queue we want to monitor and the queue length that we will use as the scaling point, there is also the authentication object where we say from where to fetch the authentication.

The other KEDA related object is a trigger authentication and it's where we bind the secret created earlier with the KEDA scaled object

Finally you can also see the deployment doesn't specify replicas since KEDA will take care of this we don't have to specify the replicat number

Now that we understand these objects lets deploy them so we can see KEDA in action:

```powershell
kubectl apply -f ./materials/deploy/deploy-consumer.yaml
```

and after applying this we see nothing changes

![No changes in LENS](./media/content6.png)

This is because we don't have any messages on the queue, to fix this let's apply the deploy-publisher-job.yaml manifest that has the publisher creating messages

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: rabbitmq-publish
spec:
  template:
    spec:
      containers:
      - name: rabbitmq-client
        image: pub-sub-app
        imagePullPolicy: IfNotPresent
        command: ["send",  "amqp://user:PASSWORD@rabbitmq.default.svc.cluster.local:5672", "300"]
      restartPolicy: Never
  backoffLimit: 4
```

This manifest defines a job that will create 300 messages to the queue and this will make KEDA scale up our deployment.

So we apply this Manifest:

```powershell
kubectl apply -f ./materials/deploy/deploy-publisher-job.yaml
```

And with just that we can see our number of pods increasing

![Publisher running](./media/content7.png)

And with this we see how KEDA can help us create take our event driven applications to the next level.