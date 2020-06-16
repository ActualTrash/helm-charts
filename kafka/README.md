# Install Kafka For "Full" Setup

This directory contains the files needed to install kafka with 3 brokers with a sidecar for each broker.  This will follow the pattern described in the full setup.

## Steps

1. Create kafka namespace and add secrets

    ```bash
    kubectl create namespace kafka-gm
    kubectl get secret docker.secret --export -o yaml | kubectl apply --namespace=kafka-gm -f -
    kubectl get secret sidecar-certs --export -o yaml | kubectl apply --namespace=kafka-gm -f -
    ```

2. Install kafka/sidecars

    ```bash
    kubectl apply -f kafka/configmap-b0.yaml
    kubectl apply -f kafka/configmap-b1.yaml
    kubectl apply -f kafka/configmap-b2.yaml
    kubectl apply -f kafka/svc-b0.yaml
    kubectl apply -f kafka/svc-b1.yaml
    kubectl apply -f kafka/svc-b2.yaml
    kubectl apply -f kafka/kafka_template.yaml -n kafka-gm
    ```

3. Install coughka deployment for testing

Run a kafka client and create any topics - by default in coughka we're using coughka-test-topic, so:

```bash
kubectl run kafka-observables-client --rm --tty -i --restart='Never' --image docker.io/bitnami/kafka:2.4.0-debian-9-r22 --namespace kafka-gm --command -- bash
```

and then:

```bash
kafka-topics.sh --create --bootstrap-server kafka-broker-1.kafka-gm.svc.cluster.local:9093 --topic coughka-test-topic
```

Verify with

```bash
kafka-topics.sh --list --zookeeper kafka-observables-zookeeper-headless.kafka-gm.svc.cluster.local:2181
```

Now configure the mesh for the incoming coughka/sidecar combo:

```bash
for cl in kafka/coughka/mesh/clusters/*.json; do greymatter create cluster < $cl; done
for cl in kafka/coughka/mesh/domains/*.json; do greymatter create domain < $cl; done
for cl in kafka/coughka/mesh/listeners/*.json; do greymatter create listener < $cl; done
for cl in kafka/coughka/mesh/proxies/*.json; do greymatter create proxy < $cl; done
for cl in kafka/coughka/mesh/rules/*.json; do greymatter create shared_rules < $cl; done
for cl in kafka/coughka/mesh/routes/*.json; do greymatter create route < $cl; done
```

```bash
kubectl apply -f kafka/coughka/coughka-deployment.yaml
```

Once everything is running, there should be no errors in the coughka container logs.  You can use a consumer to check the messages or go to `services/coughka/published` to see the list of messages being published by the coughka service and `/services/coughka/subscribed` to see the list of messages being consumed by the coughka service.
