# Deploying Kafka with Kubernetes on Minikube

This mini-project demonstrates how to set up a Kafka cluster on Kubernetes using Minikube. The setup is cloud-neutral and can be adapted for other cloud providers with minor adjustments.

## Prerequisites

- Minikube v1.25.2
- kubectl client v1.23.3
- KCat (formerly Kafkacat) for testing

## Setup

### 1. Start Minikube

```bash
minikube start
```

Verify the status:

```bash
minikube status
```

Expected output:
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

## Deployment Steps

### 1. Create Kafka Namespace

Create `00-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "kafka"
  labels:
    name: "kafka"
```

Apply the configuration:

```bash
kubectl apply -f 00-namespace.yaml
```

Verify:

```bash
kubectl get namespaces
```

### 2. Deploy Zookeeper

Create `01-zookeeper.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: zookeeper-service
  name: zookeeper-service
  namespace: kafka
spec:
  type: NodePort
  ports:
    - name: zookeeper-port
      port: 2181
      nodePort: 30181
      targetPort: 2181
  selector:
    app: zookeeper
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: zookeeper
  name: zookeeper
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - image: wurstmeister/zookeeper
          imagePullPolicy: IfNotPresent
          name: zookeeper
          ports:
            - containerPort: 2181
```

Apply the configuration:

```bash
kubectl apply -f 01-zookeeper.yaml
```

Verify the service:

```bash
kubectl get services -n kafka
```

Note the Zookeeper CLUSTER-IP for the next step.

### 3. Deploy Kafka Broker

Create `02-kafka.yaml` (replace `<ZOOKEEPER-INTERNAL-IP>` with the IP from previous step):

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kafka-broker
  name: kafka-service
  namespace: kafka
spec:
  ports:
  - port: 9092
  selector:
    app: kafka-broker
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kafka-broker
  name: kafka-broker
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-broker
  template:
    metadata:
      labels:
        app: kafka-broker
    spec:
      hostname: kafka-broker
      containers:
      - env:
        - name: KAFKA_BROKER_ID
          value: "1"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: <ZOOKEEPER-INTERNAL-IP>:2181
        - name: KAFKA_LISTENERS
          value: PLAINTEXT://:9092
        - name: KAFKA_ADVERTISED_LISTENERS
          value: PLAINTEXT://kafka-broker:9092
        image: wurstmeister/kafka
        imagePullPolicy: IfNotPresent
        name: kafka-broker
        ports:
        - containerPort: 9092
```

Apply the configuration:

```bash
kubectl apply -f 02-kafka.yaml
```

Verify the pods:

```bash
kubectl get pods -n kafka
```

### 4. Update Local Hosts File

Add this line to your `/etc/hosts` file:

```
127.0.0.1 kafka-broker
```

## Testing the Setup

### 1. Expose Kafka Port

```bash
kubectl port-forward <kafka-pod-name> 9092 -n kafka
```

### 2. Produce a Test Message

```bash
echo "hello world!" | kafkacat -P -b localhost:9092 -t test
```

### 3. Consume the Message

```bash
kafkacat -C -b localhost:9092 -t test
```

Expected output:
```
hello world!
% Reached end of topic test [0] at offset 1
```
